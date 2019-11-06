<div align="center">

  <img src="https://github.com/yewstack/yew/blob/master/.static/yew.svg" width="150" />

  <h1>
    Yew &nbsp;
    <a href="https://crates.io/crates/yew"><img alt="Build Status" src="https://img.shields.io/crates/v/yew.svg"/></a>
  </h1>

  <p>
    <strong>Rust / Wasm UI framework</strong>
  </p>

  <p>
    <a href="https://travis-ci.com/yewstack/yew"><img alt="Build Status" src="https://travis-ci.com/yewstack/yew.svg?branch=master"/></a>
    <a href="https://gitter.im/yewframework/Lobby"><img alt="Gitter Chat" src="https://badges.gitter.im/yewframework.svg"/></a>
    <a href="https://blog.rust-lang.org/2019/05/23/Rust-1.35.0.html"><img alt="Rustc Version 1.35+" src="https://img.shields.io/badge/rustc-1.35+-lightgray.svg"/></a>
  </p>

  <h4>
    <a href="#running-the-examples">Examples</a>
    <span> | </span>
    <a href="https://github.com/ryugou/yew/blob/master/CHANGELOG.md">Changelog</a>
    <span> | </span>
    <a href="https://github.com/ryugou/yew/blob/master/CODE_OF_CONDUCT.md">Code of Conduct</a>
  </h4>
</div>

## 概要

**Yew**は Elm と React に触発された最新の Rustフレームワークです。
WebAssembly を使用してマルチスレッドフロントエンドアプリを作成します。

フレームワークは***マルチスレッドと同時実行***をすぐに使えるようサポートします。
[Web Workers API]を使用して、別々のスレッドでアクター（エージェント）を生成します。
また、並行タスクのスレッドに接続されたローカルスケジューラを使用します。

[Web Workers API]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API

[Become a sponsor on Patreon](https://www.patreon.com/deniskolodin)

[Check out a live demo](https://yew-todomvc.netlify.com/) powered by [`yew-wasm-pack-template`](https://github.com/yewstack/yew-wasm-pack-template)

## 最先端のテクノロジー

### Rust to WASM compilation

このフレームワークは、最新のブラウザーのランタイム（wasm、asm.js、emscripten）にコンパイルされるように設計されています。
開発環境を準備するには、次のインストール手順を使用してください([wasm-and-rust](https://github.com/raphamorim/wasm-and-rust))。

### ElmとReduxに触発されたアーキテクチャ

Yewはメッセージの受け渡しと更新に基づいて、厳密なアプリケーション状態管理を実装しています。

`src/main.rs`

```rust
use yew::{html, Component, ComponentLink, Html, ShouldRender};

struct Model { }

enum Msg {
    DoIt,
}

impl Component for Model {
    // 一部の詳細は省略されています。詳細を確認するには examples をご覧ください。

    type Message = Msg;
    type Properties = ();

    fn create(_: Self::Properties, _: ComponentLink<Self>) -> Self {
        Model { }
    }

    fn update(&mut self, msg: Self::Message) -> ShouldRender {
        match msg {
            Msg::DoIt => {
                // イベントでModelを更新する
                true
            }
        }
    }

    fn view(&self) -> Html<Self> {
        html! {
            // ここでModelをレンダリングします
            <button onclick=|_| Msg::DoIt>{ "Click me!" }</button>
        }
    }
}

fn main() {
    yew::start_app::<Model>();
}
```

予測可能な可変性とライフタイムにより、更新のたびに新しいインスタンスを作成することなく、モデルの単一インスタンスを再利用できます。また、メモリ割り当ての削減にも役立ちます。


### `html!`マクロを使用したJSXライクなテンプレート

すべてのコンパイラを使用して純粋なRustコードをHTMLタグに入れて、チェッカーの利点を借りてください。

```rust
html! {
    <section class="todoapp">
        <header class="header">
            <h1>{ "todos" }</h1>
            { view_input(&model) }
        </header>
        <section class="main">
            <input class="toggle-all"
                   type="checkbox"
                   checked=model.is_all_completed()
                   onclick=|_| Msg::ToggleAll />
            { view_entries(&model) }
        </section>
    </section>
}
```

### エージェント-ErlangとActixに触発されたアクタModel

すべての「コンポーネント」はエージェントを生成し、それに接続できます。
エージェントは、グローバル状態を調整し、長時間実行されるタスクを生成し、タスクをWebワーカーにオフロードできます。
コンポーネントとは独立して実行されますが、更新メカニズムにうまくフックします。

```rust
use yew::worker::*;

struct Worker {
    link: AgentLink<Worker>,
}

#[derive(Serialize, Deserialize, Debug)]
pub enum Request {
    Question(String),
}

#[derive(Serialize, Deserialize, Debug)]
pub enum Response {
    Answer(String),
}

impl Agent for Worker {
    // Available:
    // - `Job` (メインスレッドのブリッジごとに1つ)
    // - `Context` (メインスレッドで共有)
    // - `Private` (別のスレッドのブリッジごとに1つ)
    // - `Public` (別のスレッドで共有)
    type Reach = Context; // メインスレッドでインスタンスを1つだけ生成します（すべてのコンポーネントがこのエージェントを共有できます）
    type Message = Msg;
    type Input = Request;
    type Output = Response;

    // エージェントの環境へのリンクを持つインスタンスを作成します。
    fn create(link: AgentLink<Self>) -> Self {
        Worker { link }
    }

    // (`send_back`コールバックのサービスの）内部メッセージを処理する

    fn update(&mut self, msg: Self::Message) { /* ... */ }

    // 他のエージェントのコンポーネントからの着信メッセージを処理します。
    fn handle(&mut self, msg: Self::Input, who: HandlerId) {
        match msg {
            Request::Question(_) => {
                self.link.response(who, Response::Answer("That's cool!".into()));
            },
        }
    }
}
```

このエージェントのインスタンスへのブリッジを構築します。
エージェントのタイプに応じて、ワーカーを自動的に生成するか、既存のワーカーを再利用します。

```rust
struct Model {
    context: Box<Bridge<context::Worker>>,
}

enum Msg {
    ContextMsg(context::Response),
}

impl Component for Model {
    type Message = Msg;
    type Properties = ();

    fn create(_: Self::Properties, link: ComponentLink<Self>) -> Self {
        let callback = link.send_back(|_| Msg::ContextMsg);
        // `Worker::bridge`は、誰も利用できない場合にインスタンスを生成します
        let context = context::Worker::bridge(callback); // 接続済み! :tada:
        Model { context }
    }
}
```

必要な数のエージェントを使用できます。たとえば、サーバーとのすべての対話を個別のスレッド（Webワーカーがネイティブスレッドにマップするため、実際のOSスレッド）に分離できます。

> **REMEMBER**すべてのAPIがすべての環境で利用できるわけではありません。たとえば、別のスレッドから`StorageService`を使用することはできません。`Public`または`Private`エージェントでは動作せず、`Job`および`Context`エージェントでのみ動作します。

### コンポーネント

Yewはコンポーネントをサポートしています。`Component`トレイトを実装し、それを`html!`テンプレートに直接含めることで、新しいものを作成できます。

```rust
html! {
    <nav class="menu">
        <MyButton title="First Button" />
        <MyButton title="Second Button "/>
        <MyList name="Grocery List">
          <MyListItem text="Apples" />
        </MyList>
    </nav>
}
```

### スコープ

コンポーネントは、**parent-to-child** *(properties)*および**child-to-parent** *(events)*の相互作用を持つ、Angularのようなスコープに存在します。

プロパティは、コンパイル中に厳密な型チェックを行う純粋なRust型でもあります。

```rust
// my_button.rs

#[derive(Properties, PartialEq)]
pub struct Properties {
    pub hidden: bool,
    #[props(required)]
    pub color: Color,
    #[props(required)]
    pub onclick: Callback<()>,
}

```

```rust
// confirm_dialog.rs

html! {
    <div class="confirm-dialog">
        <MyButton onclick=|_| DialogMsg::Cancel color=Color::Red hidden=true />
        <MyButton onclick=|_| DialogMsg::Submit color=Color::Blue />
    </div>
}
```

### フラグメント

Yewはフラグメントをサポートします。親のない要素はどこかに接続できます。

```rust
html! {
    <>
        <tr><td>{ "Row" }</td></tr>
        <tr><td>{ "Row" }</td></tr>
        <tr><td>{ "Row" }</td></tr>
    </>
}
```

### Virtual DOM、独立ループ、細かい更新

Yewは独自の**virtual-dom**実装を使用します。 要素のプロパティが変更されると、小さなパッチでブラウザのDOMを更新します。 すべてのコンポーネントは、メッセージの受け渡しを通じて環境(`Scope`)と対話する独自の独立したループ内に存在し、レンダリングの微調整をサポートします。

`ShouldRender`は、コンポーネントをいつ再レンダリングする必要があるかをループに通知する値を返します。

```rust
fn update(&mut self, msg: Self::Message) -> ShouldRender {
    match msg {
        Msg::UpdateValue(value) => {
            self.value = value;
            true
        }
        Msg::Ignore => {
            false
        }
    }
}
```

モデルへのすべての変更がビューの更新を引き起こすわけではないため、更新のたびにモデルを比較するよりも`ShouldRender`を使用する方が効果的です。フレームワークは、ビューのレンダリングに不可欠なモデルの一部のみを比較できます。

### テンプレート内のRust/JS/Cスタイルのコメント

htmlテンプレート内で単一行または複数行のRustコメントを使用します。

```rust
html! {
    <section>
   /* Write some ideas
    * in multiline comments
    */
    <p>{ "and tags can be placed between comments!" }</p>
    // <li>{ "or single-line comments" }</li>
    </section>
}
```

### サードパーティのcratesと内部の純粋なRust式

外部クレートを使用し、それらからテンプレートに値を入力します。

```rust
extern crate chrono;
use chrono::prelude::*;

impl Renderable<Model> for Model {
    fn render(&self) -> Html<Self> {
        html! {
            <p>{ Local::now() }</p>
        }
    }
}
```

> いくつかのクレートは、真のwasmターゲット(`wasm32-unknown-unknown`)をまだサポートしていません。

### サービス

YewはJavaScriptアラート、タイムアウト、ストレージ、フェッチ、Webソケットなどの外部APIを呼び出すことができるプラグ可能なサービスを実装しています。
サブスクリプションに代わる便利な代替手段です。

実装済み
* `IntervalService`
* `RenderService`
* `ResizeService`
* `TimeoutService`
* `StorageService`
* `DialogService`
* `ConsoleService`
* `FetchService`
* `WebSocketService`
* `KeyboardService`

```rust
use yew::services::{ConsoleService, TimeoutService};

struct Model {
    link: ComponentLink<Model>,
    console: ConsoleService,
    timeout: TimeoutService,
}

impl Component for Model {
    fn update(&mut self, msg: Self::Message) -> ShouldRender {
        match msg {
            Msg::Fire => {
                let send_msg = self.link.send_back(|_| Msg::Timeout);
                self.timeout.spawn(Duration::from_secs(5), send_msg);
            }
            Msg::Timeout => {
                self.console.log("Timeout!");
            }
        }
    }
}
```

不可欠なサービスが見つかりませんか？`npm`のライブラリを使用したいですか？
`stdweb`を使用して`JavaScript`ライブラリをラップし、独自のサービス実装を作成できます。 以下は`ccxt`(https://www.npmjs.com/package/ccxt)ライブラリをラップする方法の例です。

```rust
pub struct CcxtService(Option<Value>);

impl CcxtService {
    pub fn new() -> Self {
        let lib = js! {
            return ccxt;
        };
        CcxtService(Some(lib))
    }

    pub fn exchanges(&mut self) -> Vec<String> {
        let lib = self.0.as_ref().expect("ccxt library object lost");
        let v: Value = js! {
            var ccxt = @{lib};
            console.log(ccxt.exchanges);
            return ccxt.exchanges;
        };
        let v: Vec<String> = v.try_into().expect("can't extract exchanges");
        v
    }

    //ここに他のメソッドをラップします！
}
```

### 使いやすいデータ変換および構造化
Yewではシリアル化（保存/送信および復元/受信）形式が可能です。

実装済み: `JSON`, `TOML`, `YAML`, `MSGPACK`, `CBOR`.

開発中: `BSON`, `XML`.

```rust
use yew::format::Json;

#[derive(Serialize, Deserialize)]
struct Client {
    first_name: String,
    last_name: String,
}

struct Model {
    local_storage: StorageService,
    clients: Vec<Client>,
}

impl Component for Model {
    fn update(&mut self, msg: Self::Message) -> ShouldRender {
        Msg::Store => {
            // JSONで保存します
            self.local_storage.store(KEY, Json(&model.clients));
        }
        Msg::Restore => {
            // JSON形式のデータとして読み取りおよび構造化を試みます
            if let Json(Ok(clients)) = self.local_storage.restore(KEY) {
                model.clients = clients;
            }
        }
    }
}
```

デフォルトでは`JSON`のみが使用可能ですが、プロジェクトの`Cargo.toml`の機能を使用して残りをアクティブにできます。

```toml
[dependencies]
yew = { git = "https://github.com/yewstack/yew", features = ["toml", "yaml", "msgpack", "cbor"] }
```

## 開発セットアップ

このリポジトリをCloneまたはダウンロードします。

### [cargo-web]のインストール 

これは、Webアプリケーションの展開を簡素化するオプションツールです。

```bash
cargo install cargo-web
```

> `--force`オプションを追加して、最新バージョンを確実にインストールしてください。

### ビルド

```bash
cargo web build

# without cargo-web, only the wasm32-unknown-unknown target is supported
cargo build --target wasm32-unknown-unknown
```

### テストの実行

```bash
./ci/run_tests.sh
```

### サンプルを実行する

フレームワークの仕組みを示す多くの例があります。
[counter], [crm], [custom_components], [dashboard], [fragments],
[game_of_life], [mount_point], [npm_and_rest], [timer], [todomvc], [two_apps].

サンプルを開始するには、ディレクトリを入力し、[cargo-web]で開始します。

```bash
cargo web start
```

デバッグビルドの代わりに最適化ビルドを実行するには次のようにします。

```bash
cargo web start --release
```

これはデフォルトで`wasm32-unknown-unknown`ターゲットを使用します。これはRustのネイティブWebAssemblyターゲットです。
Emscriptenベースの`wasm32-unknown-emscripten`および`asmjs-unknown-emscripten`ターゲットは、`--target`パラメーターを使用してビルドするように`cargo-web`に指示した場合にもサポートされます。

[counter]: examples/counter
[crm]: examples/crm
[custom_components]: examples/custom_components
[dashboard]: examples/dashboard
[fragments]: examples/fragments
[game_of_life]: examples/game_of_life
[mount_point]: examples/mount_point
[npm_and_rest]: examples/npm_and_rest
[timer]: examples/timer
[todomvc]: examples/todomvc
[two_apps]: examples/two_apps
[cargo-web]: https://github.com/koute/cargo-web


## プロジェクトテンプレート

* [`yew-wasm-pack-template`](https://github.com/ryugou/yew-wasm-pack-template)
* [`yew-wasm-pack-minimal`](https://github.com/ryugou/yew-wasm-pack-minimal)
