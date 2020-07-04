### 写在前面

#### gqlgen介绍

> gqlgen is a Go library for building GraphQL servers without any fuss

gqlgen是构建Graphql服务的golang框架，相比golang的其他框架，单单是独立的schema文件就简单很多

### 源码阅读方式

先从官方主库的example例子入手，通过调试和跳转查看代码的调用流程，进而大概知道框架的结构

### 第一个例子

首先从最简单的todo开始

#### schema
```go
schema {
    query: MyQuery
    mutation: MyMutation
}

type MyQuery {
    todo(id: ID!): Todo
    lastTodo: Todo
    todos: [Todo!]!
}

type MyMutation {
    createTodo(todo: TodoInput!): Todo!
    updateTodo(id: ID!, changes: Map!): Todo
}

type Todo {
    id: ID!
    text: String!
    done: Boolean! @hasRole(role: OWNER) # only the owner can see if a todo is done
}

"Passed to createTodo to create a new todo"
input TodoInput {
    "The body text"
    text: String!
    "Is it done already?"
    done: Boolean
}

scalar Map

"Prevents access to a field if the user doesnt have the matching role"
directive @hasRole(role: Role!) on FIELD_DEFINITION
directive @user(id: ID!) on MUTATION | QUERY | FIELD

enum Role {
    ADMIN
    OWNER
}
```
#### config
```yml
schema:
  - schema.graphql
models:
  Todo:
    model: todo.Todo
  ID:
    model: # override the default id marshaller to use ints
      - github.com/99designs/gqlgen/graphql.IntID
      - github.com/99designs/gqlgen/graphql.ID
```
运行```
go run github.com/99designs/gqlgen generate``` 生成resolvers

#### server
```go
import (
	"context"
	"errors"
	"github.com/99designs/gqlgen/graphql/handler"
	"github.com/99designs/gqlgen/graphql/playground"
	"gqlgen_example/todo"
	"log"
	"net/http"
	"runtime/debug"
)

func main() {
	srv := handler.NewDefaultServer(todo.NewExecutableSchema(todo.New()))
	srv.SetRecoverFunc(func(ctx context.Context, err interface{}) (userMessage error) {
		// send this panic somewhere
		log.Print(err)
		debug.PrintStack()
		return errors.New("user message on panic")
	})

	http.Handle("/", playground.Handler("Todo", "/query"))
	http.Handle("/query", srv)
	log.Fatal(http.ListenAndServe(":8081", nil))
}
```
这样一个简单graphql服务就完成了

### 源码调试

接着开始对着源码一行行调试，大概理清整个服务的流程

```go
srv := handler.NewDefaultServer(todo.NewExecutableSchema(todo.New()))
```
```go
// generated.go

// NewExecutableSchema creates an ExecutableSchema from the ResolverRoot interface.
func NewExecutableSchema(cfg Config) graphql.ExecutableSchema {
	return &executableSchema{
		resolvers:  cfg.Resolvers,
		directives: cfg.Directives,
		complexity: cfg.Complexity,
	}
}
```
```go
//github.com/99designs/gqlgen/graphql/executable_scheme.go

type ExecutableSchema interface {
	Schema() *ast.Schema	

	Complexity(typeName, fieldName string, childComplexity int, args map[string]interface{}) (int, bool)
	Exec(ctx context.Context) ResponseHandler
}
```
**Schema**返回包含了整个schema文件中定义的解析,schema文件如何解析以后再讲。**Complexity**表示请求查询时的复杂度,**Exec**返回请求结果,executableSchema实现了ExecutableSchema接口
```go
//github.com/99designs/gqlgen/graphql/handler/server.go

func New(es graphql.ExecutableSchema) *Server {
	return &Server{
		exec: executor.New(es),	//这个executor很重要，整个服务的请求处理都是通过exector来进行的
	}
}

func NewDefaultServer(es graphql.ExecutableSchema) *Server {
	srv := New(es)

	//设置transport,作用相当于net/http中的transport
	srv.AddTransport(transport.Websocket{
		KeepAlivePingInterval: 10 * time.Second,
	})
	srv.AddTransport(transport.Options{})
	srv.AddTransport(transport.GET{})
	srv.AddTransport(transport.POST{})
	srv.AddTransport(transport.MultipartForm{})

	srv.SetQueryCache(lru.New(1000))

	srv.Use(extension.Introspection{})
	srv.Use(extension.AutomaticPersistedQuery{
		Cache: lru.New(100),
	})

	return srv
}
```
```go
//github.com/99designs/gqlgen/graphql/executor/executor.go

// Executor executes graphql queries against a schema.
type Executor struct {
	es         graphql.ExecutableSchema
	extensions []graphql.HandlerExtension
	ext        extensions

	errorPresenter graphql.ErrorPresenterFunc
	recoverFunc    graphql.RecoverFunc
	queryCache     graphql.Cache
}

var _ graphql.GraphExecutor = &Executor{}

// New creates a new Executor with the given schema, and a default error and
// recovery callbacks, and no query cache or extensions.
func New(es graphql.ExecutableSchema) *Executor {
	e := &Executor{
		es:             es,
		errorPresenter: graphql.DefaultErrorPresenter,
		recoverFunc:    graphql.DefaultRecover,
		queryCache:     graphql.NoCache{},
		ext:            processExtensions(nil),
	}
	return e
}
```
回到自己定义的server这边,设置中间件，设置路由，启动服务
```go
//设置服务出现panic时的一个中间件
srv.SetRecoverFunc(func(ctx context.Context, err interface{}) (userMessage error) {
		// send this panic somewhere
		log.Print(err)
		debug.PrintStack()
		return errors.New("user message on panic")
})

http.Handle("/", playground.Handler("Todo", "/query"))
http.Handle("/query", srv)
log.Fatal(http.ListenAndServe(":8081", nil))
```