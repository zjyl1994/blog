---
title: "Ringbuffer环形缓冲区"
date: 2018-08-05T11:53:00+08:00
lastmod: 2018-08-05T11:53:00+08:00
draft: false
---

# 前言

环形缓冲区是很常用的数据结构，用途很广泛。我目前遇到的使用场景就是TCP分片转回TCP流，一个线程读取TCP分片拆包，写入缓存区，另一个线程从缓存区中读取，互相不影响。

为此Google了一番，发现了kfifo这个Linux内核的实现，十分精妙。

# 原理

## 基本原理

```c
struct kfifo {   
    unsigned char *buffer;    /* the buffer holding the data */   
    unsigned int size;    /* the size of the allocated buffer */   
    unsigned int in;    /* data is added at offset (in % size) */   
    unsigned int out;    /* data is extracted from off. (out % size) */   
    spinlock_t *lock;    /* protects concurrent modifications */   
};
```

先看一下基本定义，buffer是数据缓存区，size是缓存区长度，in/out是输入输出指针位置（%size后就是真实指针了）。

size需要检测是否为2的次幂，不是的话需要升到2的次幂。方便后续计算。（可以看内核的roundup_pow_of_two实现）

in、out每次操作都+对应的读写长度，通过取模运算回落size区间。

data := kfifo.buffer[kfifo.out % kfifo.size:kfifo.in % kfifo.size]

data就是有效数据的缓存区区间。

## 快速取模

kfifo使用了二进制的位运算实现了2的幂取模运算，并且利用了无符号数的溢出做回绕。很精妙的设计。

首先，缓存区尺寸要取整为2的幂次，这样可以利用位运算进行取模。

`M mod N = M & (N-1)`,当N为2的幂次时有效。

取模运算对于计算机来说运算速度也是很慢的（虽然你感觉不出来），而位运算就是分分钟的事了。

## 回绕

无符号数溢出后，就会变成0从头开始，借助这个特性可以绕开逻辑判断。

<!--more-->

# Go版本实现

```go
type RingBuffer struct {
	data []byte
	size uint
	in   uint
	out  uint
}

func NewRingBuffer(size uint) RingBuffer {
	// if size not pow of 2 round up it
	if (size & (size - 1)) != 0 {
		size = size | (size >> 1)
		size = size | (size >> 2)
		size = size | (size >> 4)
		size = size | (size >> 8)
		size = size | (size >> 16)
		size++
	}
	var rb RingBuffer
	rb.size = size
	rb.data = make([]byte, size)
	rb.in = 0
	rb.out = 0
	return rb
}

func (rb *RingBuffer) Write(p []byte) (n int, err error) {
	lData := uint(len(p))
	lData = min(lData, rb.size-rb.in+rb.out)
	l := min(lData, rb.size-(rb.in&(rb.size-1)))
	copy(rb.data[rb.in&(rb.size-1):], p[:l])
	copy(rb.data[:lData-l], p[l:])
	rb.in += lData
	return int(lData), nil
}

func (rb *RingBuffer) Read(p []byte) (n int, err error) {
	lData := uint(len(p))
	lData = min(lData, rb.in-rb.out)
	l := min(lData, rb.size-(rb.out&(rb.size-1)))
	copy(p, rb.data[rb.out&(rb.size-1):rb.out&(rb.size-1)+l])
	copy(p[l:], rb.data[:lData-l])
	rb.out += lData
	return int(lData), nil
}

func min(x, y uint) uint {
	if x < y {
		return x
	}
	return y
}
```


# Linux内核的Kfifo原版实现

```c
struct kfifo {   
    unsigned char *buffer;    /* the buffer holding the data */   
    unsigned int size;    /* the size of the allocated buffer */   
    unsigned int in;    /* data is added at offset (in % size) */   
    unsigned int out;    /* data is extracted from off. (out % size) */   
    spinlock_t *lock;    /* protects concurrent modifications */   
};

struct kfifo *kfifo_alloc(unsigned int size, gfp_t gfp_mask, spinlock_t *lock)   
{   
    unsigned char *buffer;   
    struct kfifo *ret;   
  
    /*  
     * round up to the next power of 2, since our 'let the indices  
     * wrap' tachnique works only in this case.  
     */   
    if (size & (size - 1)) {   
        BUG_ON(size > 0x80000000);   
        size = roundup_pow_of_two(size);   
    }   
  
    buffer = kmalloc(size, gfp_mask);   
    if (!buffer)   
        return ERR_PTR(-ENOMEM);   
  
    ret = kfifo_init(buffer, size, gfp_mask, lock);   
  
    if (IS_ERR(ret))   
        kfree(buffer);   
  
    return ret;   
}

unsigned int __kfifo_put(struct kfifo *fifo,   
             unsigned char *buffer, unsigned int len)   
{   
    unsigned int l;   
  
    len = min(len, fifo->size - fifo->in + fifo->out);   
  
    /*  
     * Ensure that we sample the fifo->out index -before- we  
     * start putting bytes into the kfifo.  
     */   
  
    smp_mb();   
  
    /* first put the data starting from fifo->in to buffer end */   
    l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));   
    memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), buffer, l);   
  
    /* then put the rest (if any) at the beginning of the buffer */   
    memcpy(fifo->buffer, buffer + l, len - l);   
  
    /*  
     * Ensure that we add the bytes to the kfifo -before-  
     * we update the fifo->in index.  
     */   
  
    smp_wmb();   
  
    fifo->in += len;   
  
    return len;   
}  
  
unsigned int __kfifo_get(struct kfifo *fifo,   
             unsigned char *buffer, unsigned int len)   
{   
    unsigned int l;   
  
    len = min(len, fifo->in - fifo->out);   
  
    /*  
     * Ensure that we sample the fifo->in index -before- we  
     * start removing bytes from the kfifo.  
     */   
  
    smp_rmb();   
  
    /* first get the data from fifo->out until the end of the buffer */   
    l = min(len, fifo->size - (fifo->out & (fifo->size - 1)));   
    memcpy(buffer, fifo->buffer + (fifo->out & (fifo->size - 1)), l);   
  
    /* then get the rest (if any) from the beginning of the buffer */   
    memcpy(buffer + l, fifo->buffer, len - l);   
  
    /*  
     * Ensure that we remove the bytes from the kfifo -before-  
     * we update the fifo->out index.  
     */   
  
    smp_mb();   
  
    fifo->out += len;   
  
    return len;   
}   

```
