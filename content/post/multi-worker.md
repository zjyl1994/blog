---
title: "Golang 泛型初体验"
date: 2022-08-05T21:53:00+08:00
lastmod: 2022-08-05T21:53:00+08:00
draft: false
---

> 最近给公司的几个服务升级到了Go1.18,总算能好好的体验泛型了。

1.18泛型更新了一段时间，我们公司的基础构架也上了1.18的底包，可以愉快的玩耍泛型了。

Golang没有函数重载，之前也没有泛型，大部分业务写起来也没什么问题。
但是当你想把某些重复逻辑封装成函数的时候会面临一个问题，强类型。

没有泛型之前，需要针对每个类型都写一份实现。很多时候你只要把值传来传去，并不关心内部的值是什么。比如我最常用的多线程异步执行并获取结果。

没有泛型之前需要这么写：

```Go

func MultiWorker(ctx context.Context, workerNum int, inputs []interface{},
	fn func(context.Context, interface{}) (interface{}, error)) ([]interface{}, error) {
	var resultLock sync.Mutex
	results := make([]interface{}, 0, len(inputs))
	jobCh := make(chan interface{}, workerNum)
	waitg, ctx := errgroup.WithContext(ctx)

	for i := 0; i < workerNum; i++ {
		waitg.Go(func() error {
			for input := range jobCh {
				result, err := fn(ctx, input)
				if err != nil {
					return err
				} else {
					resultLock.Lock()
					results = append(results, result)
					resultLock.Unlock()
				}
			}
			return nil
		})
	}

	for _, input := range inputs {
		jobCh <- input
	}
	close(jobCh)

	err := waitg.Wait()
	return results, err
}
```

表面看起来没什么问题，但是实际使用中体验很差，输入的`inputs`无论什么类型都需要先用for循环转换成`[]interface{}`才能使用，输出的值也需要进行类型断言才能得到需要的值，一旦中途混了其他类型的数据进去，断言失败会得到一个 `panic`，这很不好。

现在有了泛型以后，Golang 编译器会在你使用到泛型部分代码的时候进行类型展开，生成对应类型的代码编译进去，对于程序员来说是无感知的。这样下来只会在最终的二进制中膨胀出一点点汇编代码，效率性能也不会有多少损失。

有了泛型以后就可以这么写了：

```go
import (
	"context"
	"sync"

	"golang.org/x/sync/errgroup"
)

func MultiWorker[K, V any](ctx context.Context, workerNum int, inputs []K,
	fn func(context.Context, K) (V, error)) ([]V, error) {
	var resultLock sync.Mutex
	results := make([]V, 0, len(inputs))
	jobCh := make(chan K, workerNum)
	waitg, ctx := errgroup.WithContext(ctx)

	for i := 0; i < workerNum; i++ {
		waitg.Go(func() error {
			for input := range jobCh {
				result, err := fn(ctx, input)
				if err != nil {
					return err
				} else {
					resultLock.Lock()
					results = append(results, result)
					resultLock.Unlock()
				}
			}
			return nil
		})
	}

	for _, input := range inputs {
		jobCh <- input
	}
	close(jobCh)

	err := waitg.Wait()
	return results, err
}
```

优势呢，全程带着类型，不需要任何额外的类型断言，如果写错编译期检测都不会过，强壮性得到了极大增强。

同时，泛型也催动了一批新的基础包广泛使用。
`github.com/samber/lo`提供了对标`Lodash`的大量函数，都由1.18泛型写成，现在Github已经超过7K star。

我在公司项目的重构中就大量使用了`lo`这个软件包。`Filter`过滤数组元素、`Map`批量处理数据格式转换，都非常方便。
如果打开源码查看，会发现他们的实现都很简单。而且根据Golang的一贯优化，这种超简单的确定性代码最终都会被内联，所以完全不用担心性能问题。

不得不说，这次1.18更新的泛型是一个非常赞的东西，开发愉快度大大提升。

附：使用了泛型的多线程异步执行程序（完整版本）
```go
import (
	"context"
	"sync"

	"golang.org/x/sync/errgroup"
)

func MultiWorkerDo[K any](ctx context.Context, workerNum int, inputs []K,
	fn func(context.Context, K) error) error {
	jobCh := make(chan K, workerNum)
	waitg, ctx := errgroup.WithContext(ctx)

	for i := 0; i < workerNum; i++ {
		waitg.Go(func() error {
			for input := range jobCh {
				err := fn(ctx, input)
				if err != nil {
					return err
				}
			}
			return nil
		})
	}

	for _, input := range inputs {
		jobCh <- input
	}
	close(jobCh)

	return waitg.Wait()
}

func MultiWorkerResult[K, V any](ctx context.Context, workerNum int, inputs []K,
	fn func(context.Context, K) (V, error)) ([]V, error) {
	var resultLock sync.Mutex
	results := make([]V, 0, len(inputs))
	jobCh := make(chan K, workerNum)
	waitg, ctx := errgroup.WithContext(ctx)

	for i := 0; i < workerNum; i++ {
		waitg.Go(func() error {
			for input := range jobCh {
				result, err := fn(ctx, input)
				if err != nil {
					return err
				} else {
					resultLock.Lock()
					results = append(results, result)
					resultLock.Unlock()
				}
			}
			return nil
		})
	}

	for _, input := range inputs {
		jobCh <- input
	}
	close(jobCh)

	err := waitg.Wait()
	return results, err
}
```