---
title: "Dieselで条件に応じた検索条件を組み立てる(前編)"
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

    let query = match keyword {
        Some(k) => {
            let keyword = format!("%{}%", k);
            threads.filter(thread_title.like(&keyword))
        },
        None => threads
    };

    let ts: Vec<Thread> = query
        .order(updated_at.desc())
        .limit(THREADS_PER_PAGE)
        .offset(offset)
        .get_results(&**conn)?;
    Ok(ts)
}
```

しかしコンパイルすると以下のエラー。
```sh
error[E0308]: `match` arms have incompatible types
  --> src/functions.rs:60:13
   |
54 |       let query = match keyword {
   |  _________________-
55 | |         Some(k) => {
56 | |             let keyword = format!("%{}%", k);
57 | |             threads.filter(thread_title.like(&keyword))
   | |             ------------------------------------------- this is found to be of type `diesel::query_builder::SelectStatement<schema::threads::table, diesel::query_builder::select_clause::DefaultSelectClause, diesel::query_builder::distinct_clause::NoDistinctClause, diesel::query_builder::where_clause::WhereClause<diesel::expression::operators::Like<schema::threads::columns::thread_title, diesel::expression::bound::Bound<diesel::sql_types::Text, &std::string::String>>>>`
...  |
60 | |             threads
   | |             ^^^^^^^ expected struct `diesel::query_builder::SelectStatement`, found struct `schema::threads::table`
61 | |         }
62 | |     };
   | |_____- `match` arms have incompatible types
   |
   = note: expected type `diesel::query_builder::SelectStatement<schema::threads::table, diesel::query_builder::select_clause::DefaultSelectClause, diesel::query_builder::distinct_clause::NoDistinctClause, diesel::query_builder::where_clause::WhereClause<diesel::expression::operators::Like<schema::threads::columns::thread_title, diesel::expression::bound::Bound<diesel::sql_types::Text, &std::string::String>>>>`
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

どうしたもんかね、とぐぐってみるとそれらしきIssueを発見。軽く読んだけど同じ問題にぶち当たってるみたいだし、参考になりそう。(続く)  
[How to build a conditional query #455](https://github.com/diesel-rs/diesel/issues/455)