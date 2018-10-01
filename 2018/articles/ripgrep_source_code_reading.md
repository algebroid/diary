## ripgrepのソースコードを読む

ripgrepはRustで書かれたgrepツールである。 ripgrepのソースコードは、RustコミュニティではRustらしい作法で書かれたクリーンなコードとして知られている((redditかユーザーフォーラムでたまに名前が挙がる。ソース失念))。

[ここでgrep系ツールの機能が比較されている](https://beyondgrep.com/feature-comparison/)

今回はバージョン[0.10.0のコード](https://github.com/BurntSushi/ripgrep/tree/0.10.0)を読もうと思います。

## `main`

ripgrepは[binアプリケーションであり、mainをエントリポイントとしている](https://github.com/BurntSushi/ripgrep/blob/master/Cargo.toml#L25-L28)。

> [`main.rs`](https://github.com/BurntSushi/ripgrep/blob/master/src/main.rs)

mainは[次のようになっている](https://github.com/BurntSushi/ripgrep/blob/0.10.0/src/main.rs#L38-L47)。

```rust
fn main() {
    match Args::parse().and_then(try_main) {
        Ok(true) => process::exit(0),
        Ok(false) => process::exit(1),
        Err(err) => {
            eprintln!("{}", err);
            process::exit(2);
        }
    }
}
```

`Args`モジュールからコマンドライン引数をパースし、`try_main`関数を実行している。`try_main`の返り値の型は`Results<bool>`で、結果に応じて終了処理を振り分けている。

`Arg::parse()`から何をしているか見ていく。

## `Args::parse()`

まず、`Args`とは何か。コメントには以下のように記されている。

>  The primary configuration object used throughout ripgrep. It provides a
>  high-level convenient interface to the provided command line arguments.
> 
>  An `Args` object is cheap to clone and can be used from multiple threads
>  simultaneously.

> （Argは）ripgrepのプログラム中を通じて使われる一級の設定オブジェクトである。
> 
> これは与えられたコマンドライン引数に対する高レベルで便利なインターフェースを提供する。
> 
> Argsオブジェクトはcloneが安価であり、複数のスレッドから同時にアクセスすることができる。

実際、`Args`構造体の定義は`pub struct Args(Arc<ArgsImp>);`というタプル構造体となっているので、スレッドセーフにアクセス可能であることがわかる。

`Arg::parse()`はどうなっているのか。実装は以下のようになっている。

> [`Args::parse()`](https://github.com/BurntSushi/ripgrep/blob/0.10.0/src/args.rs#L125-L160)

```rust
pub fn parse() -> Result<Args> {
        // We parse the args given on CLI. This does not include args from
        // the config. We use the CLI args as an initial configuration while
        // trying to parse config files. If a config file exists and has
        // arguments, then we re-parse argv, otherwise we just use the matches
        // we have here.
        let early_matches = ArgMatches::new(app::app().get_matches());
        set_messages(!early_matches.is_present("no-messages"));
        set_ignore_messages(!early_matches.is_present("no-ignore-messages"));

        if let Err(err) = Logger::init() {
            return Err(format!("failed to initialize logger: {}", err).into());
        }
        if early_matches.is_present("trace") {
            log::set_max_level(log::LevelFilter::Trace);
        } else if early_matches.is_present("debug") {
            log::set_max_level(log::LevelFilter::Debug);
        } else {
            log::set_max_level(log::LevelFilter::Warn);
        }

        let matches = early_matches.reconfigure();
```

まず最初の行でCLIからパース与えられた引数をパースしている。コメントにあるようにこれはconfigの引数を含まない。

`app::app()`は`clap`アプリケーションを構築する関数である。`app`モジュール内では`ripgrep`がサポートするコマンドライン引数を定義している。`ripgrep`のソースコード内で`clap`を扱っているのは`src/app.rs`を除いては`src/args.rs`のみである。

`.get_matches()`はclapの引数をパースするメソッドであり`clap`の`clap::ArgMatches`を返す。`ArgMatches::new()`は`clap::ArgMatches`をラップしたripgrep独自の構造体で、`struct ArgMatches(clap::ArgMatches<'static>)`という型を持っている。

`set_messages`や`set_ignore_messages`は`messages.rs`に定義されているモジュール関数である。`no-messages`オプション、`no-ignore-messages`オプションが指定されているかをフラグとして、static生存期間を持つ`AtomicBool`変数に保存している。

次いでロガーを構築している。`--trace`オプションは`rg --help`には紹介されていないが使えるようだ。

`early_matches.reconfigure()`はコメントにある通り、configファイルが存在すればそれに基づいて`ArgMatches`を再構築し、存在しなければなにもしない。

関数の後半を見ていく。

```rust
        // The logging level may have changed if we brought in additional
        // arguments from a configuration file, so recheck it and set the log
        // level as appropriate.
        if matches.is_present("trace") {
            log::set_max_level(log::LevelFilter::Trace);
        } else if matches.is_present("debug") {
            log::set_max_level(log::LevelFilter::Debug);
        } else {
            log::set_max_level(log::LevelFilter::Warn);
        }
        set_messages(!matches.is_present("no-messages"));
        set_ignore_messages(!matches.is_present("no-ignore-messages"));
        matches.to_args()
}
```

デジャヴかな？　ロガーの設定をやり直している。

`matches.to_args()`で結果を返している。これは`ArgsMatches`の値をバラして、パターンや利用するマッチャを解析してからripgrepの設定の元締めたる`ArgsImp`構造体に包みなおしている。

