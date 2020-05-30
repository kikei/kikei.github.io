---
layout: post
title:  "Actix-web type-safe extractionを理解する その2"
categories: rust
---

Actix-web はプログラミング言語 Rust で開発された Web フレームワークである。
近年の Rust の Web フレームワーク界隈では最も開発が盛んだと言われている。

Actix-web には Type-safe information extraction という機能があり、不思議なことに、URL とマッピングされる関数の型がバラバラでも記述でき、 実際、直感通りに動いてしまう。 さらに型どころか、引数の数が違ったり、順番が違っても問題無い。
詳細は前回の記事に書いた。

[Actix\-web type\-safe extractionを理解する その1](https://kikei.github.io/rust/2020/05/23/actix-extractor-1.html)

今回はこの続編で、Actix-webのソースコードを追跡しどのようにして実現しているのか調べた。

### 1. サンプルアプリケーション

まず、実装を追う起点として、下記の単純なアプリケーションを作成した。

```rs
/// main.rs

use actix_web::{web, App, HttpServer};

async fn index(info: web::Path<(u32, String)>) -> impl Responder {
    format!("Welcome {}, (userid: {}) !", &info.1, &info.0)
}

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().route(
            "/users/{userid}/{nickname}", web::get().to(index),
        )
    })
    .bind("127.0.0.1:8088")?
    .workers(1)
    .run()
    .await
}
```

### 2. Actix-web ソースコードの取得

Actix-web のソースコードは公式の GitHub レポジトリより取得できる。

[actix/actix\-web: Actix web is a small, pragmatic, and extremely fast rust web framework\.](https://github.com/actix/actix-web)

以降ではこの中からソースコードを引用、抜粋しながら旅をする。
引用しているのは現時点 2020/5/25 のコードである。
コード中のコメントなどは気分で省略する場合もある。

actix-web は、Actix プロジェクトの他のライブラリにも依存しているが、本稿ではそこまであまり踏み入れない。
ただし actix-http と actix-router の引用も一部行なっている。

### 3. ハンドラの生成

まずは `web::get().to(index)` について。
ある URL で GET リクエストを受信したときに、サンプルアプリケーションの関数 `index` を呼び出す。

web.rs では以下のように実装されている。

```rs
/// actix-web/src/web.rs

pub fn get() -> Route {
    method(Method::GET)
}

pub fn method(method: Method) -> Route {
    Route::new().method(method)
}
```

次に `struct Route` と `fn to` の定義を見る。
`handler` はサンプルの関数 `index` を指す。
特に変わったことはしていないが、`RouteNewService`、`Extract`、`Handler` で包んでから保持している。

```rs
/// actix-web/src/route.rs

pub struct Route {
    service: BoxedRouteNewService<ServiceRequest, ServiceResponse>,
    guards: Rc<Vec<Box<dyn Guard>>>,
}

impl Route {
    pub fn to<F, T, R, U>(mut self, handler: F) -> Self
    where
        F: Factory<T, R, U>,
        T: FromRequest + 'static,
        R: Future<Output = U> + 'static,
        U: Responder + 'static,
    {
        self.service =
            Box::new(RouteNewService::new(Extract::new(Handler::new(handler))));
        self
    }
}
```

`handler` の型は `Factory<T, R, U>` である。
`T` はリクエスト相当の型。
`handler` は非同期関数のため `R` を `Future<Output = U>` とし、
`U` は `Responder` である。
これはサンプルの `index` の戻値 `impl Responder` に対応する。

ここで `handler` はただの関数だったはずだが、`Factory` として受け取っている。
この `Factory` はいったい何なのか。

#### Factory

`Factory` は次のように定義される。

```rs
/// actix-web/src/handler.rs

/// Async handler converter factory
pub trait Factory<T, R, O>: Clone + 'static
where
    R: Future<Output = O>,
    O: Responder,
{
    fn call(&self, param: T) -> R;
}
```

これは `T` を受け取って `R` を返す非同期関数 `call` を実装する型を表現している。
つまり、`factory: &Factory` だとしたら、
`let responder: R = factory.call(param: T).await`
のように実行できる。

この `Factory` がクロージャに対して実装されているため、
サンプルの関数 `index` を `Factory` として扱える。
クロージャに対する `Factory` の実装は以下のように、引数の数ごとに行われている。

引数無しバージョン:

```rs
impl<F, R, O> Factory<(), R, O> for F
where
    F: Fn() -> R + Clone + 'static,
    R: Future<Output = O>,
    O: Responder,
{
    fn call(&self, _: ()) -> R {
        (self)()
    }
}
```

引数1個バージョン:

```rs
impl<Func, A, Res, O> Factory<(A), Res, O> for Func
    where Func: Fn(A) -> Res + Clone + 'static,
          Res: Future<Output = O>,
          O: Responder,
    {
         fn call(&self, param: (A)) -> Res {
             (self)(param.0)
         }
    }
```

引数2個バージョン:

```rs
impl<Func, A, B, Res, O> Factory<(A, B), Res, O> for Func
    where Func: Fn(A, B) -> Res + Clone + 'static,
          Res: Future<Output = O>,
          O: Responder,
    {
         fn call(&self, param: (A, B)) -> Res {
             (self)(param.0, param.1)
         }
    }
});
```

引数3個バージョン:

```rs
impl<Func, A, B, C, Res, O> Factory<(A, B, C), Res, O> for Func
    where Func: Fn(A, B, C) -> Res + Clone + 'static,
          Res: Future<Output = O>,
          O: Responder,
    {
         fn call(&self, param: (A, B, C)) -> Res {
             (self)(param.0, param.1, param.2)
         }
    }
});
```

`call` は引数の数と同じだけの要素数を持つタプルを受け取り、
それを展開してクロージャである `self` に適用する。

実のところ、引数1個以上のものは一つ一つ手書きではなく、
マクロ `factory_tuple` を使って生成されている。

```rs
/// actix-web/src/handler.rs

/// FromRequest trait impl for tuples
macro_rules! factory_tuple ({ $(($n:tt, $T:ident)),+} => {
    impl<Func, $($T,)+ Res, O> Factory<($($T,)+), Res, O> for Func
    where Func: Fn($($T,)+) -> Res + Clone + 'static,
          Res: Future<Output = O>,
          O: Responder,
    {
        fn call(&self, param: ($($T,)+)) -> Res {
            (self)($(param.$n,)+)
        }
    }
});
```

上記のマクロを利用し、クロージャに対する `Factory` の実装を簡単に生成する。

```rs
/// actix-web/src/handler.rs

factory_tuple!((0, A));
factory_tuple!((0, A), (1, B));
factory_tuple!((0, A), (1, B), (2, C));
factory_tuple!((0, A), (1, B), (2, C), (3, D));
factory_tuple!((0, A), (1, B), (2, C), (3, D), (4, E));
factory_tuple!((0, A), (1, B), (2, C), (3, D), (4, E), (5, F));
factory_tuple!((0, A), (1, B), (2, C), (3, D), (4, E), (5, F), (6, G));
factory_tuple!((0, A), (1, B), (2, C), (3, D), (4, E), (5, F), (6, G), (7, H));
factory_tuple!((0, A), (1, B), (2, C), (3, D), (4, E), (5, F), (6, G), (7, H), (8, I));
factory_tuple!((0, A), (1, B), (2, C), (3, D), (4, E), (5, F), (6, G), (7, H), (8, I), (9, J));
```

このようにして、0〜10個の引数を受け取るクロージャを取り扱えるようになった。

このクロージャを呼びだすためには、その引数に相当する型をもったタプルを作り、`call` で渡してあげればよい。

でもそんなタプル、どうやって作り出すの？ということはここから調査していく。

ただその前に、
せっかくなので ` RouteNewService`、`Handler`、`Extract` の定義も見ておこう。

RouteNewService:

```rs
/// actix-web/src/route.rs

struct RouteNewService<T>
where
    T: ServiceFactory<Request = ServiceRequest, Error = (Error, ServiceRequest)>,
{
    service: T,
}
```

Handler:

```rs
/// actix-web/src/handler.rs

pub struct Handler<F, T, R, O>
where
    F: Factory<T, R, O>,
    R: Future<Output = O>,
    O: Responder,
{
    hnd: F,
    _t: PhantomData<(T, R, O)>,
}
```

Extract:

```rs
/// Extract arguments from request
pub struct Extract<T: FromRequest, S> {
    service: S,
    _t: PhantomData<T>,
}
```

特に何もせず、渡されたものをただ構造体に入れているだけのようだ。

`PhantomData` が使われている。`Handler<F, T, R, O>` では `T` `R` `O`、`Extract<T: FromRequest, S>` では `S` が定義上出現しないため、`PhantomData` で消費しているのだろう。

### 4. ルーティングテーブル作成

ここまで `web::get().to` を見てきた。
この関数により、HTTP リクエストを処理する任意の型の関数が `Route` として表現された。

次は `App::new()::route()` について調べる。
この関数には URL と `Route` を渡すことができるが、
繰り返し呼び出してチェーンさせることでルーティングテーブルを作成することができる。

`App` の定義は下記の通り。

```rs
/// actix-web/src/app.rs

pub struct App<T, B> {
    endpoint: T,
    services: Vec<Box<dyn AppServiceFactory>>,
    default: Option<Rc<HttpNewService>>,
    factory_ref: Rc<RefCell<Option<AppRoutingFactory>>>,
    data: Vec<Box<dyn DataFactory>>,
    data_factories: Vec<FnDataFactory>,
    external: Vec<ResourceDef>,
    extensions: Extensions,
    _t: PhantomData<B>,
}

impl<T, B> App<T, B>
where
    B: MessageBody,
    T: ServiceFactory<
        Config = (),
        Request = ServiceRequest,
        Response = ServiceResponse<B>,
        Error = Error,
        InitError = (),
    >,
{
    pub fn route(self, path: &str, mut route: Route) -> Self {
        self.service(
            Resource::new(path)
                .add_guards(route.take_guards())
                .route(route),
        )
    }
    
    pub fn service<F>(mut self, factory: F) -> Self
    where
        F: HttpServiceFactory + 'static,
    {
        self.services
            .push(Box::new(ServiceFactoryWrapper::new(factory)));
        self
    }
}
```

`Vec` である `App.services` にテーブルが登録された。

テーブルの各行、URL とハンドラは `Resource` によって対応付けされる。
これは `HttpServiceFactory` を実装し、`ServiceFactoryWrapper` で包まれている。



### 5. HTTPサーバー生成

構築したルーティングテーブルを含んだ `App` 構造体は、
`|| { App::new().route(...) }` というように、
この構造体を返すファクトリ関数の形で `HttpServer::new()` に渡す。

最終的にはこのルーティングテーブルを使った HTTP サーバーを起動するのだが、ここは長い旅になる。
しかし長旅にも関わらず Extraction の理解には特に関係無いため、この項は読み飛ばして問題無い。

実装の中で、`Service` と `ServiceFactory` が何度も登場する。
`Service` は `Request` を受け取って `Response` を返す非同期処理として抽象化されている。
Actix では、単にHTTP リクエスト/レスポンスの関係だけでなく、サーバーの生成など多くの処理が `Service` で表現され、
それらを繋ぎ合わせてシステム全体が構成される。
ミニマルな光景が延々と続くため、ソースコードから全体的な処理を追うのがやや難しくなっている。

[actix\_service::Service \- Rust](https://docs.rs/actix-service/0.4.2/actix_service/trait.Service.html)

それでは、まず `HttpServer` の定義は以下:

```rs
/// actix-web/src/server.rs

impl<F, I, S, B> HttpServer<F, I, S, B>
where
    F: Fn() -> I + Send + Clone + 'static,
    I: IntoServiceFactory<S>,
    S: ServiceFactory<Config = AppConfig, Request = Request>,
    S::Error: Into<Error> + 'static,
    S::InitError: fmt::Debug,
    S::Response: Into<Response<B>> + 'static,
    <S::Service as Service>::Future: 'static,
    B: MessageBody + 'static,
{
    /// Create new http server with application factory
    pub fn new(factory: F) -> Self {
        HttpServer {
            factory,
            config: Arc::new(Mutex::new(Config {
                host: None,
                keep_alive: KeepAlive::Timeout(5),
                client_timeout: 5000,
                client_shutdown: 5000,
            })),
            backlog: 1024,
            sockets: Vec::new(),
            builder: ServerBuilder::default(),
            _t: PhantomData,
        }
    }
}
```

`HttpServer::run` を実行すると、HTTP サーバーの初期化処理が実行され、
`HttpServer::listen` に辿り着く。
その中で `HttpService` を生成するときに
先のファクトリ関数 `factory` を実行し `App` を取り出す。

```rs
/// actix-web/src/server.rs

    /// Use listener for accepting incoming connection requests
    ///
    /// HttpServer does not change any configuration for TcpListener,
    /// it needs to be configured before passing it to listen() method.
    pub fn listen(mut self, lst: net::TcpListener) -> io::Result<Self> {
        let cfg = self.config.clone();
        let factory = self.factory.clone();
        let addr = lst.local_addr().unwrap();
        self.sockets.push(Socket {
            addr,
            scheme: "http",
        });

        self.builder = self.builder.listen(
            format!("actix-web-service-{}", addr),
            lst,
            move || {
                let c = cfg.lock().unwrap();
                let cfg = AppConfig::new(
                    false,
                    addr,
                    c.host.clone().unwrap_or_else(|| format!("{}", addr)),
                );

                HttpService::build()
                    .keep_alive(c.keep_alive)
                    .client_timeout(c.client_timeout)
                    .local_addr(addr)
                    .finish(map_config(factory(), move |_| cfg.clone()))
                    .tcp()
            },
        )?;
        Ok(self)
    }
}
```

さらに `HttpService::new` の途中で
`App` から `into_factory` を使い `AppInit` に変換する。

```rs
/// actix-http/src/service.rs

impl<T, S, B> HttpService<T, S, B>
where
    S: ServiceFactory<Config = (), Request = Request>,
    S::Error: Into<Error> + 'static,
    S::InitError: fmt::Debug,
    S::Response: Into<Response<B>> + 'static,
    <S::Service as Service>::Future: 'static,
    B: MessageBody + 'static,
{
    /// Create new `HttpService` instance.
    pub fn new<F: IntoServiceFactory<S>>(service: F) -> Self {
        let cfg = ServiceConfig::new(KeepAlive::Timeout(5), 5000, 0, false, None);

        HttpService {
            cfg,
            srv: service.into_factory(),
            expect: h1::ExpectHandler,
            upgrade: None,
            on_connect: None,
            _t: PhantomData,
        }
    }
}
```

`App` の `into_factory` 実装はこう。

```rs
/// actix-web/src/app.rs

impl<T, B> IntoServiceFactory<AppInit<T, B>> for App<T, B>
where
    B: MessageBody,
    T: ServiceFactory<
        Config = (),
        Request = ServiceRequest,
        Response = ServiceResponse<B>,
        Error = Error,
        InitError = (),
    >,
{
    fn into_factory(self) -> AppInit<T, B> {
        AppInit {
            data: Rc::new(self.data),
            data_factories: Rc::new(self.data_factories),
            endpoint: self.endpoint,
            services: Rc::new(RefCell::new(self.services)),
            external: RefCell::new(self.external),
            default: self.default,
            factory_ref: self.factory_ref,
            extensions: RefCell::new(Some(self.extensions)),
        }
    }
}
```

`AppInit` は `ServiceFactory` を実装している。
これは actix_services の Trait である。

次にアプリケーションを初期化し `AppInitResult` を返す。

```rs
/// src/lib.rs

impl<T, B> ServiceFactory for AppInit<T, B>
where
    T: ServiceFactory<
        Config = (),
        Request = ServiceRequest,
        Response = ServiceResponse<B>,
        Error = Error,
        InitError = (),
    >,
{
    type Config = AppConfig;
    type Request = Request;
    type Response = ServiceResponse<B>;
    type Error = T::Error;
    type InitError = T::InitError;
    type Service = AppInitService<T::Service, B>;
    type Future = AppInitResult<T, B>;

    fn new_service(&self, config: AppConfig) -> Self::Future {
        // update resource default service
        let default = self.default.clone().unwrap_or_else(|| {
            Rc::new(boxed::factory(fn_service(|req: ServiceRequest| {
                ok(req.into_response(Response::NotFound().finish()))
            })))
        });

        // App config
        let mut config = AppService::new(config, default.clone(), self.data.clone());

        // register services
        std::mem::take(&mut *self.services.borrow_mut())
            .into_iter()
            .for_each(|mut srv| srv.register(&mut config));

        let mut rmap = ResourceMap::new(ResourceDef::new(""));

        let (config, services) = config.into_services();

        // complete pipeline creation
        *self.factory_ref.borrow_mut() = Some(AppRoutingFactory {
            default,
            services: Rc::new(
                services
                    .into_iter()
                    .map(|(mut rdef, srv, guards, nested)| {
                        rmap.add(&mut rdef, nested);
                        (rdef, srv, RefCell::new(guards))
                    })
                    .collect(),
            ),
        });

        // external resources
        for mut rdef in std::mem::take(&mut *self.external.borrow_mut()) {
            rmap.add(&mut rdef, None);
        }

        // complete ResourceMap tree creation
        let rmap = Rc::new(rmap);
        rmap.finish(rmap.clone());

        // start all data factory futures
        let factory_futs = join_all(self.data_factories.iter().map(|f| f()));

        AppInitResult {
            endpoint: None,
            endpoint_fut: self.endpoint.new_service(()),
            data: self.data.clone(),
            data_factories: None,
            data_factories_fut: factory_futs.boxed_local(),
            extensions: Some(
                self.extensions
                    .borrow_mut()
                    .take()
                    .unwrap_or_else(Extensions::new),
            ),
            config,
            rmap,
            _t: PhantomData,
        }
    }
}
```

`AppInitResult` から非同期に `AppInitService` を生成。

```rs
/// actix-web/src/app_service.rs

impl<T, B> Future for AppInitResult<T, B>
where
    T: ServiceFactory<
        Config = (),
        Request = ServiceRequest,
        Response = ServiceResponse<B>,
        Error = Error,
        InitError = (),
    >,
{
    type Output = Result<AppInitService<T::Service, B>, ()>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();

        // async data factories
        if let Poll::Ready(factories) = this.data_factories_fut.poll(cx) {
            let factories: Result<Vec<_>, ()> = factories.into_iter().collect();

            if let Ok(factories) = factories {
                this.data_factories.replace(factories);
            } else {
                return Poll::Ready(Err(()));
            }
        }

        // app service and middleware
        if this.endpoint.is_none() {
            if let Poll::Ready(srv) = this.endpoint_fut.poll(cx)? {
                *this.endpoint = Some(srv);
            }
        }

        // not using if let so condition only needs shared ref
        if this.endpoint.is_some() && this.data_factories.is_some() {
            // create app data container
            let mut data = this.extensions.take().unwrap();

            for f in this.data.iter() {
                f.create(&mut data);
            }

            for f in this.data_factories.take().unwrap().iter() {
                f.create(&mut data);
            }

            return Poll::Ready(Ok(AppInitService {
                service: this.endpoint.take().unwrap(),
                rmap: this.rmap.clone(),
                config: this.config.clone(),
                data: Rc::new(data),
                pool: HttpRequestPool::create(),
            }));
        }

        Poll::Pending
    }
}
```

この `AppInitService` を以って HTTP サーバーアプリケーションが起動された。


### 6. HTTP リクエストの受信

ようやくリクエストを取得していそうなところまで着いた。

```rs
/// actix-web/src/app_service.rs

impl<T, B> Service for AppInitService<T, B>
where
    T: Service<Request = ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
{
    type Request = Request;
    type Response = ServiceResponse<B>;
    type Error = T::Error;
    type Future = T::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.service.poll_ready(cx)
    }

    fn call(&mut self, req: Request) -> Self::Future {
        let (head, payload) = req.into_parts();

        let req = if let Some(mut req) = self.pool.get_request() {
            let inner = Rc::get_mut(&mut req.0).unwrap();
            inner.path.get_mut().update(&head.uri);
            inner.path.reset();
            inner.head = head;
            inner.payload = payload;
            inner.app_data.push(self.data.clone());
            req
        } else {
            HttpRequest::new(
                Path::new(Url::new(head.uri.clone())),
                head,
                payload,
                self.rmap.clone(),
                self.config.clone(),
                self.data.clone(),
                self.pool,
            )
        };
        self.service.call(ServiceRequest::new(req))
    }
}
```

ここで `HttpRequest` は URL 情報、ヘッダ、ボディ、アプリケーションデータ、コネクションプールの全てを持つ。
アプリケーションデータ、コネクションプールは、アプリケーションのトップレベルで `App` の生成時に指定可能な、アプリケーション内の共通データである。

```rs
pub struct HttpRequest(pub(crate) Rc<HttpRequestInner>);

pub(crate) struct HttpRequestInner {
    pub(crate) head: Message<RequestHead>,
    pub(crate) path: Path<Url>,
    pub(crate) payload: Payload,
    pub(crate) app_data: TinyVec<[Rc<Extensions>; 4]>,
    rmap: Rc<ResourceMap>,
    config: AppConfig,
    pool: &'static HttpRequestPool,
}
```

`HttpRequest` は `ServiceRequest` で包み、`add_data_container` を使えるようにしている。

```rs
/// service.rs
pub struct ServiceRequest(HttpRequest);

impl ServiceRequest {
    /// Add app data container to request's resolution set.
    pub fn add_data_container(&mut self, extensions: Rc<Extensions>) {
        Rc::get_mut(&mut (self.0).0)
            .unwrap()
            .app_data
            .push(extensions);
    }
}
```

少し端折って、この `ServiceRequest` は `AppRouting` に渡される。

### 7. ルーティング

受信した HTTP リクエストに対応するリソースが設定されているか、
ルーティングテーブルを調べる処理は `AppRouting` から呼び出される。

下記コード中の `self.router.recognize_mut_checked(&mut req, |req, guards| {})` が
それ。

`recognize_mut_checked` の第2引数は `Guard` で、単に URL が条件を満すだけでなく、
HTTP リクエストメソッドが一致すること、
アプリケーションのトップレベルで設定した `guard` が満たされることを調べている。

ここで全ての条件を満たすリソースが見つかれば、
`srv.call(req)` が、無ければ `default.call(req)` に入っていく。

```rs
/// actix-web/src/app_service.rs

type Guards = Vec<Box<dyn Guard>>;
type HttpService = BoxService<ServiceRequest, ServiceResponse, Error>;

pub struct AppRouting {
    router: Router<HttpService, Guards>,
    ready: Option<(ServiceRequest, ResourceInfo)>,
    default: Option<HttpService>,
}

impl Service for AppRouting {
    type Request = ServiceRequest;
    type Response = ServiceResponse;
    type Error = Error;
    type Future = BoxResponse;

    fn call(&mut self, mut req: ServiceRequest) -> Self::Future {
        let res = self.router.recognize_mut_checked(&mut req, |req, guards| {
            if let Some(ref guards) = guards {
                for f in guards {
                    if !f.check(req.head()) {
                        return false;
                    }
                }
            }
            true
        });

        if let Some((srv, _info)) = res {
            srv.call(req)
        } else if let Some(ref mut default) = self.default {
            default.call(req)
        } else {
            let req = req.into_parts().0;
            ok(ServiceResponse::new(req, Response::NotFound().finish())).boxed_local()
        }
    }
}
```

`recognize_mut_checked` は以下のようになっている。
ここでルーティングテーブルの各行について順番に、`match_path_checked` を呼び出し、初めに見つかったものを返却する。


```rs
/// actix-router/src/router.rs

pub struct Router<T, U = ()>(Vec<(ResourceDef, T, Option<U>)>);

impl<T, U> Router<T, U> {
    pub fn recognize_mut_checked<R, P, F>(
        &mut self,
        resource: &mut R,
        check: F,
    ) -> Option<(&mut T, ResourceId)>
    where
        F: Fn(&R, &Option<U>) -> bool,
        R: Resource<P>,
        P: ResourcePath,
    {
        for item in self.0.iter_mut() {
            if item.0.match_path_checked(resource, &check, &item.2) {
                return Some((&mut item.1, ResourceId(item.0.id())));
            }
        }
        None
    }
}
```

`match_path_checked` は個別の URL のマッチと `check` クロージャの呼び出しを行う。
下記では省略したが各 URL パターンに対する比較処理の最後に `check` 実行をする。

`checked` でないバージョンもあるが、その場合は `check` の実行が無い。

```src
/// actix-router/src/resource.rs

impl ResourceDef {
    /// Is the given path and parameters a match against this pattern?
    pub fn match_path_checked<R, T, F, U>(
        &self,
        res: &mut R,
        check: &F,
        user_data: &Option<U>,
    ) -> bool
    where
        T: ResourcePath,
        R: Resource<T>,
        F: Fn(&R, &Option<U>) -> bool,
    {
        match self.tp {
            PatternType::Static(ref s) => {/*...*/}
            PatternType::Prefix(ref s) => {/*...*/}
            PatternType::Dynamic(ref re, ref names, len) => {/*...*/}
            PatternType::DynamicSet(ref re, ref params) => {/*...*/}
        }
    }
}
```

### 8. ハンドラ抽出

ここからは `call` を連続しながら元の階層まで戻っていく。

```rs
/// actix-web/src/resource.rs

impl Service for ResourceService {
    type Request = ServiceRequest;
    type Response = ServiceResponse;
    type Error = Error;
    type Future = Either<
        Ready<Result<ServiceResponse, Error>>,
        LocalBoxFuture<'static, Result<ServiceResponse, Error>>,
    >;
    fn call(&mut self, mut req: ServiceRequest) -> Self::Future {
        for route in self.routes.iter_mut() {
            if route.check(&mut req) {
                if let Some(ref data) = self.data {
                    req.add_data_container(data.clone());
                }
                return Either::Right(route.call(req));
            }
        }
        if let Some(ref mut default) = self.default {
            if let Some(ref data) = self.data {
                req.add_data_container(data.clone());
            }
            Either::Right(default.call(req))
        } else {
            let req = req.into_parts().0;
            Either::Left(ok(ServiceResponse::new(
                req,
                Response::MethodNotAllowed().finish(),
            )))
        }
    }
}
```

こうして、

```rs
/// actix-web/src/route.ts

impl Service for RouteService {
    type Request = ServiceRequest;
    type Response = ServiceResponse;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        println!("route::RouteService::poll_ready");
        self.service.poll_ready(cx)
    }

    fn call(&mut self, req: ServiceRequest) -> Self::Future {
        println!("route::RouteService::call");
        self.service.call(req).boxed_local()
    }
}
```

こう…

```rs
/// actix-web/src/resource.rs

impl Service for ResourceService {
    fn call(&mut self, mut req: ServiceRequest) -> Self::Future {
        for route in self.routes.iter_mut() {
            if route.check(&mut req) {
                if let Some(ref data) = self.data {
                    req.add_data_container(data.clone());
                }
                return Either::Right(route.call(req));
            }
        }
    }
}
```

そしてこう…

```rs
/// actix-web/src/route.rs

impl Service for RouteService {
    type Request = ServiceRequest;
    type Response = ServiceResponse;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.service.poll_ready(cx)
    }

    fn call(&mut self, req: ServiceRequest) -> Self::Future {
        self.service.call(req).boxed_local()
    }
}
```

さらにこう…。

```rs
/// actix-web/src/handler.rs

impl<T: FromRequest, S> Service for ExtractService<T, S>
where
    S: Service<
            Request = (T, HttpRequest),
            Response = ServiceResponse,
            Error = Infallible,
        > + Clone,
{
    type Request = ServiceRequest;
    type Response = ServiceResponse;
    type Error = (Error, ServiceRequest);
    type Future = ExtractResponse<T, S>;

    fn poll_ready(&mut self, _: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: ServiceRequest) -> Self::Future {
        println!("handler::ExtractService::call");
        let (req, mut payload) = req.into_parts();
        let fut = T::from_request(&req, &mut payload);

        ExtractResponse {
            fut,
            req,
            fut_s: None,
            service: self.service.clone(),
        }
    }
}
```

上のコードで `T::from_request(&req, &mut payload)` という操作が出てきた。
これが探していた HTTP リクエストからタプル (を返す `Future`) を作り出す処理である。
 
### 9. ハンドラ実行

ハンドラの実行は

1. HTTP リクエストからハンドラの引数を含むタプルを返す `Future` を生成し、
2. `Future` からタブルを取り出し、
3. ハンドラにタプルを渡す。

#### FromRequest

まず 1 については、引数の数ごとに実装されている。
引数の数と同じ要素数を持つタプルについて、その要素数ごとに `FromRequest` を実装する。
`fn from_request` は `Ready<Result</* タプル */, Error>>` を返却する。

引数が無い場合:

```rs
/// web-actix/src/extract.rs

impl FromRequest for () {
    type Config = ();
    type Error = Error;
    type Future = Ready<Result<(), Error>>;

    fn from_request(_: &HttpRequest, _: &mut Payload) -> Self::Future {
        ok(())
    }
}
```

引数が1つの場合:

```rs
pub struct TupleFromRequest1<A: FromRequest> {
    items: (Option<A>),
    #[pin]
    futs: FutWrapper<A>
}

impl<A: FromRequest + 'static> FromRequest for (A)
{
    type Error = Error;
    type Future = TupleFromRequest1<A>;
    type Config = (A::Config);

    fn from_request(req: &HttpRequest, payload: &mut Payload) -> Self::Future {
        TupleFromRequest1 {
            items: <(A)>::default(),
            futs: FutWrapper(A::from_request(req, payload)),
        }
    }
}
```

引数が2つの場合:

```rs
pub struct TupleFromRequest2<A: FromRequest, B: FromRequest> {
    items: (Option<A>, Option<B>),
    #[pin]
    futs: FutWrapper<A, B>
}

impl<A: FromRequest + 'static,
     B: FromRequest + 'static'> FromRequest for (A, B)
{
    type Error = Error;
    type Future = TupleFromRequest2<A, B>;
    type Config = (A::Config, B::Config);

    fn from_request(req: &HttpRequest, payload: &mut Payload) -> Self::Future {
        TupleFromRequest2 {
            items: <(A, B)>::default(),
            futs: FutWrapper(A::from_request(req, payload),
                             B::from_request(req, payload)),
        }
    }
}
```

タプルの各要素 (ハンドラの引数) に使えるのは
`Path` とか `Query`、`Json`、`Payload` とかで、全て `FromRequest` を実装しているため、
それぞれが `HttpRequest` から生成される。

上記の実装ではジェネリクスを利用して `A` や `B` 等が `Path` とか `Query` とかになっているので、
`A::from_request`、`B::from_request` というようにして引数を生成できる。

#### Future

上記で定義した `TupleFromRequest1` や `TupleFromRequest2` を返却する `Future` から値を取り出す `poll` 関数を実装する。
`struct` ごとに異なる引数の数の実装になる。

引数が1つの場合:

```rs
impl<A> Future for TupleFromRequest1<A>
{
    type Output = Result<(A), Error>;
    
    fn poll(self: Pin<&mut Selt> cs: &mut Context<'_>) -> Poll<Self::Output> {
        let mut this = self.project();
        
        let mut ready = true;
        if this.items.0.is_none() {
            match this.futs.as_mut().project().0.poll(cs) {
                Poll::Ready(Ok(item)) => {
                    this.items.0 = Some(item);
                }
                Poll::Pending => ready + false,
                Poll::Ready(Err(e)) => return Poll:Ready(Err(e.into()))
            }
        }
        
        if ready {
            Poll::Ready(Ok(
                (this.items.0.take().unwrap())
            ))
        } else {
            Poll.Pending
        }
    }
}
```

引数が2つの場合:

```rs
impl<A, B> Future for TupleFromRequest2<A, B>
{
    type Output = Result<(A, B), Error>;
    
    fn poll(self: Pin<&mut Selt> cs: &mut Context<'_>) -> Poll<Self::Output> {
        let mut this = self.project();
        
        let mut ready = true;
        if this.items.0.is_none() {
            match this.futs.as_mut().project().0.poll(cs) {
                Poll::Ready(Ok(item)) => {
                    this.items.0 = Some(item);
                }
                Poll::Pending => ready + false,
                Poll::Ready(Err(e)) => return Poll:Ready(Err(e.into()))
            }
        }
        if this.items.1.is_none() {
            match this.futs.as_mut().project().1.poll(cs) {
                Poll::Ready(Ok(item)) => {
                    this.items.1 = Some(item);
                }
                Poll::Pending => ready + false,
                Poll::Ready(Err(e)) => return Poll:Ready(Err(e.into()))
            }
        }
        
        if ready {
            Poll::Ready(Ok(
                (this.items.0.take().unwrap(),
                 this.items.1.take().unwrap())
            ))
        } else {
            Poll.Pending
        }
    }
}
```

タプルの要素が `Ready` になったら、それぞれの生の値を格納したタプルを `Ready` で返却するようになっている。

すでにお気づきだと思うが、これらはマクロ (`tuple_from_req`)で記述されている。

```rs
/// actix-web/src/extract.rs
macro_rules! tuple_from_req ({$fut_type:ident, $(($n:tt, $T:ident)),+} => {

    // This module is a trick to get around the inability of
    // `macro_rules!` macros to make new idents. We want to make
    // a new `FutWrapper` struct for each distinct invocation of
    // this macro. Ideally, we would name it something like
    // `FutWrapper_$fut_type`, but this can't be done in a macro_rules
    // macro.
    //
    // Instead, we put everything in a module named `$fut_type`, thus allowing
    // us to use the name `FutWrapper` without worrying about conflicts.
    // This macro only exists to generate trait impls for tuples - these
    // are inherently global, so users don't have to care about this
    // weird trick.
    #[allow(non_snake_case)]
    mod $fut_type {

        // Bring everything into scope, so we don't need
        // redundant imports
        use super::*;

        /// A helper struct to allow us to pin-project through
        /// to individual fields
        #[pin_project::pin_project]
        struct FutWrapper<$($T: FromRequest),+>($(#[pin] $T::Future),+);

        /// FromRequest implementation for tuple
        #[doc(hidden)]
        #[allow(unused_parens)]
        impl<$($T: FromRequest + 'static),+> FromRequest for ($($T,)+)
        {
            type Error = Error;
            type Future = $fut_type<$($T),+>;
            type Config = ($($T::Config),+);

            fn from_request(req: &HttpRequest, payload: &mut Payload) -> Self::Future {
                $fut_type {
                    items: <($(Option<$T>,)+)>::default(),
                    futs: FutWrapper($($T::from_request(req, payload),)+),
                }
            }
        }

        #[doc(hidden)]
       #[pin_project::pin_project]
        pub struct $fut_type<$($T: FromRequest),+> {
            items: ($(Option<$T>,)+),
            #[pin]
            futs: FutWrapper<$($T,)+>,
        }

        impl<$($T: FromRequest),+> Future for $fut_type<$($T),+>
        {
            type Output = Result<($($T,)+), Error>;

            fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
                let mut this = self.project();

                let mut ready = true;
                $(
                    if this.items.$n.is_none() {
                        match this.futs.as_mut().project().$n.poll(cx) {
                            Poll::Ready(Ok(item)) => {
                                this.items.$n = Some(item);
                            }
                            Poll::Pending => ready = false,
                            Poll::Ready(Err(e)) => return Poll::Ready(Err(e.into())),
                        }
                    }
                )+

                    if ready {
                        Poll::Ready(Ok(
                            ($(this.items.$n.take().unwrap(),)+)
                        ))
                    } else {
                        Poll::Pending
                    }
            }
        }
    }
});
```

以下のようにして、0〜10個の引数に対応している。

```rs
tuple_from_req!(TupleFromRequest1, (0, A));
tuple_from_req!(TupleFromRequest2, (0, A), (1, B));
tuple_from_req!(TupleFromRequest3, (0, A), (1, B), (2, C));
tuple_from_req!(TupleFromRequest4, (0, A), (1, B), (2, C), (3, D));
tuple_from_req!(TupleFromRequest5, (0, A), (1, B), (2, C), (3, D), (4, E));
tuple_from_req!(TupleFromRequest6, (0, A), (1, B), (2, C), (3, D), (4, E), (5, F));
tuple_from_req!(TupleFromRequest7, (0, A), (1, B), (2, C), (3, D), (4, E), (5, F), (6, G));
tuple_from_req!(TupleFromRequest8, (0, A), (1, B), (2, C), (3, D), (4, E), (5, F), (6, G), (7, H));
tuple_from_req!(TupleFromRequest9, (0, A), (1, B), (2, C), (3, D), (4, E), (5, F), (6, G), (7, H), (8, I));
tuple_from_req!(TupleFromRequest10, (0, A), (1, B), (2, C), (3, D), (4, E), (5, F), (6, G), (7, H), (8, I), (9, J));
```

#### Factory

3の呼び出しについては、序盤で明らかにした通り、
0〜10個の引数を受け取るクロージャに対して `Factory` を実装することで実装している。
`fn call`にタプルを渡すと、それを引数に展開してクロージャを実行する。

#### ExtractService

最後に、ハンドラが呼び出される部分のコードを眺めておく。
まず前項の続きで `ExtractResponse` に入る。

`this.fut.poll(cx)` でタプルを返す `Future` が得られる。
それから取り出したタプルを使い、`this.service.call((item, this.req.clone()))` にて `Handler` を呼び出す。


```rs
/// web-actix/src/handler.rs

impl<T: FromRequest, S> Future for ExtractResponse<T, S>
where
    S: Service<
        Request = (T, HttpRequest),
        Response = ServiceResponse,
        Error = Infallible,
    >,
{
    type Output = Result<ServiceResponse, (Error, ServiceRequest)>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.as_mut().project();

        if let Some(fut) = this.fut_s.as_pin_mut() {
            return fut.poll(cx).map_err(|_| panic!());
        }

        match ready!(this.fut.poll(cx)) {
            Err(e) => {
                let req = ServiceRequest::new(this.req.clone());
                Poll::Ready(Err((e.into(), req)))
            }
            Ok(item) => {
                let fut = Some(this.service.call((item, this.req.clone())));
                self.as_mut().project().fut_s.set(fut);
                self.poll(cx)
            }
        }
    }
}
```

`Handler` のコードは以下。

```rs
/// handlers.rs

pub struct Handler<F, T, R, O>
where
    F: Factory<T, R, O>,
    R: Future<Output = O>,
    O: Responder,
{
    hnd: F,
    _t: PhantomData<(T, R, O)>,
}

impl<F, T, R, O> Service for Handler<F, T, R, O>
where
    F: Factory<T, R, O>,
    R: Future<Output = O>,
    O: Responder,
{
    type Request = (T, HttpRequest);
    type Response = ServiceResponse;
    type Error = Infallible;
    type Future = HandlerServiceResponse<R, O>;

    fn call(&mut self, (param, req): (T, HttpRequest)) -> Self::Future {
        HandlerServiceResponse {
            fut: self.hnd.call(param),
            fut2: None,
            req: Some(req),
        }
    }
}
```

このジェネリクス型 `T` の変数 `param` が引数を格納したタプルで、
`self.hnd` がクロージャを抽象化した `Factory` である。

### 10. まとめ

Actix-web の Typesafe extraction では可変長引数、可変型の関数を渡すことができるが、これはジェネリクスと trait を使って実現されている。
ハンドラの引数の型は完全に自由なわけではなく、`FromRequest` を実装している必要があり、何でもありというわけでは無い。

