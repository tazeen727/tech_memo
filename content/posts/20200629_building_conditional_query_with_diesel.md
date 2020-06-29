---
title: "Dieselで条件に応じた検索条件を組み立てる"
date: 2020-06-29T21:00:00+09:00
draft: false
tags: ["Rust", "Diesel", "O/R Mapper"]
---
久しぶりの投稿。
現在、超絶簡易な掲示板Webアプリを作成している。主な言語としてRustを使用しており、O/R MapperとしてDieselを選択した。
<!--more-->

というかRustのO/R MapperってDiesel以外なかなか出てこないんだよね。
まあ使った言語やらフレームワークやら、技術選択の良し悪しはここでは置いておく。
今回ハマったのは、Dieselで動的にクエリを組み立てる方法。Rustの型システムがJavaとかScalaと異なることも相まって、二日くらい悩んだ。

### やりたいこと
スレッド一覧画面で、スレッド検索機能をつけたい。検索ワードを入力してボタンを押すと
タイトルの一部に検索ワードを含むスレッドの一覧を表示する、というごく簡易なものだ。
ただし、検索結果はスレッド一覧画面そのものなので、検索ワードが未入力なら検索ワードによる条件もなし。

つまり以下の条件分岐となる
- 検索ワード指定あり ... 検索ワードを含むスレッドだけ表示
- 検索ワード指定なし ... すべてのスレッドを表示

検索ワードありの場合のDieselのクエリは以下。
```rust
threads
    .filter(thread_title.like(&keyword))
    .order(updated_at.desc())
    .limit(THREADS_PER_PAGE)
    .offset(offset)
    .get_results(&**conn)?;
```

対して、検索ワードなしの場合は以下。
```rust
threads
    .order(updated_at.desc())
    .limit(THREADS_PER_PAGE)
    .offset(offset)
    .get_results(&**conn)?;
```

`filter()`メソッドがあるかないか、の違いでしかない。重複を排除したくなるのは当然だろう。
したがってこう書きたい。

```rust
pub fn get_threads(conn: &DbConn, keyword: Option<String>, offset: i64) -> Result<Vec<Thread>> {
    use schema::threads::dsl::*;
    use crate::globals::THREADS_PER_PAGE;

    let keyword = keyword.map(|k| format!("%{}%", k));

    let query = match keyword {
        Some(k) => {
            threads.filter(thread_title.like(k))
        },
        None => {
            threads
        }
    };

    let ts: Vec<Thread> = query
        .order(updated_at.desc())
        .limit(THREADS_PER_PAGE)
        .offset(offset)
        .load(&**conn)?;
    Ok(ts)
}
```

しかしコンパイルすると以下のエラー。
```sh
error[E0308]: `match` arms have incompatible types
  --> src/functions.rs:61:13
   |
56 |       let query = match keyword {
   |  _________________-
57 | |         Some(k) => {
58 | |             threads.filter(thread_title.like(k))
   | |             ------------------------------------ this is found to be of type `diesel::query_builder::SelectStatement<schema::threads::table, diesel::query_builder::select_clause::DefaultSelectClause, diesel::query_builder::distinct_clause::NoDistinctClause, diesel::query_builder::where_clause::WhereClause<diesel::expression::operators::Like<schema::threads::columns::thread_title, diesel::expression::bound::Bound<diesel::sql_types::Text, std::string::String>>>>`
59 | |         },
60 | |         None => {
61 | |             threads
   | |             ^^^^^^^ expected struct `diesel::query_builder::SelectStatement`, found struct `schema::threads::table`
62 | |         }
63 | |     };
   | |_____- `match` arms have incompatible types
   |
   = note: expected type `diesel::query_builder::SelectStatement<schema::threads::table, diesel::query_builder::select_clause::DefaultSelectClause, diesel::query_builder::distinct_clause::NoDistinctClause, diesel::query_builder::where_clause::WhereClause<diesel::expression::operators::Like<schema::threads::columns::thread_title, diesel::expression::bound::Bound<diesel::sql_types::Text, std::string::String>>>>`
            found struct `schema::threads::table`

```

要するに、match式の腕の型が一致してないぞと。しかも`Some(k)`の方の型はなんだこりゃ。
中に`WhereClause<Like<thread_title>>`とかが入っているのを見る限り、クエリの組み立てに使った関数によって型が変わるってこと？
それだと条件に応じて違うクエリを組み立てるとか無理じゃない？

ちなみに変数`query`の型を`impl QueryDsl`とかにしてもダメだった。match式の腕は全部同じ「型」でないといけない。
同じtraitを実装しているかどうかは関係ないのだ。JavaとかScalaだと同じinterfaceやtraitを実装しているclassのインスタンスは、
interfaceやtraitを型とする変数に格納できたけどRustでは**traitは型ではない**。trait objectを使えば動的ディスパッチを実現できるけど
[厳しい制約がある。](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#object-safety-is-required-for-trait-objects)
今回扱いたい`QueryDsl`はobject safeじゃなかった。

どうしたもんかね、とぐぐってみるとそれらしきIssueとドキュメントへのLinkを発見。軽く読んだけど同じ問題にぶち当たってるみたいだし、参考になりそう。
- [How to build a conditional query #455](https://github.com/diesel-rs/diesel/issues/455)
- [diesel::query\_dsl::QueryDsl \- Rust](http://docs.diesel.rs/diesel/query_dsl/trait.QueryDsl.html#method.into_boxed)

つまり、いろんな型をもつ`QueryDsl`でも`into_boxed()`を呼び出すと単一の型に変換できるということらしい。
その代わり多少パフォーマンスは落ちるとのこと。

また、上のドキュメントにも記述されている通り、`into_boxed()`を呼び出すには型パラメータ`DB: Backend`を指定しないといけない。
書き方には少し気をつけないと、型パラメータの推論ができずにコンパイルできない。
どの`Backend`に対してクエリを組み立てるかという情報は`Connection` traitを実装している変数`conn`で持っている。
したがって`into_boxed()`を呼び出したのと同じ変数で`load()`メソッドを呼び出してやれば、型推論してくれるので`DB`を指定する必要はない。

以上を踏まえてコードを修正すると以下のようになる。
```rust
pub fn get_threads(conn: &DbConn, keyword: Option<String>, offset: i64) -> Result<Vec<Thread>> {
    use schema::threads::dsl::*;
    use crate::globals::THREADS_PER_PAGE;

    let keyword = keyword.map(|k| format!("%{}%", k));
    
    let mut query = match keyword {
        Some(k) => {
            threads.filter(thread_title.like(k)).into_boxed()
        },
        None => {
            threads.into_boxed()
        }
    };

    query = query
        .order(updated_at.desc())
        .limit(THREADS_PER_PAGE)
        .offset(offset);

    let ts: Vec<Thread> = query.load(&**conn)?;
    Ok(ts)
}
```

ひとまずコンパイルは通るようになった。けどなんか野暮ったので上のコードはまた修正するかも。
