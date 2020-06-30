### 写在前面

#### gqlgen介绍

> gqlgen is a Go library for building GraphQL servers without any fuss

gqlgen是构建Graphql服务的golang框架，相比golang的其他框架，单单是独立的scheme文件就简单很多

### 源码阅读方式

先从官方主库的example例子入手，通过调试和跳转查看代码的调用流程，进而大概知道框架的结构

### 第一个例子

首先从最简单的todo开始

#### scheme
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