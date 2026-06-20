---
title: "Eloquent ORM の基本を理解する"
emoji: "🗄️"
type: "tech"
topics: ["laravel", "php", "database"]
published: false
---

## 一言で言うと
テーブルをPHPクラスとして扱い、メソッド・プロパティの操作を自動でSQLに変換するORM。

## 問い（復習時にここだけ見て答える）
- Eloquentとは何か？
- `$user->email` でプロパティにアクセスできるのはなぜか？
- `$user->name = 'Kenta'` はDBに保存されるか？

## ノート

### ORM（Object-Relational Mapping）
オブジェクト（PHPのクラス）とリレーショナルDB（テーブル）をマッピングする仕組みのこと。EloquentはLaravelのORM実装。

### モデルとテーブルの対応
- `User` クラス → `users` テーブル
- Userオブジェクトの1インスタンス → テーブルの1行（レコード）

### 内部の仕組み
Eloquentの親クラス `Model` は取得したレコードを内部の `$attributes` 配列に持つ。

```php
class Model {
    protected $attributes = [];

    public function __get($key) {
        return $this->attributes[$key];
    }

    public function __set($key, $value) {
        $this->attributes[$key] = $value;
    }
}
```

`$user->email` → `__get('email')` → `$attributes['email']` を返す  
`$user->name = 'Kenta'` → `__set('name', 'Kenta')` → `$attributes['name']` を上書き（まだDB未反映）

### データの取得・保存の流れ
| 操作 | 発行されるSQL |
|------|--------------|
| `User::find(1)` | `SELECT * FROM users WHERE id = 1` |
| `$user->save()` | `UPDATE users SET ... WHERE id = 1` |

## 自分の言葉で説明する
モデルを使ってSQLを発行するもの。直接SQLを書かなくても、PHPのオブジェクト操作で自動的にDB操作ができる。

## なぜそうなっているのか（Why）
PHPの `__get()` / `__set()` マジックメソッドを利用しているから。存在しないプロパティへのアクセスを横取りして、内部の配列から値を引っ張る仕組みになっている。

## 詰まったポイント・誤解していたこと
- `$user->email` でアクセスできるのは「Userクラスにemailプロパティが定義されているから」と思いがちだが、実際はテーブルのカラムを自動で読んでいる。クラスに書かなくていい。
- `$user->name = 'Kenta'` はメモリ上の変更のみ。`save()` を呼ばないとDBには反映されない。
