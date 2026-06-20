---
title: "EloquentとRaw SQLの使い分け"
emoji: "🆚"
type: "tech"
topics: ["laravel", "php", "sql", "database"]
published: true
---

## 一言で言うと
Eloquent と生SQL は二択ではなく、グラデーションになっている。

## 問い（復習時にここだけ見て答える）
- `selectRaw` は Eloquent か生SQL か？
- 生SQL（`DB::select()`）を使うべき場面を2つ挙げよ。
- Eloquent で大量一括処理が遅い理由は何か？

## ノート

### グラデーションの全体像

```
Eloquent 純粋 ←─────────────────────────→ 生SQL 純粋
User::find(1)   selectRaw()   DB::select('...')
```

`selectRaw`, `whereRaw`, `orderByRaw` は「Eloquent の枠組みの中に SQL の断片を埋め込む」もの。
複雑なクエリでもいきなり生SQL に逃げるのではなく、まずこれらで凌げないか試す。

### 使い分けの基準

| 場面 | 使うもの |
|---|---|
| 通常のCRUD | Eloquent |
| 複雑な集計（GROUP BY等） | `selectRaw` 等で凌ぐ、無理なら生SQL |
| Window関数・CTE（WITH句）等 | 生SQL |
| 大量一括処理 | Query Builder（`DB::table()->insert([])`） |

### 生SQLを使うべき場面

**1. Eloquent が対応していない SQL 構文**

`WINDOW関数`、`CTE (WITH句)`、DB固有の関数など、`Raw` 系メソッドでも表現しきれない場合。

**2. 大量データの一括処理**

Eloquent で `User::create()` を1万回ループすると、1回ごとに以下が走る：
- モデルのインスタンス生成
- `creating` / `created` イベントの発火
- タイムスタンプの自動セット

1回あたりのオーバーヘッドは小さいが、1万回積み重なると無視できなくなる。

生SQL なら SQL 1発で済む：

```php
DB::table('users')->insert([
    ['name' => 'A', 'email' => 'a@example.com'],
    ['name' => 'B', 'email' => 'b@example.com'],
    // 1万件
]);
```

## 自分の言葉で説明する
EloquentとSQLは二択でなくグラデーション。複雑なクエリはまず `selectRaw` 等で凌ぎ、それでも無理なときだけ完全な生SQLに切り替える。

## なぜそうなっているのか（Why）
Eloquentはモデルのインスタンス生成・イベント発火・タイムスタンプ自動セットをクエリごとに行う。1回あたりのコストは小さいが、大量処理では積み重なる。

## 詰まったポイント・誤解していたこと
- 「Eloquent か生SQL か」の二択で考えがちだが、`selectRaw` 等のミドルレイヤーがある。
- 「Eloquent のオーバーヘッドは小さい」は1回あたりの話。大量処理では積み重なる。
- 大量一括処理で Eloquent が遅いのは「SQLへの変換コスト」ではなく、モデルのインスタンス生成・イベント発火が毎回走るコスト。
