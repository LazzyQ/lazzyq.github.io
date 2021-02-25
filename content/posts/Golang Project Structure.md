---
title: "Golang Project Structure"
date: 2021-01-30T15:31:28+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - Golang基础
---

[https://www.youtube.com/watch?v=oL6JBUk6tj0](https://www.youtube.com/watch?v=oL6JBUk6tj0)

### Demo Project: A Beer Reviewing Service

- Users can add a beer
- Users  can add a  review for a  beer
- Users can list  all beers
- User  can list all reviews for  a given beer
- Option to  store data either in  memory or  in a JSON file
- Ability to  add  some sample data

<!--more-->

### Flat Structure

```jsx
flat
├── data.go
├── handlers.go
├── handlers_test.go
├── main.go
├── model.go
├── storage.go
├── storage_json.go
├── storage_json_test.go
├── storage_mem.go
├── storage_mem_test.go
└── storage_test.go
```

**优点**

- 结构简单，适合不太复杂的项目或者库
- 避免了循环依赖的问题，因为都在main包

缺点

- 全局作用域，无法做到访问控制
- 欠缺表达，只看文件名，无法知道项目的功能，必须看里面的逻辑才知道

### Group By Function(Layered Architecture)

- presentation/user interface
- business logic
- external dependencies/infrastructure

可以按照分层将代码放到不同层级的包下，换一些词比如controllers,models,views,daos这些在其他语言中经常出现的术语

```jsx
layered
├── data.go
├── handlers
│   ├── beers.go
│   └── reviews.go
├── main.go
├── models
│   ├── beer.go
│   ├── review.go
│   └── storage.go
└── storage
    ├── json.go
    └── memory.
```

**优点**

- 明确代码应该写在哪
- 访问控制，功能相同的放到同一包下

**缺点**

- 包之间共享变量不明确位置
- main初始化storage然后传递给每个model，还是每个model初始化依赖的storage
- 在这个例子，我们是用beers.go处理所有和beer相关的逻辑；还是用single_beer.go处理单个beer的逻辑，multi_beer.go处理多个beer的逻辑
- 开发者不友好，在这个例子，只定义了一个beer，也就意味着要把这个beer定义用在不同的场景，比如添加beer时，beer结构里面有个Id字段，但是当添加的时候你没有Id，当获取beer时才会有Id，所以作为一个使用者你不知道是否需要指定Id
- 循环引用，当编译这个项目的时候，会发现storage包依赖来model包中的定义，model包依赖storage包调用database
- 当项目更复杂之后，可能会导致可维护性降低
- 欠缺表达，这样的结构也没有很好的表达应用的功能

### Group By Module

```jsx
modular
├── beers
│   ├── beer.go
│   └── handler.go
├── main.go
├── reviews
│   ├── handler.go
│   └── review.go
└── storage
    ├── data.go
    ├── json.go
    ├── memory.go
    └── storage.go
```

**优点**

- 按逻辑分组

**缺点**

- 仍然很难确定代码写在哪，比如beer review是 放到 beer包还是rerview包或者是beer_review包
- 命名很糟糕，因为出现了beers/beer 这样的命名

### Group By Context

按照DDD的方式组织

- Context: beer tasting
- Language: beer,review,storage,...
- Entities: Beer,Review,...
- Value Objects: Brewery,Author,...
- Aggregates: BeerReview
- Service: Beer adder/adding, Review adder,Beer lister/listing,Review lister
- Events: Beer added,Review added,Beer  already exists,Beer  not found,...
- Repository: Beer repository,Review repository

```jsx
domain
├── adding
│   ├── endpoint.go
│   └── service.go
├── beers
│   ├── beer.go
│   └── sample_beers.go
├── listing
│   ├── endpoint.go
│   └── service.go
├── main.go
├── reviewing
│   ├── endpoint.go
│   └── service.go
├── reviews
│   ├── review.go
│   └── sample_reviews.go
└── storage
    ├── json.go
    ├── memory.go
    └── type.go
```

**优点**

- 避免了循环依赖 service→stoage→model

**缺点**

- sample data和程序耦合，不能单独运行
- 添加非http的endpoint会很奇怪

### Hexagonal  Architecture

```jsx
domain-hex
├── cmd
│   ├── beer-server
│   │   └── main.go
│   └── sample-data
│       ├── main.go
│       ├── sample-data
│       ├── sample_beers.go
│       └── sample_reviews.go
└── pkg
    ├── adding
    │   ├── beer.go
    │   ├── service.go
    │   └── service_test.go
    ├── http
    │   └── rest
    │       └── handler.go
    ├── listing
    │   ├── beer.go
    │   ├── review.go
    │   └── service.go
    ├── reviewing
    │   ├── review.go
    │   └── service.go
    └── storage
        ├── idgen.go
        ├── idgen_test.go
        ├── json
        │   ├── beer.go
        │   ├── repository.go
        │   └── review.go
        └── memory
            ├── beer.go
            ├── repository.go
            └── review.go
```

优点

- 表达清楚，看目录结构和文件名就知道功能
- 二进制和逻辑分开