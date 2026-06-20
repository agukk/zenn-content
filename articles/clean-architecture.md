---
title: "Clean Architectureをざっくり理解する"
emoji: "🏗️"
type: "tech"
topics: ["cleanarchitecture", "laravel", "php"]
published: true
---

## 一言で言うと
ビジネスのルールが外側（Laravel・DB）に依存しないよう、依存の向きを外→内の一方向に制限した設計。

## 問い（復習時にここだけ見て答える）
- Clean Architectureとは何か、一言で説明せよ。
- 4つの層（Entities・Use Cases・Interface Adapters・Frameworks）それぞれの役割は？
- なぜビジネスのルールをLaravelやDBに依存させてはいけないのか？
- 「内側」と「外側」とは何を指すか？具体例で答えよ。

## ノート

### 4つの層

```
外側
  Frameworks（Laravel・DB）
    外側
      Interface Adapters（Controller・Repository）
        内側
          Use Cases（Service）
            一番内側
              Entities（isStopped()など）
```

| 層 | 役割 | 自分のコードで言うと |
|---|---|---|
| Entities | ビジネスのルールを定義する | `isStopped()`・`isExpired()` などモデルのメソッド |
| Use Cases | ビジネスのフロー（処理の流れ）を担う | Serviceクラス |
| Interface Adapters | 内側と外側のつなぎ役 | Controller・Repository実装 |
| Frameworks | ツール全般 | Laravel・Eloquent・DB |

### 依存のルール
- 外側が内側を使う（依存する）
- 内側は外側を知らない
- 矢印は常に内側に向かう一方向のみ

### なぜServiceがEloquentモデルを直接使うと問題か
- ServiceがEloquentを知っている = ServiceはLaravelに依存している
- ORM差し替えのとき、Repositoryだけでなく Serviceも書き換えが必要になる
- テスト時にDBが必要になる（偽物に差し替えられない）

理想はServiceがDTOを受け取る形（Pure PHPクラス）だが、Laravelの現場ではEloquentをServiceで使うことは一般的。規模次第でYAGNI判断。

## 自分の言葉で説明する
クリーンアーキテクチャというのは、依存の向きを適切にするもので、ビジネスに関するルールが一番内側にある。その内側にある内容は、外側の内容（LaravelやDB）に依存しないよう、適切にルールを定義付けるもの。

## なぜそうなっているのか（Why）
- ビジネスのルール（`isStopped()`など）はLaravelと関係ない。「停止しているかどうか」を判断するのにORMは不要。
- 内側がLaravelを知らなければ、LaravelをDoctrine等に差し替えてもEntitiesとUse Casesは一切触らなくて済む。
- テストもDBなしで内側だけ検証できる。

## 詰まったポイント・誤解していたこと
- 「内側・外側」という表現がぼんやりしていた → 同心円のイメージで理解できた（真ん中がビジネス、外に行くほどツールに近い）
- Clean Architectureを「新しい概念」として捉えていたが、Service・Repository・DIをすでに実践していた = 半分すでにやっていた
- 「依存を分散させる」と誤解していた → 正しくは「内側に向かう一方向に制限する」
