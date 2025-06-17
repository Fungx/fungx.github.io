---
title: go-redis 连接池原理
date: 2025-06-18 01:51:00 +0800
categories: [Golang]
tags: [golang, go-redis, pool]     # TAG names should always be lowercase
---
# 前言
在之前的工作中，曾经遇到过RPC框架连接池负载均衡失效的问题。由此，也对常用的各种 SDK 的连接池实现产生了兴趣。
打算用这篇文章作为开始，学习 go-redis 连接池的设计思想与实现。最后，期望能总结一些使用的经验。

代码：[go-redis repo](https://github.com/redis/go-redis)

# 配置
在深入代码前，让我们先了解一下 go-redis 关于连接池有哪些配置。考虑到实际项目中基本都是用 redis 集群，后面的例子也会以集群对应的 client 为例。

[ClusterOptions](https://github.com/redis/go-redis/blob/c609828c9b0cdeaf1d5ee5cce12778a3bbf5f829/osscluster.go#L33) 定义了初始化 client 时的所有配置项，这里只展示和连接池比较相关的部分。

一般来说，在各种常见的SDK实现里，每个 `host` 都会有自己的一个连接池。例如，下面配置里的 `ClusterOptions.MinIdleConns`，就每个集群节点 `clusterNode` 的最小空闲连接数，而不是整个 `client` 的最小空闲连接数。

```go
type ClusterOptions struct {
	// A seed list of host:port addresses of cluster nodes.
	Addrs []string

	PoolFIFO        bool
	PoolSize        int // applies per cluster node and not for the whole cluster
	PoolTimeout     time.Duration
	MinIdleConns    int
	MaxIdleConns    int
	MaxActiveConns  int // applies per cluster node and not for the whole cluster
	ConnMaxIdleTime time.Duration
	ConnMaxLifetime time.Duration
    // ...
}
```

# 结构
连接池的实现都在包路径 `/internal/pool` 下，我们从 `/internal/pool/pool.go` 先初步感受下连接池的整体结构。

`Pooler` 接口提供了对连接池的各种操作，`clusterNode` 会引用 `Pooler` 来初始化、获取以及释放连接。

```go
type Pooler interface {
	NewConn(context.Context) (*Conn, error)
	CloseConn(*Conn) error

	Get(context.Context) (*Conn, error)
	Put(context.Context, *Conn)
	Remove(context.Context, *Conn, error)

	Len() int
	IdleLen() int
	Stats() *Stats

	Close() error
}
```
`Pooler` 接口被 `ConnPool` 实现。其中 `conns` 和 `idleConns` 分别保存了所有连接 以及 空闲连接；还有`queue` 这是一个带缓冲区的 `channel`，它用于在 `checkMinIdleConns` 中控制创建连接的速度，以及重试。

`NewConnPool` 是初始化连接池的函数，可以看到里面的逻辑很简单。它只是获取了 `connsMu` 锁，然后调用了`checkMinIdleConns` 来使连接池的空闲连接数达到配置中的`MinIdleConns`.
```go
type ConnPool struct {
	cfg *Options

	dialErrorsNum uint32 // atomic
	lastDialError atomic.Value

	queue chan struct{} // 带缓冲区的channel，用于控制初始化连接的速度

	connsMu   sync.Mutex
	conns     []*Conn // 全部连接
	idleConns []*Conn // 空闲连接

	poolSize     int // 当前连接数
	idleConnsLen int // 当前空闲连接数

	stats Stats

	_closed uint32 // atomic
}

var _ Pooler = (*ConnPool)(nil)

func NewConnPool(opt *Options) *ConnPool {
	p := &ConnPool{
		cfg: opt,

		queue:     make(chan struct{}, opt.PoolSize),
		conns:     make([]*Conn, 0, opt.PoolSize),
		idleConns: make([]*Conn, 0, opt.PoolSize),
	}

	p.connsMu.Lock()
	p.checkMinIdleConns()
	p.connsMu.Unlock()

	return p
}
```
# 初始化
## 大致流程
在深入 `NewConnPool` 的代码前，我们先在 `NewConnPool` 上打个断点，了解 `ConnPool` 与其他模块的关系。
我们用以下的代码，初始化一个 client，并执行一次 `Get`。
```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

func main() {
	cli := redis.NewClusterClient(&redis.ClusterOptions{
		Addrs: []string{
			"localhost:7010",
		},
		MinIdleConns: 10,
	})

	cmd := cli.Get(context.Background(), "key")
	if err := cmd.Err(); err != nil {
		fmt.Printf("err: %v\n", err)
		return
	}

	fmt.Printf("got: %v", cmd.Val())
}
```

这是执行 `NewConnPool` 前的调用栈。

![alt text](../assets/lib/2025-06-03-go-redis-connection-pool/stack.png)

可以看到，连接池初始化的大致流程如下
1. `client` 调用 redis 命令，需要在 `(*ClusterClient).process` 里先计算当前操作 key 对应的 `slot`，并根据 `slot` 通过 `c.cmdNode` 获取 `clusterNode`.
   ```go
   func (c *ClusterClient) process(ctx context.Context, cmd Cmder) error {
        slot := c.cmdSlot(ctx, cmd) // 计算key对应的slot
        var node *clusterNode
        var moved bool
        var ask bool
        var lastErr error
        for attempt := 0; attempt <= c.opt.MaxRedirects; attempt++ {
            // MOVED and ASK responses are not transient errors that require retry delay; they
            // should be attempted immediately.
            if attempt > 0 && !moved && !ask {
                if err := internal.Sleep(ctx, c.retryBackoff(attempt)); err != nil {
                    return err
                }
            }

            if node == nil {
                var err error
                node, err = c.cmdNode(ctx, cmd.Name(), slot) // 获取slot对应的 clusterNode
                if err != nil {
                    return err
                }
            }
            // ...
    }
   ```
2. 因为第一次调用时 `clusterNode` 还未初始化，这时会调用 `newClusterNode` 来创建一个集群节点。`clusterNode` 的核心是一个`Client`. 这个`Client` 就是单机模式的 `Client` 实现，go-redis 中的集群模式其实是对单机模式的封装，最底层的命令执行逻辑都是一样的。
   ```go
   func newClusterNode(clOpt *ClusterOptions, addr string) *clusterNode {
        opt := clOpt.clientOptions()
        opt.Addr = addr
        node := clusterNode{
            Client: clOpt.NewClient(opt),
        }

        node.latency = math.MaxUint32
        if clOpt.RouteByLatency {
            go node.updateLatency()
        }

        return &node
    }
   ```
3. `NewClient` 调用了 `newConnPool` 完成连接池的初始化。值得注意的是，单机模式下连接池是在创建 `Client` 时立刻初始化的，这与集群模式的懒加载不一样。
    ```go
    // NewClient returns a client to the Redis Server specified by Options.
    func NewClient(opt *Options) *Client {
        opt.init()

        c := Client{
            baseClient: &baseClient{
                opt: opt,
            },
        }
        c.init()
        c.connPool = newConnPool(opt, c.dialHook)

        return &c
    }
    ```

## checkMinIdleConns
上面的流程最后调用了 `newConnPool`，而里面实际执行初始化的函数是 `checkMinIdleConns`.
```go
func newConnPool(
	opt *Options,
	dialer func(ctx context.Context, network, addr string) (net.Conn, error),
) *pool.ConnPool {
	return pool.NewConnPool(&pool.Options{
		Dialer: func(ctx context.Context) (net.Conn, error) {
			return dialer(ctx, opt.Network, opt.Addr)
		},
		PoolFIFO:        opt.PoolFIFO,
		PoolSize:        opt.PoolSize,
		PoolTimeout:     opt.PoolTimeout,
		DialTimeout:     opt.DialTimeout,
		MinIdleConns:    opt.MinIdleConns,
		MaxIdleConns:    opt.MaxIdleConns,
		MaxActiveConns:  opt.MaxActiveConns,
		ConnMaxIdleTime: opt.ConnMaxIdleTime,
		ConnMaxLifetime: opt.ConnMaxLifetime,
	})
}

func NewConnPool(opt *Options) *ConnPool {
	p := &ConnPool{
		cfg: opt,

		queue:     make(chan struct{}, opt.PoolSize), // 控制创建连接的并发度
		conns:     make([]*Conn, 0, opt.PoolSize),
		idleConns: make([]*Conn, 0, opt.PoolSize),
	}

	p.connsMu.Lock()
	p.checkMinIdleConns() // 实际的初始化逻辑
	p.connsMu.Unlock()

	return p
}
```

`checkMinIdelConns` 异步地创建连接并添加到连接池里。注意这里`p.queue` 使用 `make(chan struct{}, opt.PoolSize)` 初始化，留出了缓冲区。在缓冲区填满前，不会在创建连接处阻塞。通过 `p.queue` 可以 保证同一时刻最多只有 `opt.PoolSize` 个协程在创建连接。

另外，这里 `p.queue` 的含义可以理解为一个信号量。`p.queue` 缓冲区中的token数量等于**使用中**的连接数。
在后面我们可以看到，每次获取和创建连接时，会执行 `p.queue <- struct{}{}` 放入token；而放回连接时，则会执行 `<- p.queue` 取出token。

通过这种设计，go-redis 实现了连接获取与释放的并发控制。也保证了连接操作的吞吐量，同一时刻，最多可以有 `opt.PoolSize` 个连接被使用。

```go
func (p *ConnPool) checkMinIdleConns() {
	if p.cfg.MinIdleConns == 0 {
		return
	}
	for p.poolSize < p.cfg.PoolSize && p.idleConnsLen < p.cfg.MinIdleConns {
		select {
		case p.queue <- struct{}{}:
			p.poolSize++
			p.idleConnsLen++

			go func() {
				err := p.addIdleConn()
				if err != nil && err != ErrClosed {
					p.connsMu.Lock()
					p.poolSize--
					p.idleConnsLen--
					p.connsMu.Unlock()
				}

				p.freeTurn()
			}()
		default:
			return
		}
	}
}
```
另外，这里也有一些细节：
1. 因为连接是在自增了 `poolSize` 和 `idleConnsLen` 后，异步创建的。所以他们表示的连接数包含了 **正在创建中的连接**。
2. 除了创建连接时，`checkMinIdleConns` 还会在 `(*ConnPool).popIdle` （获取空闲连接）以及 `(*ConnPool).removeConn`（移除连接）时被调用，以确保连接池能及时补充空闲连接。
   
# 连接的获取与释放
## 获取
通过 `(*ConnPool).Get` 可以获取一个可用的连接。首先，线程会调用 `waitTurn` ，尝试往 `p.queue` 里放入一个token。如果队列已满，说明当前使用中的连接已经达到最大值，则通过 `p.queue <- struct{}{}` 阻塞一段时间，等待其他线程释放连接。如果成功放入token，就可以通过 `popIdle` 取出连接；否则发生超时，会尝试自己创建一个连接。

在获取连接，以及新建连接失败时，都会调用 `freeTurn` ，从 `p.queue` 中取出token。表示这次操作没有使用中的连接。

```go
// Get returns existed connection from the pool or creates a new one.
func (p *ConnPool) Get(ctx context.Context) (*Conn, error) {
    // 检查连接池是否关闭
	if p.closed() {
		return nil, ErrClosed
	}
    // 尝试放入令牌，这里如果超时会返回err
	if err := p.waitTurn(ctx); err != nil {
		return nil, err
	}
    
    // 循环直到拿到连接
	for {
        // 对连接池加锁
		p.connsMu.Lock()
        // 从 p.idleConns 中取出一个连接
		cn, err := p.popIdle()
		p.connsMu.Unlock()

		if err != nil {
            // 如果获取连接失败，取出令牌
			p.freeTurn()
			return nil, err
		}
        // 没有可用的连接，退出循环，尝试新建一个连接
		if cn == nil {
			break
		}
        // 如果连接已经不可用，则关闭他，重新拿一个
		if !p.isHealthyConn(cn) {
			_ = p.CloseConn(cn)
			continue
		}

		atomic.AddUint32(&p.stats.Hits, 1)
        // 拿到连接，返回
		return cn, nil
	}

	atomic.AddUint32(&p.stats.Misses, 1)
    // 没有可用连接时，新建一个
	newcn, err := p.newConn(ctx, true)
	if err != nil {
        // 新建失败，取出令牌
		p.freeTurn()
		return nil, err
	}

	return newcn, nil
}
```
## 释放
连接使用结束后，会通过 `(*ConnPool).Put` 释放连接。

释放连接的核心 `p.Remove` 函数依次做了3件事：从连接池中删除连接、从`p.queue`取出token、关闭当前连接。

首先，如果连接状态异常，或原先不在连接池中，则会调用 `p.Remove` 进行释放。而对于正常使用完成的连接，如果连接池有空闲，则放回连接池。

对于连接池已满的情况，这里处理比较特殊。可以看到没有直接调用 `p.Remove` ，而是拆开了里面的3个步骤：

1. `p.removeConn` 从连接池中删除连接，标记需要关闭连接，然后**释放连接池锁**
2. `p.freeTurn` 取出token
3. `p.closeConn` 关闭连接

这是因为连接池每一次 `Get` 和 `Put` 都要获取一把互斥锁，为了提高吞吐量，在执行完连接池相关的操作后需要立刻释放锁。

```go
func (p *ConnPool) Put(ctx context.Context, cn *Conn) {
    // 连接里的数据没读完，发生异常
	if cn.rd.Buffered() > 0 {
		internal.Logger.Printf(ctx, "Conn has unread data")
        // 关闭连接，并把连接从连接池中删除
		p.Remove(ctx, cn, BadConnError{})
		return
	}
    // 如果连接不在连接池里，删除
	if !cn.pooled {
		p.Remove(ctx, cn, nil)
		return
	}

	var shouldCloseConn bool

	p.connsMu.Lock()

	if p.cfg.MaxIdleConns == 0 || p.idleConnsLen < p.cfg.MaxIdleConns {
        // 没达到最大空闲连接数，把连接放回到连接池队尾
		p.idleConns = append(p.idleConns, cn)
		p.idleConnsLen++
	} else {
        // 否则，从连接池中删除连接，并标记关闭连接
		p.removeConn(cn)
		shouldCloseConn = true
	}

	p.connsMu.Unlock()

    // 从 p.queue 取出token
	p.freeTurn()

    // 关闭连接
	if shouldCloseConn {
		_ = p.closeConn(cn)
	}
}
```

其实在 `p.Remove` 里，也是一样的处理方式。「从连接池中删除连接」这个行为，抽出了一个独立的函数 `P.removeConnWithLock` 并在内部加锁。
```go
func (p *ConnPool) Remove(_ context.Context, cn *Conn, reason error) {
	p.removeConnWithLock(cn)
	p.freeTurn()
	_ = p.closeConn(cn)
}
```
## FIFO 与 FILO 的区别
开头提到，连接的获取有 FIFO 与 FILO 两种方式，实现区别就在 `(*ConnPool).popIdld` 中，默认是 FILO.

```go
func (p *ConnPool) popIdle() (*Conn, error) {
	if p.closed() {
		return nil, ErrClosed
	}
	n := len(p.idleConns)
	if n == 0 {
		return nil, nil
	}

	var cn *Conn
	if p.cfg.PoolFIFO {
		cn = p.idleConns[0]
		copy(p.idleConns, p.idleConns[1:])
		p.idleConns = p.idleConns[:n-1]
	} else {
		idx := n - 1
		cn = p.idleConns[idx]
		p.idleConns = p.idleConns[:idx]
	}
	p.idleConnsLen--
	p.checkMinIdleConns()
	return cn, nil
}
```
关于 FIFO 的引入背景，可以看 [issue#1819](https://github.com/redis/go-redis/issues/1819). 

issue 中描述的场景如下图所示。他们通过 一个 HostName 作为集群的门户，使用单个 client 来接入整个集群。
每次创建连接时，会经过 DNS 拿到一个 Proxy 节点的 IP，并对其建立连接。预期每个 Proxy 节点是负载均衡的。

但实际上，由于连接池默认使用了 FILO，连接的 放入/取出 都是在队列末尾进行。导致每个连接被获取的机会不是均等的。使用频率越高的连接，越容易被获取。这会导致大量的流量倾斜到少数的 若干个 Proxy 节点上，负载均衡失效。

FIFO 则可以让每个连接都有平等的机会被使用，在这种场景下依然能够保证负载均衡。


![redis 集群](../assets/lib/2025-06-03-go-redis-connection-pool/redis-cluster.png)

另外，FILO 除了解决负载均衡的问题外，与 FIFO 也有一些适用场景的区别
 
| 特性 | FIFO (先进先出) | FILO (后进先出) |
|------|---------------|-------------------|
| 获取顺序 | 从队列头部获取 | 从队列尾部获取 |
| 性能开销 | 略高，每一次获取连接都比FILO多了一次 `copy` | 略低 |
| 空闲连接关闭 | 更快关闭空闲连接，有助于减小连接池大小 | 空闲连接关闭较慢 |
| 适用场景 | 需要频繁关闭空闲连接的场景 | 常规场景，默认选择 |


# 总结
这篇文章初步分析了 go-redis 连接池的实现原理。可以看到实现还是比较简单的，比如 连接池直接用了 slice 而不是 队列来实现；连接池本身也没有使用无锁的数据结构，而是简单靠一把互斥锁做并发控制。但显然，这些设计已经足够使用了，不然也不会存在那么久。

另外，比较有意思的就是 FIFO 和 FILO 的区别。之前工作中也遇到过类似的场景，我们作为上游通过单个 VIP 访问下游的集群，上游服务完全无法感知下游的节点数量和状态。这种情况下，RPC 框架本身的负载均衡失效了，导致下游每个节点的负载差距非常大，经常出现单节点过载。

在我的场景里，如果连接池使用了 FIFO，可以解决当时的问题吗？感觉可以有很大的优化，将来有空再写篇文章看看 Kitex 这个 RPC 框架的连接池。