---
layout: post
title:  "auxm 注册多个router参数时报错"
date:   2024-11-03 10:29:20 +0800
categories:
      - rust
tags:
      - auxm
      - controller注册多个参数报错
---

#### auxm在controller函数定义多个接收参数时报错

最近手上有个活，打算用点新东西开发。 刚好最近在看rust相关资料。于是心里想：就他了。

因为是web项目，网上一搜索，rust web框架auxm还不错。 打开官网就开始搭建demo。 最开始肯定是先写简单的demo函数，`get/post`路径传参，json传参，form表单传参等等。

以上都很简单，有手就行。 接着就整合数据库，使用了`sqlx`。

根据auxm官网资料， auxm整合sqlx需要先初始化实例，然后注册到路由上下文中，这样就可以动态注入了。

官网的代码一般给的示例都是单独的，比如接收path参数示例，router方法就一个接收path参数。接收json方法，也只定义了一个接收json的参数。我去看了下router注册函数，官方解释是可以接收一个或多个extract。

那就很简单，把所有需要的申明出来就行了

所以大概代码长下面这样（注意addusers函数）：

```rust
pub async  fn init(dburi:&String)  -> Router {
// 初始化数据库连接
    let dbpool:Pool<MySql> = MySqlPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(3))
    .connect(dburi)
    .await
    .expect("can't connect to database");

    let  route = Router::new()
    .route("/", get(|| async { "Hello, World!" }))
    .route(
        "/addusers/:id",
        post(controller::create_user),
    )
    // 注入到router中
    .with_state(dbpool)
    ;
    route
}


pub async fn create_user( 
    Path(id): Path<u32>,
    Json(payload): Json<CreateUserPayload>,
    State(pool): State<Pool<MySql>>
     
 
    ) -> Json<BaseRes>{


    Json(BaseRes { code: 0, msg: payload.name })
}

```

写完代码，立马我被教育了：vscode马上提示错误，router注册`addusers`函数那里代码报错:
```
the trait bound `fn(axum::extract::Path<u32>, axum::Json<CreateUserPayload>, axum::extract::State<Pool<MySql>>) -> impl Future<Output = axum::Json<BaseRes>> {create_user}: Handler<_, _>` is not satisfied
the full name for the type has been written to 'D:\workspace\rust\rust-web-starter\target\debug\deps\rust_web_starter-37beb1a7e3065925.long-type-10239921290377372162.txt'
consider using `--verbose` to print the full type name to the console
Consider using `#[axum::debug_handler]` to improve the error message
the following other types implement trait `Handler<T, S>`:
  `MethodRouter<S>` implements `Handler<(), S>`
  `axum::handler::Layered<L, H, T, S>` implements `Handler<T, S>`rustcClick for full compiler diagnostic
```
我不过就是整合了下，怎么就报错了嘞？

不死心又在官网上看了一圈，依然没找到什么。心里也是懵了，这什么情况？

最后不死心，又找了几个小时，终于官网找到了答案：就是router定义参数，必须严格按照顺序，先解析请求头，再解析上下文请求体，最后解析请求体。所以以上代码定义应该是：

```rust
pub async fn create_user( 
    Path(id): Path<u32>,
    State(pool): State<Pool<MySql>>,
    Json(payload): Json<CreateUserPayload>,

     
 
    ) -> Json<BaseRes>{


    Json(BaseRes { code: 0, msg: payload.name })
}

```
上面代码就不报错了，唉。活见久。

[点击查看官方文档-router函数参数顺序](https://docs.rs/axum/latest/axum/extract/index.html#the-order-of-extractors)
