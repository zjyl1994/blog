---
title: "Golang 优化之合并请求"
date: 2025-03-23T20:00:00+08:00
lastmod: 2025-03-23T20:00:00+08:00
draft: false
tags: ["Golang"]
categories: ["Golang"]

---

很多时候，一些请求是可以合并的，比如库存扣减常用的 `UPDATE qty = qty -1`
如果我们在并发场景下，每个请求都去执行一次 `UPDATE qty = qty -1`，那么就会有大量的请求竞争同一行数据，造成性能瓶颈。
所以我们可以通过合并请求的方式，将多个请求合并成一个请求，一个动作完成多次扣减，然后再执行。

下面就是我封装好的工具包，用于合并请求。

<!--more-->

WriteGroup 会根据 key 进行合并动作，把一组相同 key 的请求合并成一个数组交给回调函数进行处理。
回调函数要求输入输出数量保持一致，否则可能产生异常问题。

```go
package writegroup

import (
	"context"
	"sync"
	"time"
)

type WriteGroup[T, R any] interface {
	Do(ctx context.Context, key string, input T, fn func(context.Context, []T) []WriteResult[R]) (R, error)
}

// 组写入方法
type writeGroup[T, R any] struct {
	batchTimeout time.Duration
	bufferSize   int
	batchSize    int
	workerMu     keyedMutex
	chMu         sync.Mutex
	workerCh     map[string]chan writeTask[T, R]
}

// 具体的写入任务
type writeTask[T, R any] struct {
	input  T
	result chan<- WriteResult[R]
}

// 承载返回值
type WriteResult[R any] struct {
	Result R
	Err    error
}

func NewWriteGroup[T, R any](bufferSize, batchSize int, batchTimeout time.Duration) WriteGroup[T, R] {
	return &writeGroup[T, R]{
		bufferSize:   bufferSize,
		batchSize:    batchSize,
		batchTimeout: batchTimeout,
		workerCh:     make(map[string]chan writeTask[T, R]),
	}
}

func (w *writeGroup[T, R]) Do(ctx context.Context, key string, input T, fn func(context.Context, []T) []WriteResult[R]) (R, error) {
	// 初始化任务队列并投递
	w.chMu.Lock() // 如果队列填满，后面无法解锁，此处会阻塞等待
	ch, ok := w.workerCh[key]
	if !ok {
		ch = make(chan writeTask[T, R], w.bufferSize)
		w.workerCh[key] = ch
	}
	w.chMu.Unlock()
	resultCh := make(chan WriteResult[R], 1)
	ch <- writeTask[T, R]{
		input:  input,
		result: resultCh,
	}
	// 启动处理线程消耗ch中的数据
	workerMu := w.workerMu.GetMutex(key)
	if workerMu.TryLock() { // 抢到锁的进行处理
		go func(lock *sync.Mutex) {
			defer lock.Unlock()
			// 封装的工作函数，负责执行writeTask并发送执行结果
			worker := func(jobs []writeTask[T, R]) {
				jobInputs := make([]T, len(jobs))
				for i, job := range jobs {
					jobInputs[i] = job.input
				}

				results := fn(ctx, jobInputs)
				for i, job := range jobs {
					job.result <- results[i]
				}
			}
			// 启动读取超时定时器
			timeoutTimer := time.NewTimer(w.batchTimeout)
			defer timeoutTimer.Stop()
			// 循环读直到ch无数据或关闭再退出
			for {
				jobs := make([]writeTask[T, R], 0, w.bufferSize)
				timeoutTimer.Reset(w.batchTimeout)
			PROC:
				for { // 读ch直到满或超时
					select {
					case <-timeoutTimer.C: // 超时情况，处理已在队列中的部分
						if len(jobs) > 0 {
							worker(jobs)
						}
						break PROC
					case val, ok := <-ch:
						if !ok { // 队列关闭，处理队列中的部分
							if len(jobs) > 0 {
								worker(jobs)
							}
							break PROC
						}
						// 队列正常，有新消息输入
						jobs = append(jobs, val)
						if len(jobs) >= w.batchSize { // 超过批量上限，处理一波
							worker(jobs)
							break PROC
						}
						// 没超过上限进入下一轮继续读
					}
				}
				if ctr := len(ch); ctr == 0 { // 队列中没有了，停止工作
					return
				}
			}
		}(workerMu)
	}
	// 获得计算好的结果，支持根据context超时
	select {
	case <-ctx.Done():
		return *new(R), ctx.Err()
	case ret := <-resultCh:
		return ret.Result, ret.Err
	}
}

type keyedMutex struct {
	m sync.Map // map[string]*sync.Mutex
}

func (km *keyedMutex) GetMutex(key string) *sync.Mutex {
	if mu, ok := km.m.Load(key); ok {
		return mu.(*sync.Mutex)
	}

	newMu := &sync.Mutex{}
	if value, loaded := km.m.LoadOrStore(key, newMu); !loaded {
		return newMu
	} else {
		return value.(*sync.Mutex)
	}
}
```