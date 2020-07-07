### 写在前面
本节还是以todo为例，通过跳转调试来获取gqlgen处理请求的整个流程
### todo server
```go
// 定义默认的server
srv := handler.NewDefaultServer(todo.NewExecutableSchema(todo.New()))
// 设置panic中间件
srv.SetRecoverFunc(func(ctx context.Context, err interface{}) (userMessage error) {
		// send this panic somewhere
	log.Print(err)
	debug.PrintStack()
	return errors.New("user message on panic")
})

// 设置路由
http.Handle("/", playground.Handler("Todo", "/query"))
http.Handle("/query", srv)
log.Fatal(http.ListenAndServe(":8081", nil))
```

然后来看看请求
```json
mutation{
    createTodo(todo:{
        text:"Fery important"
    })
    {
        id
    }
}
```
的流程是怎样的，http请求来到ServeHTTP
```go
// github.com/99designs/gqlgen/graphql/handler/server.go

func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	defer func() {
		if err := recover(); err != nil {
			err := s.exec.PresentRecoveredError(r.Context(), err)
			resp := &graphql.Response{Errors: []*gqlerror.Error{err}}
			b, _ := json.Marshal(resp)
			w.WriteHeader(http.StatusUnprocessableEntity)
			w.Write(b)
		}
	}()

	r = r.WithContext(graphql.StartOperationTrace(r.Context()))

    // 获取transport,在NewDefaultServer函数中已添加http请求中的post，get等方式请求的transport
	transport := s.getTransport(r)
	if transport == nil {
		sendErrorf(w, http.StatusBadRequest, "transport not supported")
		return
	}

    // 开始处理请求
	transport.Do(w, r, s.exec)
}
```
```go
// github.com/99designs/gqlgen/graphql/handler/transport/http_post.go

func (h POST) Do(w http.ResponseWriter, r *http.Request, exec graphql.GraphExecutor) {
	w.Header().Set("Content-Type", "application/json")

	var params *graphql.RawParams
	start := graphql.Now()
	if err := jsonDecode(r.Body, &params); err != nil {
		w.WriteHeader(http.StatusBadRequest)
		writeJsonErrorf(w, "json body could not be decoded: "+err.Error())
		return
	}
	params.ReadTime = graphql.TraceTiming{
		Start: start,
		End:   graphql.Now(),
	}

	rc, err := exec.CreateOperationContext(r.Context(), params)
	if err != nil {
		w.WriteHeader(statusFor(err))
		resp := exec.DispatchError(graphql.WithOperationContext(r.Context(), rc), err)
		writeJson(w, resp)
		return
	}
	ctx := graphql.WithOperationContext(r.Context(), rc)
	responses, ctx := exec.DispatchOperation(ctx, rc)
	writeJson(w, responses(ctx))
}
```

