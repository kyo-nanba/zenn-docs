---
title: "外部キー制約が設定されているカラムのインデックスを張り直す on Rails"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails"]
published: true
---

## What

Ruby on Railsを使ったシステムでデータベースの変更を行う際、既に外部キー制約とindexが設定されているカラムにunique indexを張り直そうとしたところ少し面倒だったのでその際のメモです。

## Why

最近触っているRailsを使ったプロジェクトに[database_consistency](https://github.com/djezzzl/database_consistency)を導入したところ `MissingIndexChecker` で怒られました。これに関する詳細な説明は[公式のREADME](<https://github.com/djezzzl/database_consistency#missingindexchecker>)に記載がありますが、ざっくりいうと Foo has one Bar のようなリレーションにおいてbarsテーブルのfoo_idにunique indexが張ってなかった場合に発生するものです。

元々外部キー制約およびindexは設定していたのですがunique indexではなかったので、確かになーと思い今回設定しておこうとしたというのが作業の動機です。

## How

最終的にマイグレーションファイルのchangeメソッドはこうなりました。

```rb
def change
  safety_assured do
    change_table :bars, bulk: true do |t|
      t.remove_foreign_key :foos
      t.remove_index :foo_id
      t.index :foo_id, unique: true
      t.foreign_key :foos
    end
  end
end
```

以下は上記に至るまでの流れです。

### 最初

`add_index :bars, :foo_id, unique: true` とだけ書いて実行しようとする。

### 既にindexがあるぞと怒られる

確かに。と思い前行に `remove_index :bars, :foo_id` を加える。

### 外部キー制約があるカラムからindexは消せないぞと怒られる

確かに。と思い `remove_foreign_key :bars, foos` と `add_foreign_key :bars, foos` を処理の最初と最後に加える。

一応動いたようで `bin/rails db:rollback` も `bin/rails db:migrate:redo` も意図した動きになったので大丈夫そうだ。

### modelにuniquenessのバリデーションがないぞと怒られる

`bundle exec database_consistency` を実行すると[UniqueIndexChecker](https://github.com/djezzzl/database_consistency#uniqueindexchecker)が発生。

確かに。と思いBarモデルに `validates :foo_id, uniqueness: true` を追加。

### `Rails/BulkChangeTable` で怒られる

rubocopを実行したところrubocop-railsの [Rails/BulkChangeTable](https://docs.rubocop.org/rubocop-rails/cops_rails.html#railsbulkchangetable) で怒られが発生。

確かに。と思い `change_table :bars, bulk: true` を使った書き方に修正。

### Possibly dangerous operationと怒られる

これは[strong_migrations](https://github.com/ankane/strong_migrations)という別のgemの話なので本稿とは直接は関係ないが、危険な可能性のある処理だとわかって行う必要がある作業なので気をつけましょうという話。

確かに。と思いながら `safety_assured` のブロックで囲む。

大丈夫そう。というわけで最初のコードが出来上がり。

## 最後

というわけで、出来上がる処理は大したものではないものの後から修正するのは面倒なので has one の関係性のテーブルを作成する場合は最初からunique indexを設定した外部キー制約をつけましょうという話でした。

## 余談

`create_table` の段階で `references` に外部キー制約とユニークキー制約をつけたい場合に `foreign_key: true, unique: true` のように書いてしまい、動かしても何故かユニークキー制約が設定されないなーとなりがち。

正しくはこう。

```rb
create_table :bars do |t|
  t.references :foo, null: false, foreign_key: true, index: { unique: true }
end
```
