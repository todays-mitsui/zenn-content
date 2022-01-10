---
title: "Elmとaxumでポートフォリオ作ってみた"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Elm", "rust"]
published: false
---

# はじめに

私は駆け出してすらいないプログラマーですが、1～2年後に駆け出してみようかなという選択肢も浮かんでいるので、勉強のためにポートフォリオサイトを作ってみようと思いました。

# 技術選定
以下の技術要素で開発を行いました

- フロントエンド
    - Elm 
- バックエンド
    - Rust
        - axum (Webフレームワーク)
        - sqlx (データベースを使うためのクレート)
        - tracing (ロギングをするクレート)
        - and more

フロントエンドをElmにした理由はJavaScriptが難しすぎて私の実力では全く書けないからで、バックエンドをRustにしたのはその他の言語をよく知らないからです。
# 制作物
こんな感じで制作しました。(CSSが全く分からないのでクソみたいな見た目です。)
デプロイするメリットがとくにないのと、DDoSなどで高額請求がきたら嫌なのでデプロイはしてません。バックエンドは一応コンテナ化はしています。
バックエンドでAPIサーバーを建てる必要が全くないほどシンプルなのですが、勉強のためにバックエンドからデータを取得するようにしています。

## TOPページ
![](/images/top.jpeg)
## Aboutページ
![](/images/about.jpeg)
## Careerページ
![](/images/career.jpeg)
## Skillページ
![](/images/skill.jpeg)

GitHubのURLは以下です
- https://github.com/hayaoR/portfolio-frontend
- https://github.com/hayaoR/portfolio-backend

# フロントエンド
フロントエンドはTOPページ、Aboutページ、Careerページ、SkillページからなるSPAにしました。
SPAにする必要は全くないのですが、練習のためにSPAにしました。

## SPA
ElmでSPAを作るなら[elm-spa](https://www.elm-spa.dev/)を使ったほうが良いんだろうなと思いましたが、私は勉強のために素のElmで実装しました。
とはいえほとんど[Elm in Action](https://www.manning.com/books/elm-in-action)に載っていたSPAのコードの構成に習っています。

以下ざっくりポイントを説明します。
まず1ページ1ページを普通のElmのアプリケーションだと思って作ります。そこからどうSPA化するかという話です。
`Main.elm`(名前は何でもいい)というファイルを作ってそこに記述していきます。

### ページ
ページという型を導入し、それぞれにそのページのModelをもたせます。
```elm
type Page
    = TopPage Top.Model
    | AboutPage About.Model
    | CareerPage Career.Model
    | SkillPage Skill.Model
    | NotFound
```

### モデル
モデルはページとNavigation.Keyを持っています。Navigation.KeyはURLを変更するNavigationコマンドで必要とされるものです。
これにより、pageの値を適切にコントロールすることで、各ページは自身のModelにアクセスできるようになります。

```elm
type alias Model =
    { page : Page
    , key : Nav.Key
    }
```

### ビュー
ビューはmodelのpageに応じておのおのpageのview関数を呼び出します。
それぞれのページのviewが返すMsgはMainのMsgではないので調整が必要です。

```elm
view model =
    let
        content =
            case model.page of
                TopPage top ->
                    Top.view top
                        |> Html.map GotTopMsg

                AboutPage about ->
                    About.view about
                        |> Html.map GotAboutMsg

                CareerPage career ->
                    Career.view career
                        |> Html.map GotCareerMsg

                SkillPage skill ->
                    Skill.view skill
                        |> Html.map GotSkillMsg

                NotFound ->
                    text "Not Found"
    in
    { title = "Hayao's portfolio"
    , body =
        [ div [ class "container" ]
            [ viewHeader model.page
            , content
            , viewFooter
            ]
        ]
    }
```

### アップデート
他のURLに変更しようとしたときとURLが変わったときにメッセージがくるので、その対処をします
`updateUrl url model`がCmd Got～Msgを出すようにしているのでGot～Msgが来たらそれぞれのページのupdate関数を呼びます。
それぞれのページのupdateが返すMsgはMainのMsgではないので調整が必要です。

```elm

update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
    case msg of
        -- URL変更リクエストがきたとき　ここの記述は定型文っぽい
        UrlRequest urlRequest ->
            case urlRequest of
                Browser.External href ->
                    ( model, Nav.load href )

                Browser.Internal url ->
                    ( model, Nav.pushUrl model.key (Url.toString url) )

        -- URLが変更されたとき、URLをパースして適切なMsgを送る
        ChangedUrl url ->
            updateUrl url model

        GotTopMsg topMsg ->
            case model.page of
                TopPage top ->
                    toTop model (Top.update topMsg top)

                _ ->
                    ( model, Cmd.none )

        GotAboutMsg aboutMsg ->
            case model.page of
                AboutPage about ->
                    toAbout model (About.update aboutMsg about)

                _ ->
                    ( model, Cmd.none )

        GotCareerMsg careerMsg ->
            case model.page of
                CareerPage career ->
                    toCareer model (Career.update careerMsg career)

                _ ->
                    ( model, Cmd.none )

        GotSkillMsg skillMsg ->
            case model.page of
                SkillPage skill ->
                    toSkill model (Skill.update skillMsg skill)

                _ ->
                    ( model, Cmd.none )
```

## Elmの勉強法
私はElm in Actionを読みましたが、とても良い本でした。ただ英語なので、[Elm Guide](https://guide.elm-lang.jp/)と[プログラミングElm](https://book.mynavi.jp/ec/products/detail/id=120921)を読んだほうが良いかもしれません。

# バックエンド
バックエンドはただフロントエンドで表示するデータをJSONで返すだけです。

## CORS
フロントエンドとバックエンドを違うOriginで走らせていたので、CORSの設定が必要でした。
なので[tower-http](https://github.com/tower-rs/tower-http)を[corsモジュール](https://docs.rs/tower-http/latest/tower_http/cors/index.html)を利用しました。

```rust
use tower_http::cors::{CorsLayer, Origin};

let app = Router::new()
 .route("/skills", get(skills))
 .route("/about", get(about))
 .route("/careers", get(careers))
 .layer(
     CorsLayer::new()
         .allow_origin(Origin::exact("http://localhost:8000".parse().unwrap()))
         .allow_methods(vec![Method::GET]),
 )
 .layer(AddExtensionLayer::new(connection_pool)); 
```

## Rustの勉強法
Rustは二年くらいまえから知っていたので、どう勉強したのかは忘れました。
RustのWebプログラミングについては[zero to production in rust](https://www.zero2prod.com/)という本が出色の出来です。
無料で記事も読めるみたいです。
axumについてはGitHubのexampleフォルダが充実しているのでそこを見ると勉強になります。
sqlxとtracingは良くわからないのでだれか詳しい解説記事書いてほしいです。

# おわりに
駆け出しエンジニアがよく作るポートフォリオサイトを作ってみました。(私は駆け出してすらいませんが)
JavaScriptやRuby、Pythonなどの激むず言語で作っているひとが多いですが、腕のあるプログラマーですら苦戦することが多い言語を駆け出しプログラマーを使うのはとても大変だと思います。少なくとも私はうまく使えません。
実務で広く使われているのでJavaScriptやPythonをいずれは使わねばならないとしても最初はElmやRustでプログラミングに慣れてからJavaScriptなので激むず言語にチャレンジするというルートを辿るひとがもっといてもいいのになと思いました。