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

	// decode请求的字符串
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

	// 生成整个请求周期的ctx，处理一些中间件，检验请求参数，请求复杂度，持久化查询等
	rc, err := exec.CreateOperationContext(r.Context(), params)
	if err != nil {
		w.WriteHeader(statusFor(err))
		resp := exec.DispatchError(graphql.WithOperationContext(r.Context(), rc), err)
		writeJson(w, resp)
		return
	}

	// 开始真正处理请求，访问自己写的服务
	ctx := graphql.WithOperationContext(r.Context(), rc)
	responses, ctx := exec.DispatchOperation(ctx, rc)
	writeJson(w, responses(ctx))
}
```
```go
// github.com/99designs/gqlgen/graphql/executor/executor.go

/*
* Executor是处理请求的核心
*/

// Executor executes graphql queries against a schema.
type Executor struct {
	es         graphql.ExecutableSchema
	extensions []graphql.HandlerExtension	
	ext        extensions	//自定义的中间件

	errorPresenter graphql.ErrorPresenterFunc	
	recoverFunc    graphql.RecoverFunc	// 出现panic时需要处理的中间件，比如打印发送日志
	queryCache     graphql.Cache	// 请求的缓存相关
}

...

func (e *Executor) CreateOperationContext(ctx context.Context, params *graphql.RawParams) (*graphql.OperationContext, gqlerror.List) {
	rc := &graphql.OperationContext{
		DisableIntrospection: true,
		Recover:              e.recoverFunc,	//这里就是自定义服务里设置的中间件
		ResolverMiddleware:   e.ext.fieldMiddleware,	// 请求返回字段的中间件
		Stats: graphql.Stats{
			Read:           params.ReadTime,
			OperationStart: graphql.GetStartTime(ctx),
		},
	}
	ctx = graphql.WithOperationContext(ctx, rc)

	// 中间件，这里处理持久化查询
	for _, p := range e.ext.operationParameterMutators {
		if err := p.MutateOperationParameters(ctx, params); err != nil {
			return rc, gqlerror.List{err}
		}
	}

	rc.RawQuery = params.Query
	rc.OperationName = params.OperationName

	// 解析请求的字符串
	var listErr gqlerror.List
	rc.Doc, listErr = e.parseQuery(ctx, &rc.Stats, params.Query)
	if len(listErr) != 0 {
		return rc, listErr
	}

	// 检验请求是否合法
	rc.Operation = rc.Doc.Operations.ForName(params.OperationName)
	if rc.Operation == nil {
		return rc, gqlerror.List{gqlerror.Errorf("operation %s not found", params.OperationName)}
	}

	var err *gqlerror.Error
	rc.Variables, err = validator.VariableValues(e.es.Schema(), rc.Operation, params.Variables)
	if err != nil {
		errcode.Set(err, errcode.ValidationFailed)
		return rc, gqlerror.List{err}
	}
	rc.Stats.Validation.End = graphql.Now()

	// 中间件，这里处理请求复杂度等
	for _, p := range e.ext.operationContextMutators {
		if err := p.MutateOperationContext(ctx, rc); err != nil {
			return rc, gqlerror.List{err}
		}
	}

	return rc, nil
}
```
```go
// github.com/99designs/gqlgen/graphql/executor/executor.go

func (e *Executor) DispatchOperation(ctx context.Context, rc *graphql.OperationContext) (graphql.ResponseHandler, context.Context) {
	ctx = graphql.WithOperationContext(ctx, rc)

	var innerCtx context.Context
	res := e.ext.operationMiddleware(ctx, func(ctx context.Context) graphql.ResponseHandler {
		innerCtx = ctx

		tmpResponseContext := graphql.WithResponseContext(ctx, e.errorPresenter, e.recoverFunc)
		// 这里会访问自定义的todo服务
		responses := e.es.Exec(tmpResponseContext)

		if errs := graphql.GetErrors(tmpResponseContext); errs != nil {
			return graphql.OneShot(&graphql.Response{Errors: errs})
		}

		return func(ctx context.Context) *graphql.Response {
			ctx = graphql.WithResponseContext(ctx, e.errorPresenter, e.recoverFunc)
			resp := e.ext.responseMiddleware(ctx, func(ctx context.Context) *graphql.Response {
				resp := responses(ctx)
				if resp == nil {
					return nil
				}
				resp.Errors = append(resp.Errors, graphql.GetErrors(ctx)...)
				resp.Extensions = graphql.GetExtensions(ctx)
				return resp
			})
			if resp == nil {
				return nil
			}

			return resp
		}
	})

	return res, innerCtx
}
```
```go
// todo/generated.go

func (e *executableSchema) Exec(ctx context.Context) graphql.ResponseHandler {
	rc := graphql.GetOperationContext(ctx)
	ec := executionContext{rc, e}
	first := true

	switch rc.Operation.Operation {
	case ast.Query:
		return func(ctx context.Context) *graphql.Response {
			if !first {
				return nil
			}
			first = false
			data := ec._queryMiddleware(ctx, rc.Operation, func(ctx context.Context) (interface{}, error) {
				return ec._MyQuery(ctx, rc.Operation.SelectionSet), nil
			})
			var buf bytes.Buffer
			data.MarshalGQL(&buf)

			return &graphql.Response{
				Data: buf.Bytes(),
			}
		}
	case ast.Mutation:
		return func(ctx context.Context) *graphql.Response {
			if !first {
				return nil
			}
			first = false
			data := ec._mutationMiddleware(ctx, rc.Operation, func(ctx context.Context) (interface{}, error) {
				return ec._MyMutation(ctx, rc.Operation.SelectionSet), nil
			})
			var buf bytes.Buffer
			data.MarshalGQL(&buf)

			return &graphql.Response{
				Data: buf.Bytes(),
			}
		}

	default:
		return graphql.OneShot(graphql.ErrorResponse(ctx, "unsupported GraphQL operation"))
	}
}
```
```go
// todo/generated.go

func (ec *executionContext) _mutationMiddleware(ctx context.Context, obj *ast.OperationDefinition, next func(ctx context.Context) (interface{}, error)) graphql.Marshaler {

	// 处理请求中的指令参数
	for _, d := range obj.Directives {
		switch d.Name {
		case "user":
			rawArgs := d.ArgumentMap(ec.Variables)
			args, err := ec.dir_user_args(ctx, rawArgs)
			if err != nil {
				ec.Error(ctx, err)
				return graphql.Null
			}
			n := next
			next = func(ctx context.Context) (interface{}, error) {
				if ec.directives.User == nil {
					return nil, errors.New("directive user is not implemented")
				}
				return ec.directives.User(ctx, obj, n, args["id"].(int))
			}
		}
	}
	tmp, err := next(ctx)
	if err != nil {
		ec.Error(ctx, err)
		return graphql.Null
	}
	if data, ok := tmp.(graphql.Marshaler); ok {
		return data
	}
	ec.Errorf(ctx, `unexpected type %T from directive, should be graphql.Marshaler`, tmp)
	return graphql.Null

}
```
```go
// todo/generated.go

func (ec *executionContext) _MyMutation(ctx context.Context, sel ast.SelectionSet) graphql.Marshaler {
	// 获取mutation请求中操作的名称
	fields := graphql.CollectFields(ec.OperationContext, sel, myMutationImplementors)

	ctx = graphql.WithFieldContext(ctx, &graphql.FieldContext{
		Object: "MyMutation",
	})

	// 请求的结果
	out := graphql.NewFieldSet(fields)
	var invalids uint32
	for i, field := range fields {
		switch field.Name {
		case "__typename":
			out.Values[i] = graphql.MarshalString("MyMutation")
		case "createTodo":
			// 这里返回的都是一个接口，以便最后out.Dispatch()获取所有的结果
			out.Values[i] = ec._MyMutation_createTodo(ctx, field)
			if out.Values[i] == graphql.Null {
				invalids++
			}
		case "updateTodo":
			out.Values[i] = ec._MyMutation_updateTodo(ctx, field)
		default:
			panic("unknown field " + strconv.Quote(field.Name))
		}
	}
	// 调用获取请求结果的分发函数
	out.Dispatch()
	if invalids > 0 {
		return graphql.Null
	}
	return out
}
```
```go
// todo/generated.go

func (ec *executionContext) _MyMutation_createTodo(ctx context.Context, field graphql.CollectedField) (ret graphql.Marshaler) {
	defer func() {
		if r := recover(); r != nil {
			ec.Error(ctx, ec.Recover(ctx, r))
			ret = graphql.Null
		}
	}()
	fc := &graphql.FieldContext{
		Object:   "MyMutation",
		Field:    field,
		Args:     nil,
		IsMethod: true,
	}

	// 获取请求的参数
	ctx = graphql.WithFieldContext(ctx, fc)
	rawArgs := field.ArgumentMap(ec.Variables)
	args, err := ec.field_MyMutation_createTodo_args(ctx, rawArgs)
	if err != nil {
		ec.Error(ctx, err)
		return graphql.Null
	}
	fc.Args = args
	resTmp := ec._fieldMiddleware(ctx, nil, func(rctx context.Context) (interface{}, error) {
		ctx = rctx // use context from middleware stack in children
		// 终于到自己写的服务了
		return ec.resolvers.MyMutation().CreateTodo(rctx, args["todo"].(TodoInput))
	})

	if resTmp == nil {
		if !graphql.HasFieldError(ctx, fc) {
			ec.Errorf(ctx, "must not be null")
		}
		return graphql.Null
	}
	res := resTmp.(*Todo)
	fc.Result = res
	// 拿到服务的结果后，再根据请求中指定返回的字段返回结果
	return ec.marshalNTodo2ᚖgithubᚗcomᚋ99designsᚋgqlgenᚋexampleᚋtodoᚐTodo(ctx, field.Selections, res)
}
```
```go
// todo/todo.go

func (r *MutationResolver) CreateTodo(ctx context.Context, todo TodoInput) (*Todo, error) {
	newID := r.id()

	newTodo := &Todo{
		ID:    newID,
		Text:  todo.Text,
		owner: you,
	}

	if todo.Done != nil {
		newTodo.Done = *todo.Done
	}

	r.todos = append(r.todos, newTodo)

	return newTodo, nil
}
```
```go
// github.com/99designs/gqlgen/graphql/fieldset.go

// 获取结果
func (m *FieldSet) Dispatch() {
	if len(m.delayed) == 1 {
		// only one concurrent task, no need to spawn a goroutine or deal create waitgroups
		d := m.delayed[0]
		m.Values[d.i] = d.f()
	} else if len(m.delayed) > 1 {
		// more than one concurrent task, use the main goroutine to do one, only spawn goroutines for the others

		// 等待所有协程返回
		var wg sync.WaitGroup
		for _, d := range m.delayed[1:] {
			wg.Add(1)
			go func(d delayedResult) {
				m.Values[d.i] = d.f()
				wg.Done()
			}(d)
		}

		m.Values[m.delayed[0].i] = m.delayed[0].f()
		wg.Wait()
	}
}
```
```go
// todo/generated.go

func (ec *executionContext) marshalNTodo2ᚖgithubᚗcomᚋ99designsᚋgqlgenᚋexampleᚋtodoᚐTodo(ctx context.Context, sel ast.SelectionSet, v *Todo) graphql.Marshaler {
	if v == nil {
		if !graphql.HasFieldError(ctx, graphql.GetFieldContext(ctx)) {
			ec.Errorf(ctx, "must not be null")
		}
		return graphql.Null
	}
	return ec._Todo(ctx, sel, v)
}

...

// 根据指定的字段返回
func (ec *executionContext) _Todo(ctx context.Context, sel ast.SelectionSet, obj *Todo) graphql.Marshaler {
	fields := graphql.CollectFields(ec.OperationContext, sel, todoImplementors)

	out := graphql.NewFieldSet(fields)
	var invalids uint32
	for i, field := range fields {
		switch field.Name {
		case "__typename":
			out.Values[i] = graphql.MarshalString("Todo")
		case "id":
			out.Values[i] = ec._Todo_id(ctx, field, obj)
			if out.Values[i] == graphql.Null {
				invalids++
			}
		case "text":
			out.Values[i] = ec._Todo_text(ctx, field, obj)
			if out.Values[i] == graphql.Null {
				invalids++
			}
		case "done":
			out.Values[i] = ec._Todo_done(ctx, field, obj)
			if out.Values[i] == graphql.Null {
				invalids++
			}
		default:
			panic("unknown field " + strconv.Quote(field.Name))
		}
	}
	out.Dispatch()
	if invalids > 0 {
		return graphql.Null
	}
	return out
}
```

