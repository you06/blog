title: TiDB表达式执行过程
route: tidb-expression-execflow
date: 2019-10-22T13:48:48.964Z
tags: 
    - 技术杂谈
--------------------------
# `TiDB`表达式执行过程

## 序言

`TiDB`向下兼容`MySQL`，但同时也支持许多`MySQL`里所没有的表达式，本文将描述表达式被解析和执行的过程。

<!-- more -->

## 表达式被解析的过程

我们先来看`TiDB`的源码，`server/server.go`中为`Server`定义`Run`方法里实现了监听连接的逻辑。

```go
func (s *Server) Run() error {
	...
	for {
		conn, err := s.listener.Accept()
		...
		go s.onConn(clientConn)
	}
}
```

在经过错误处理之后，这个连接对象被塞给了`onConn`方法，我们顺藤摸瓜继续看。

```go
// onConn runs in its own goroutine, handles queries from this connection.
func (s *Server) onConn(conn *clientConn) {
	ctx := logutil.WithConnID(context.Background(), conn.connectionID)
	...
	conn.Run(ctx)
}
```

经过一些处理之后，会调用`conn.Run`方法，继续看看`Run`方法里面写了些什么。

```go
// Run reads client query and writes query result to client in for loop, if there is a panic during query handling,
// it will be recovered and log the panic error.
// This function returns and the connection is closed if there is an IO error or there is a panic.
func (cc *clientConn) Run(ctx context.Context) {
	for {
		...
		data, err := cc.readPacket()
		...
		if err = cc.dispatch(ctx, data); err != nil {
			// error handle
		}
	}
}
```

确认过注释，就是我想要的代码，里面所调用的`cc.dispatch`就是处理单个查询的方法。

```go
// dispatch handles client request based on command which is the first byte of the data.
// It also gets a token from server which is used to limit the concurrently handling clients.
// The most frequently used command is ComQuery.
func (cc *clientConn) dispatch(ctx context.Context, data []byte) error {
	...
	cmd := data[0]
	data = data[1:]
	if cmd < mysql.ComEnd {
		cc.ctx.SetCommandValue(cmd)
	}

	dataStr := string(hack.String(data))

	switch cmd {
	...
	case mysql.ComQuery: // Most frequently used command.
		// For issue 1989
		// Input payload may end with byte '\0', we didn't find related mysql document about it, but mysql
		// implementation accept that case. So trim the last '\0' here as if the payload an EOF string.
		// See http://dev.mysql.com/doc/internals/en/com-query.html
		if len(data) > 0 && data[len(data)-1] == 0 {
			data = data[:len(data)-1]
			dataStr = string(hack.String(data))
		}
		return cc.handleQuery(ctx, dataStr)
	...
	default:
		return mysql.NewErrf(mysql.ErrUnknown, "command %d not supported now", cmd)
	}
}
```

这里我摘取了一部分 dispatch 的代码，以我们最常用的 query function 为例，接着看`handleQuery`函数。 

```go
func (cc *clientConn) handleQuery(ctx context.Context, sql string) (err error) {
	rs, err := cc.ctx.Execute(ctx, sql)
	...
}
```

`cc.ctx`是一个`QueryCtx`接口，找到实现了这个接口的`TiDBContext`结构体，在`server/drive_tidb.go`之中，实现的方法代码如下。

```go
// Execute implements QueryCtx Execute method.
func (tc *TiDBContext) Execute(ctx context.Context, sql string) (rs []ResultSet, err error) {
	rsList, err := tc.session.Execute(ctx, sql)
	...
}
```

`session.Execute`的实现在`session/session.go`之中，从`TiDBContext`之中调用的 sql 执行命令只是其中的一部分，还有其他 sql 调用也会在这里执行这里处理。

```go
type session struct {
	...
	parser *parser.Parser
	...
}

func (s *session) Execute(ctx context.Context, sql string) (recordSets []sqlexec.RecordSet, err error) {
	...
	charsetInfo, collation := s.sessionVars.GetCharsetInfo()
	if recordSets, err = s.execute(ctx, sql); err != nil {
		s.sessionVars.StmtCtx.AppendError(err)
	}
	return
}

func (s *session) execute(ctx context.Context, sql string) (recordSets []sqlexec.RecordSet, err error) {
	...
	stmtNodes, warns, err := s.ParseSQL(ctx, sql, charsetInfo, collation)
	...
}

func (s *session) ParseSQL(ctx context.Context, sql, charset, collation string) ([]ast.StmtNode, []error, error) {
	if span := opentracing.SpanFromContext(ctx); span != nil && span.Tracer() != nil {
		span1 := span.Tracer().StartSpan("session.ParseSQL", opentracing.ChildOf(span.Context()))
		defer span1.Finish()
	}
	s.parser.SetSQLMode(s.sessionVars.SQLMode)
	s.parser.EnableWindowFunc(s.sessionVars.EnableWindowFunction)
	return s.parser.Parse(sql, charset, collation)
}
```

进入到`session`的定义之中，一路追踪表达式被处理的流程，发现他被丢给了`parser`。

[`parser`](https://github.com/pingcap/parser)是一个被独立出来专门用来处理`TiDB`的表达式的组件。对于`parser`的逻辑解读将在下一篇博客中描述。

