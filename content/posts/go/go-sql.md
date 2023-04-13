---
title: "Go database/sql 源码解析"
date: 2023-04-04T17:00:38+08:00
draft: false
categories: ["Go"]
tags: [Go,Sql]
---

本文记录了在 Go 语言中实现数据库 API 的包`database/sql`的源码分析和个人理解。
<!--more-->

## 1. 驱动注册

官方的 database/sql 包旨在提供一个通用的数据库操作API，所以我们在使用不同的数据库系统时，都可以用这个官方包。这是因为与数据库底层的实现代码
相关的接口在 database/sql/driver包中，不同的数据库根据接口去实现适配自己的驱动即可。例如我们使用 MySQL 数据库，则需要先注册相关的驱动。

```go
_ "github.com/go-sql-driver/mysql"
_ "github.com/kshvakov/clickhouse"
```
匿名导入的目的是执行驱动包中的初始化函数。
```go
package mysql

import (
	"database/sql"
)
func init() {
	sql.Register("mysql", &MySQLDriver{})
}
```

## 2. 初始化一个DB

在使用真正的数据库连接进行数据查询前，我们会初始化一个DB结构体，此结构体维护了一个连接池，并发安全。另外开启了一个线程去监听一个通道，当需要连接时，
会向这个通道发送信号（空结构体），线程即去生成真正的连接。
```go
type DB struct {
    // Atomic access only. At top of struct to prevent mis-alignment
    // on 32-bit platforms. Of type time.Duration.
    waitDuration int64 // Total time waited for new connections.
    
    connector driver.Connector
    // numClosed is an atomic counter which represents a total number of
    // closed connections. Stmt.openStmt checks it before cleaning closed
    // connections in Stmt.css.
    numClosed uint64
    
    mu           sync.Mutex // protects following fields
    freeConn     []*driverConn  // 连接池
    connRequests map[uint64]chan connRequest    // 等待连接的请求
    nextRequest  uint64 // Next key to use in connRequests.
    numOpen      int    // number of opened and pending open connections
    // Used to signal the need for new connections
    // a goroutine running connectionOpener() reads on this chan and
    // maybeOpenNewConnections sends on the chan (one send per needed connection)
    // It is closed during db.Close(). The close tells the connectionOpener
    // goroutine to exit.
    openerCh          chan struct{}     // 需要新连接时往这里发信号
    closed            bool
    dep               map[finalCloser]depSet
    lastPut           map[*driverConn]string // stacktrace of last conn's put; debug only
    maxIdleCount      int                    // zero means defaultMaxIdleConns; negative means 0
    maxOpen           int                    // <= 0 means unlimited
    maxLifetime       time.Duration          // maximum amount of time a connection may be reused
    maxIdleTime       time.Duration          // maximum amount of time a connection may be idle before being closed
    cleanerCh         chan struct{}
    waitCount         int64 // Total number of connections waited for.
    maxIdleClosed     int64 // Total number of connections closed due to idle count.
    maxIdleTimeClosed int64 // Total number of connections closed due to idle time.
    maxLifetimeClosed int64 // Total number of connections closed due to max connection lifetime limit.
    
    stop func() // stop cancels the connection opener.
}

func OpenDB(c driver.Connector) *DB {
	ctx, cancel := context.WithCancel(context.Background())
	db := &DB{
		connector:    c,
		openerCh:     make(chan struct{}, connectionRequestQueueSize),
		lastPut:      make(map[*driverConn]string),
		connRequests: make(map[uint64]chan connRequest),
		stop:         cancel,
	}
    // 监听openerCh通道
	go db.connectionOpener(ctx)

	return db
}

```


## 3. 查询

以一个查询语句为例，看下整个流程。

首先通过重试+两种策略去查询。
```go
func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error) {
	var rows *Rows
	var err error
	// 重试2次 先从连接池拿连接
	for i := 0; i < maxBadConnRetries; i++ {
		rows, err = db.query(ctx, query, args, cachedOrNewConn)
		if err != driver.ErrBadConn {
			break
		}
	}
	if err == driver.ErrBadConn {
		return db.query(ctx, query, args, alwaysNewConn)
	}
	return rows, err
}
```

先获取一个 conn，再用拿到的 conn 去查询。
```go
func (db *DB) query(ctx context.Context, query string, args []interface{}, strategy connReuseStrategy) (*Rows, error) {
	// 先获取一个 conn 
	dc, err := db.conn(ctx, strategy)
	if err != nil {
		return nil, err
	}
    // 再查
	return db.queryDC(ctx, nil, dc, dc.releaseConn, query, args)
}
```

看下获取 conn 的过程。
```go
func (db *DB) conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error) {
	db.mu.Lock()
	if db.closed {
		db.mu.Unlock()
		return nil, errDBClosed
	}
	// Check if the context is expired.
	select {
	default:
	case <-ctx.Done():
		db.mu.Unlock()
		return nil, ctx.Err()
	}
	lifetime := db.maxLifetime

	// Prefer a free connection, if possible.
	numFree := len(db.freeConn)
	// 如果是从连接池获取且连接池的连接数量大于0
	if strategy == cachedOrNewConn && numFree > 0 {
		conn := db.freeConn[0]
		copy(db.freeConn, db.freeConn[1:])
		db.freeConn = db.freeConn[:numFree-1]
		conn.inUse = true
		if conn.expired(lifetime) {
			db.maxLifetimeClosed++
			db.mu.Unlock()
			conn.Close()
			return nil, driver.ErrBadConn
		}
		db.mu.Unlock()

		// Reset the session if required.
		if err := conn.resetSession(ctx); err == driver.ErrBadConn {
			conn.Close()
			return nil, driver.ErrBadConn
		}

		return conn, nil
	}

	// Out of free connections or we were asked not to use one. If we're not
	// allowed to open any more connections, make a request and wait.
	if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
		// Make the connRequest channel. It's buffered so that the
		// connectionOpener doesn't block while waiting for the req to be read.
		req := make(chan connRequest, 1)
		reqKey := db.nextRequestKeyLocked()
		db.connRequests[reqKey] = req
		db.waitCount++
		db.mu.Unlock()

		waitStart := nowFunc()

		// Timeout the connection request with the context.
		select {
		case <-ctx.Done():
			// Remove the connection request and ensure no value has been sent
			// on it after removing.
			db.mu.Lock()
			delete(db.connRequests, reqKey)
			db.mu.Unlock()

			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			select {
			default:
			case ret, ok := <-req:
				if ok && ret.conn != nil {
					db.putConn(ret.conn, ret.err, false)
				}
			}
			return nil, ctx.Err()
		case ret, ok := <-req:
			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			if !ok {
				return nil, errDBClosed
			}
			// Only check if the connection is expired if the strategy is cachedOrNewConns.
			// If we require a new connection, just re-use the connection without looking
			// at the expiry time. If it is expired, it will be checked when it is placed
			// back into the connection pool.
			// This prioritizes giving a valid connection to a client over the exact connection
			// lifetime, which could expire exactly after this point anyway.
			if strategy == cachedOrNewConn && ret.err == nil && ret.conn.expired(lifetime) {
				db.mu.Lock()
				db.maxLifetimeClosed++
				db.mu.Unlock()
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			if ret.conn == nil {
				return nil, ret.err
			}

			// Reset the session if required.
			if err := ret.conn.resetSession(ctx); err == driver.ErrBadConn {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			return ret.conn, ret.err
		}
	}

	db.numOpen++ // optimistically
	db.mu.Unlock()
	ci, err := db.connector.Connect(ctx)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
	db.mu.Lock()
	dc := &driverConn{
		db:         db,
		createdAt:  nowFunc(),
		returnedAt: nowFunc(),
		ci:         ci,
		inUse:      true,
	}
	db.addDepLocked(dc, dc)
	db.mu.Unlock()
	return dc, nil
}
```


总结一下连接获取的步骤：
1. 如果获取连接的策略为cachedOrNewConn且freeConn里面有空闲连接，直接返回拿到的conn
2. 如果已经建立的连接数达到最大上限且无空闲连接可用，则将请求加入db.connRequests等待队列，然后阻塞等到有连接可用时，这里拿到的是释放的连接，检查可用后返回
3. 如果不满足以上情况，则直接新建连接，创建成功则直接返回conn，失败则向DB的openerCh通道丢一个空的结构体，让监听该通道的goroutine去获取新的连接






## 4. 释放连接
连接使用关闭需要放入连接池，达到复用的过程，当然连接在使用前都会重置会话和检测连接是否可用。不可用的连接会进行关闭。
```go
// putConn adds a connection to the db's free pool.
// err is optionally the last error that occurred on this connection.
func (db *DB) putConn(dc *driverConn, err error, resetSession bool) {
	if err != driver.ErrBadConn {
		if !dc.validateConnection(resetSession) {
			err = driver.ErrBadConn
		}
	}
	db.mu.Lock()
	if !dc.inUse {
		db.mu.Unlock()
		if debugGetPut {
			fmt.Printf("putConn(%v) DUPLICATE was: %s\n\nPREVIOUS was: %s", dc, stack(), db.lastPut[dc])
		}
		panic("sql: connection returned that was never out")
	}

	if err != driver.ErrBadConn && dc.expired(db.maxLifetime) {
		db.maxLifetimeClosed++
		err = driver.ErrBadConn
	}
	if debugGetPut {
		db.lastPut[dc] = stack()
	}
	dc.inUse = false
	dc.returnedAt = nowFunc()

	for _, fn := range dc.onPut {
		fn()
	}
	dc.onPut = nil

	if err == driver.ErrBadConn {
		// Don't reuse bad connections.
		// Since the conn is considered bad and is being discarded, treat it
		// as closed. Don't decrement the open count here, finalClose will
		// take care of that.
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		dc.Close()
		return
	}
	if putConnHook != nil {
		putConnHook(db, dc)
	}
	added := db.putConnDBLocked(dc, nil)
	db.mu.Unlock()

	if !added {
		dc.Close()
		return
	}
}
```

## 5. 定期清理连接

如果设置了最大连接数和连接最大生存时间，DB 还会开启一个线程定时去检测连接的存活时间进行清理

```go
func (db *DB) connectionCleaner(d time.Duration) {
	const minInterval = time.Second

	if d < minInterval {
		d = minInterval
	}
	t := time.NewTimer(d)

	for {
		select {
		case <-t.C:
		case <-db.cleanerCh: // maxLifetime was changed or db was closed.
		}

		db.mu.Lock()

		d = db.shortestIdleTimeLocked()
		if db.closed || db.numOpen == 0 || d <= 0 {
			db.cleanerCh = nil
			db.mu.Unlock()
			return
		}

		closing := db.connectionCleanerRunLocked()
		db.mu.Unlock()
		for _, c := range closing {
			c.Close()
		}

		if d < minInterval {
			d = minInterval
		}
		t.Reset(d)
	}
}
```


## 6. 总结

通过看源码，学习到以下几点：
- 这种通用的包多使用接口，方便扩展。
- 底层逻辑和用户层代码分层设计，比如 driver 层和 DB 层。
- 锁的使用要小心谨慎且合理，比如避免重入锁。
- 符合开闭原则，比如通过添加新的接口去加入 context 的功能。

## 7. 参考

[类图生成工具](https://github.com/jfeliu007/goplantuml)

[类图在线生成平台](https://www.dumels.com/)

[Go SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers)

[数据库系统隔离级别](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Isolation_levels)