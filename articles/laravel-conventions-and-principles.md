---
title: "Laravelの慣習と原則"
emoji: "🔧"
type: "tech"
topics: ["laravel", "php", "backend"]
published: false
---

## 1. HTTPリクエストの基本

Laravelの慣習を理解する前提として、HTTPリクエストの仕組みを押さえておく必要がある。

ブラウザがサーバーに送っているのは「テキストの塊」

```
POST /register HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded

name=田中太郎&email=tanaka@example.com&password=secret123
```

これがHTTPリクエスト。ブラウザはこのテキストを自分で組み立ててサーバーに送っている。curlやPostmanは、このテキストを自分で組み立てて送るツール。

サーバー側から見ると、ブラウザから来たのかcurlから来たのか区別できない。どちらも同じフォーマットのテキストが届くだけ。

### フロントエンドだけのバリデーションが不十分な理由

フロントエンドのバリデーションはUIとしての使いやすさのためであり、curlなどで直接HTTPリクエストを送られると突破される。サーバー側のバリデーション（FormRequest）が必須。

| バリデーションの場所 | 役割 |
|---|---|
| フロントエンド（JS） | UXのため。エラーをすぐ見せる。突破される前提 |
| サーバー側（FormRequest） | セキュリティのため。これがなければ安全にならない |

---

## 2. Eloquentの慣習

### $fillable — 入力を制限する

`$fillable` に指定したカラムだけが、マスアサインメント（`create()` や `fill()` などの一括代入）で書き込みを受け付ける。

```php
protected $fillable = [
    'name',
    'email',
    'password',
    // is_admin は意図的に含めない
];
```

**マスアサインメント脆弱性**とは、悪意のあるユーザーがHTTPリクエストに `is_admin=1` などのフィールドを勝手に追加して送信し、権限を奪う攻撃。`$fillable` で保護していないと、そのフィールドがそのままDBに書き込まれる。

`$fillable` に書くのは、ユーザーが入力してくる項目だけ。

`$guarded = []`（全カラムの書き込みを許可）は、新しいカラムを追加したとき、気づかないまま脆弱性が生まれるため推奨されない。

### $hidden — 出力を制限する

`$hidden` に指定したカラムは、モデルをJSONや配列に変換するときに自動で除外される。パスワードや機密情報が外部に漏れるのを防ぐ。

```php
protected $hidden = ['password'];
```

`$fillable` と `$hidden` の方向性の違い：

| プロパティ | 方向 | 役割 |
|---|---|---|
| `$fillable` | 入力（書き込み） | 指定カラム以外の書き込みをブロック |
| `$hidden` | 出力（読み出し） | 指定カラムをJSON/配列から除外 |

### $casts — 型変換を保証する

DBから取得した値はPHPに渡す時点で文字列として扱われることがある（PDOの設定や環境によって異なる）。`$casts` を使うと、EloquentがモデルのプロパティにアクセスするたびにDB指定した型へ確実に変換してくれる。

```php
protected $casts = [
    'end_date'   => 'date',      // DBから取得した値をCarbonオブジェクトに変換
    'is_active'  => 'boolean',   // true/falseに変換
];
```

`$casts` を使うことで、呼び出し側で毎回型変換を書かなくて済む。

```php
// $casts なし
Carbon::parse($this->end_date)->endOfDay()->isPast();

// $casts あり
$this->end_date->endOfDay()->isPast();
```

**Carbonとは**：PHPのDateTime を拡張した日付操作ライブラリ。`isPast()`・`isFuture()`・`addDays()` など、日付に関する操作をメソッドとして持っている。Laravelに標準で組み込まれている。

### スコープ — whereの条件に名前をつける

スコープとは、クエリの絞り込み条件に名前をつけて、再利用できるようにしたもの。SQLで言うwhere句の条件をメソッドとして切り出すイメージ。

```php
// モデルに定義する（scopeというプレフィックスをつける）
public function scopeOfIsActive(Builder $query)
{
    return $query->where('is_active', config('master.is_active'));
}

// 呼び出し側（scopeプレフィックスを除いた名前で使える）
GuestAccount::ofIsActive()->get();
```

`scope` というプレフィックスをつけることで、Laravelがそれをスコープだと認識して、クエリビルダーに組み込んでくれる。

スコープを使う利点は2つ：

- **再利用できる** — 同じ条件を何度も書かなくていい
- **意図を伝える** — 条件の名前で何をしているかが一目でわかる

### アクセサ — 取り出すときの加工をモデルが持つ

アクセサとは、DBから取り出した値を加工して返すもの。加工の責任をモデルが持つことで、使う側（ControllerやBlade）は知らなくていい。

```php
// 旧記法（Laravel 8以前。Laravel 6.2ではこの記法が正しい）：
public function getFullNameAttribute()
{
    return $this->first_name . ' ' . $this->last_name;
}

// 呼び出し側（プロパティとしてアクセスできる）
$user->full_name
```

#### なぜプロパティアクセスで呼べるのか — `__get` の仕組み

PHPには、存在しないプロパティにアクセスすると自動で呼ばれる `__get` というマジックメソッドがある。LaravelのModelクラスはこれを実装していて、内部でうまく処理してくれる（概念的な表現）。

```php
public function __get($key)
{
    // 'status_label' → 'getStatusLabelAttribute' に変換して探す
    $method = 'get' . Str::studly($key) . 'Attribute';

    if (method_exists($this, $method)) {
        return $this->$method(); // あれば呼び出して返す
    }

    // なければDBのカラム値を返す（通常のEloquent動作）
}
```

`$guestAccount->status_label` とアクセスすると：

1. `status_label` というカラムは存在しない → `__get` が発火
2. `status_label`（スネークケース）を `StatusLabel`（スタディリーケース）に変換
3. `getStatusLabelAttribute` というメソッドを探して呼び出す。結果を返す

#### 命名規則

`get` + `スタディリーケース` + `Attribute` で定義すると、スネークケースのプロパティとして呼べる。

```
getStatusLabelAttribute()  →  $model->status_label
getFullNameAttribute()     →  $model->full_name
getEmailDomainAttribute()  →  $model->email_domain
```

#### アクセサにすべき場面・すべきでない場面

判断基準は、**「その加工は、どこで使っても同じ結果であるべきか」**。

| 対象 | アクセサにすべき | 理由 |
|---|---|---|
| `statusLabel()` → ステータスラベル | すべき | どこで使っても「止済み」「有効」は同じ。モデルの責任 |
| `isStopped()` → 止済み判定 | しない | `is_stopped` はDBカラムと見分けつかなくなる |
| 日付のフォーマット変換 | しない | 一覧では `Y/m/d`、詳細では `Y/m/d H:i` など、場所によって変わる |
| リレーションの文字列結合 | しない | 特定画面固有の表示形式になることが多い |

#### `$appends` — `toArray()` / `toJson()` に含める

アクセサを定義しただけでは `toArray()` / `toJson()` に自動では含まれない。`$appends` に追加した場合だけ含まれる。

```php
protected $appends = ['status_label'];

// これで toArray() の結果に status_label が入るようになる
$guestAccount->toArray();
// → ['name' => '田中太郎', 'email' => '...', 'status_label' => '有効', ...]
```

APIレスポンスなどでモデルをそのままJSON化する場合に使う。Bladeで個別に呼び出すだけなら `$appends` は必須ではない。

### ミューテタ — 保存するときの加工をモデルが持つ

ミューテタとは、DBに保存するときに値を加工してから保存するもの。

```php
// 旧記法（Laravel 8以前。Laravel 6.2ではこの記法が正しい）：
public function setPasswordAttribute($value)
{
    $this->attributes['password'] = Hash::make($value);
}

// 呼び出し側（ハッシュ化を意識しなくていい）
$user->password = 'secret123'; // 自動でハッシュ化されてDBに保存される
```

ミューテタを使わず `Hash::make()` を毎回書くと、新しいパスワードを保存する処理を書くたびに書き忘れるリスクが生まれる。「常に同じ加工が必要なもの」はミューテタにすべき。

| 名前 | 方向 | 役割 |
|---|---|---|
| アクセサ | DBから取り出すとき | 値を加工して返す |
| ミューテタ | DBに保存するとき | 値を加工してから保存する |

---

## 3. サービスコンテナとサービスプロバイダー

### __construct とは

`__construct` は、クラスを `new` でインスタンスを作るときに自動で最初に呼ばれるメソッド。インスタンスを使える状態にするための初期化処理を書く場所。

```php
public function __construct(GuestAccountService $service)
{
    $this->service = $service; // 受け取ったものをプロパティに保存
}
```

### サービスコンテナ

`new` を自動でやってくれる仕組み。コントローラーの `__construct` に必要なクラスを書いておくと、Laravelが自動でインスタンスを作って渡してくれる。

```php
// サービスコンテナがない世界（自分で new する）
public function index()
{
    $repository = new GuestAccountRepository();
    $service = new GuestAccountService($repository);
    // メソッドのたびに new を書く必要がある
}

// サービスコンテナがある世界（__construct に書くだけ）
public function __construct(
    GuestAccountService $service,
    GuestAccountRepository $repository
) {
    $this->service = $service;
    $this->repository = $repository;
}
```

### サービスプロバイダー

サービスコンテナへの特別な指示を書く場所。「このクラスはシングルトンにしてほしい」などを事前に伝えておく。

```php
// app/Providers/AppServiceProvider.php
public function register()
{
    $this->app->singleton(ActorService::class, function ($app) {
        return new ActorService();
    });
}
```

サービスプロバイダーへの登録が必要なケース：

| ケース | 登録の要否 |
|---|---|
| 普通のクラスをインスタンス化するだけ | 不要（自動解決） |
| シングルトンにしたい | 必要 |
| インターフェースと実装クラスを結びつけたい | 必要 |
| 特殊な設定値を渡したい | 必要 |

### シングルトンとは

アプリケーション全体で1つのインスタンスだけ作って使いまわす設計。

`ActorService`（誰が操作したかの情報を保持するクラス）を例に挙げると、シングルトンでない場合、ミドルウェアでセットした情報がコントローラーで取り出せない。

```php
// シングルトンでない場合
$actorService = new ActorService(); // インスタンス①
$actorService->setActor('tokushimaru', $userId); // ①にセット

$actorService = new ActorService(); // インスタンス②（別物）
$actorService->getActor(); // ②は空っぽ → 情報がない
```

シングルトンにすることで、どこから取り出しても同じインスタンスが返り、情報を引き継げる。

---

## 4. Facade

`Auth::user()` や `DB::table()` のように `::` で呼び出す仕組みがFacade。

`::` はスコープ解決演算子（ダブルコロン）と呼ぶ。インスタンスを作らずに静的にアクセスするときに使う。

### Facadeの正体

Facadeは裏でサービスコンテナからインスタンスを取り出してメソッドを呼んでいる。

```php
// Facade が裏でやっていること（概念的な表現）
app('auth')->user();

// これを簡潔に書けるようにしたのが Facade
Auth::user();
```

### 利点と欠点

| | 内容 |
|---|---|
| **利点** | どこでも `::` で気軽に使える。`__construct` が不要 |
| **欠点** | 使いすぎると依存関係が隠れる。`__construct` を見ても何に依存しているかわからなくなる |

### 使いどころ

| 場面 | 向いているか |
|---|---|
| ルーティングなどのアプリ設定（`Route::get()`） | 向いている |
| コントローラーやサービスの中 | `__construct` の方が望ましい |
| `Log::info()` など補助的な用途 | 使ってもいい。ただし乱用しない |

---

## 5. Middleware

### Middlewareとは

コントローラーに届く前後に「割り込んで」処理を挟む仕組み。

名前の由来は `Middle`（中間）+ `Ware`（仕組み）。リクエストとコントローラーの間に処理を挟む。

```
HTTPリクエスト
    ↓
Middleware（番人） ← ここで判断する
    ↓ 通過
Controller
    ↓
HTTPレスポンス
```

### handleメソッドの構造

全てのMiddlewareは `handle` メソッドを持つ。

```php
public function handle($request, Closure $next)
{
    // コントローラーの前の処理

    $response = $next($request); // コントローラーを実行（通過させる）

    // コントローラーの後の処理

    return $response;
}
```

- `$next($request)` → 「通過させる」。次の処理に進む
- `return redirect()` → 「通過させない」。コントローラーに届く前に止める

### Middlewareの役割

「通す・止める」だけでなく、ログを記録したり、操作者情報をセットしたりといった処理にも使われる。共通点は「コントローラーの前後に割り込んでいる処理」であること。

複数のMiddlewareは、リクエスト時は左（先頭）から順に前処理を実行し、レスポンス時は逆順に後処理を実行する。

### $middleware / $middlewareGroups / $routeMiddleware の使い分け

`app/Http/Kernel.php` で3種類のMiddlewareを管理している。

| プロパティ | いつ適用されるか | 例 |
|---|---|---|
| `$middleware` | 全リクエストに必ず適用 | メンテナンスモードのチェック |
| `$middlewareGroups` | グループ名を指定したルートに適用 | `'web'`（Cookie・CSRF・セッション） |
| `$routeMiddleware` | ルートで個別に名前を指定するときに適用 | `'auth'`・`'set_regular_password'` |

`$routeMiddleware` に名前を登録することで、ルートファイルで短い名前で指定できる。

---

## 6. Laravelのベースクラス

### 継承の目的

Laravelは「よく使う機能」をベースクラスとして用意している。継承することで、そのクラスの全機能を引き継げる。

```php
class GuestAccountController extends Controller
```

`Controller` を継承することで、`GuestAccountController` は `Controller` に定義された全メソッドを `$this->メソッド名()` で使える。

### 継承の連鎖

このプロジェクトでは継承が2段階になっている。

```
Laravelの BaseController（Laravelが用意した基礎）
    ↓ 継承
app/Http/Controllers/Controller.php（プロジェクト共通の処理を追加）
    ↓ 継承
GuestAccountController（個別の処理を書く）
```

直接 Laravel の BaseController を継承せずに `Controller.php` を挟んでいる理由は、**このプロジェクト特有の共通処理**を追加するため。

全コントローラーで使う処理をベースクラスに書いておくことで、個別のコントローラーは自分の責任だけを書けばいい。

### Authenticatable

`GuestAccount` モデルはこう定義されている。

```php
class GuestAccount extends Authenticatable
```

`Authenticatable`（正式には `Illuminate\Foundation\Auth\User`）は、認証用インターフェース `Illuminate\Contracts\Auth\Authenticatable` を実装した Laravel のベースクラス。これを継承することで、`GuestAccount` モデルは認証システムに組み込まれる。`Auth::user()` や `Auth::guard('guest')` でユーザーが取得できるのは、モデルが `Authenticatable` を実装していて、認証システムから正しく扱われるため。

---

## 7. N+1問題とEager Loading

### N+1問題とは

リレーションを持つモデルの一覧を取得するとき、ループの中でリレーションにアクセスするたびにSQLが1回ずつ発行されてしまう問題。

```php
// N+1問題のあるコード（101回のクエリが発行される）
$accounts = GuestAccount::all();  // 1回

foreach ($accounts as $account) {
    echo $account->salesCars;  // アカウントの数だけSQLが発行される
}
```

`$account->salesCars` にアクセスした瞬間、Eloquentはそのアカウントの車両を取得するSQLを発行する（**Lazy Loading / 遅延読み込み**）。

**なぜまずいか**：データが増えるほど線形に悪化する。1,000件なら1,001回のクエリ。本番でDBサーバーに過大な負荷をかけ、ページが遅くなる。

### Eager Loading — with() で解決する

`with()` を使うと、全アカウントのIDを `IN (...)` にまとめた1本のSQLで一気に取得する。クエリは **2本** で済む。

```php
// Eager Loading（2回のクエリで済む）
$accounts = GuestAccount::with('salesCars')->get();
```

```sql
-- 1回目：ゲストアカウント全件
SELECT * FROM guest_accounts;

-- 2回目：全アカウントの車両を IN でまとめて取得
SELECT sales_cars.*
FROM sales_cars
INNER JOIN guest_account_sales_cars ...
WHERE guest_account_sales_cars.guest_account_id IN (1, 2, 3, ..., 100)
```

| 方式 | SQLの本数 | 仕組み |
|---|---|---|
| Lazy Loading（デフォルト） | 1 + N本 | アクセスした瞬間に1件ずつ取得 |
| Eager Loading（`with()`） | 2本 | IN句でまとめて取得し、PHPで振り分け |

### ネストしたリレーション

`GuestAccount → SalesCar → SmStore` のように連鎖したリレーションは、ドット記法で書く。

```php
// 2段階先のリレーションまでEager Loading
GuestAccount::with('salesCars.smStore')->get();
```

### 多対多リレーション（belongsToMany）

`GuestAccount` と `SalesCar` は多対多の関係。DBでは直接つなげられないため、中間テーブル（`guest_account_sales_cars`）を通じて表現する。

```
guest_accounts        guest_account_sales_cars      sales_cars
──────────────        ────────────────────────      ──────────
id                    guest_account_id  ───────────  id
name                  sales_car_id      ───────────  name
```

### プロパティアクセスとメソッド呼び出しの違い

```php
$account->salesCars    // プロパティアクセス → SQLを実行してCollectionを返す
$account->salesCars()  // メソッド呼び出し  → まだ実行されていないRelation（QueryBuilder）を返す
```

---

## 8. Collection のメソッド

### Collection とは

Eloquentでデータを取得すると、結果は `Collection` に入って返ってくる。PHPの配列をラップした入れ物で、`map()`・`filter()`・`pluck()`・`groupBy()` などのメソッドをチェーンでつなげられる。

### 主なメソッド

#### pluck() — 特定カラムだけ取り出す

```php
// 名前だけの配列
$names = $accounts->pluck('name');
// → ['山田太郎', '鈴木花子', ...]

// IDをキー、名前を値にした連想配列（selectボックスの選択肢作成に便利）
$options = $accounts->pluck('name', 'id');
// → [1 => '山田太郎', 2 => '鈴木花子', ...]
```

#### filter() — 条件で絞り込む

```php
$activeAccounts = $accounts->filter(fn($account) => !$account->isStopped());
```

#### map() — 各要素を変換する

```php
$names = $accounts->map(fn($account) => strtoupper($account->name));
```

#### groupBy() — キーでグループ化する

```php
$grouped = $salesCars->groupBy('sm_id');
// → [1 => [1号車, 2号車], 2 => [3号車, 4号車], ...]
```

### DBクエリとCollectionの使い分け

**DBで絞れるならDBで絞る。CollectionはDBで表現しにくいこと、PHPでやること。**

```php
// 良い：DBで1,000件だけ取得してPHPに渡す
GuestAccount::where('is_active', 1)->get();

// 悪い：10,000件全部PHPに渡してからCollectionで絞る
GuestAccount::all()->filter(fn($a) => $a->is_active);
```

---

## 9. マイグレーション

### マイグレーションとは

DBのスキーマ変更（テーブルの作成・変更・削除）をコードとして定義し、Gitで管理する仕組み。`php artisan migrate` を実行するだけで、どの環境でも同じDBの状態を再現できる。

### ファイル名の構造

```
2026_05_25_100000_create_guest_accounts_table.php
↑日時                ↑何をしたかの説明
```

日時をつけている理由は2つ：
1. いつ変更したかの履歴になる
2. **実行順序を保証する**（親テーブルが先、子テーブルが後）

### ファイルの構造

```php
public function up()
{
    // php artisan migrate で実行される（変更を適用）
    Schema::create('guest_accounts', function (Blueprint $table) {
        $table->increments('id');
        $table->string('login_id', 255)->unique();
        $table->boolean('is_active')->default(true);
        $table->dateTime('invited_at')->nullable();
        $table->timestamps(); // created_at と updated_at を一括追加
    });
}

public function down()
{
    // php artisan migrate:rollback で実行される（変更を取り消す）
    Schema::dropIfExists('guest_accounts');
}
```

### 本番環境での注意点

**NOT NULLカラムを既存テーブルに追加する場合**

既存データがNULL違反になるためエラーになる。3ステップを1つのマイグレーションファイルにまとめて対処する：

```php
public function up()
{
    // Step 1: 一旦NULLで追加
    Schema::table('guest_accounts', function (Blueprint $table) {
        $table->string('new_column')->nullable();
    });

    // Step 2: 既存データを埋める
    DB::table('guest_accounts')->update(['new_column' => '']);

    // Step 3: NOT NULLに変更
    Schema::table('guest_accounts', function (Blueprint $table) {
        $table->string('new_column')->nullable(false)->change();
    });
}
```

**大量データへのインデックス追加**

MySQLはインデックス追加中にテーブル全体をロックするため、その間は書き込みが全て待ち状態になる。タイムアウトでサービスが止まる可能性がある。大量データへのインデックス追加はメンテナンス時間帯に実行する。

---

## 10. Query Builder

### Query Builderとは

Eloquentはモデルの単純なCRUDに適しているが、集計・複雑なJOIN・複数テーブルにまたがるクエリはEloquentで書くと結果がモデルと紐付きにくくなる。そういった場面で使うのがQuery Builder。

SQLの構造をそのままメソッドに置き換えた書き方で、モデルではなくテーブル名を起点にする。

```php
// Eloquent（モデルが起点）
SalesCar::where('is_active', 1)->get();

// Query Builder（テーブル名が起点）
DB::table('sales_cars')->where('is_active', 1)->get();
```

### SQLとQuery Builderの対応

```sql
SELECT sm_id, COUNT(*) as car_count
FROM   sales_cars
WHERE  is_active = 1
GROUP  BY sm_id
HAVING COUNT(*) > 3
```

```php
DB::table('sales_cars')
    ->selectRaw('sm_id, COUNT(*) as car_count')
    ->where('is_active', 1)
    ->groupBy('sm_id')
    ->having('car_count', '>', 3)
    ->get();
```

### EloquentとQuery Builderの使い分け

| | 返るもの | モデルメソッド | 向いている場面 |
|---|---|---|---|
| Eloquent | モデルインスタンス | 使える | 1テーブルのCRUD |
| Query Builder | stdClassオブジェクト | 使えない | 集計・複雑なJOIN |

Query Builderの結果はモデルではなく素の `stdClass` オブジェクト。`->name` などのプロパティアクセスはできるが、`isStopped()` などのモデルメソッドやアクセサは使えない。

---

## 11. ソフトデリート（論理削除）

### ソフトデリートとは

実際にDBからレコードを削除せず、`deleted_at` カラムに削除日時を記録することで、「削除済み」として扱う仕組み。

```
deleted_at = NULL  → 通常のレコード（クエリに含まれる）
deleted_at = 日時  → 削除済み扱い（クエリから自動除外）
```

### 実装方法

モデルに `use SoftDeletes` を1行書くだけ。

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class WishMemo extends Model
{
    use SoftDeletes;
}
```

### delete() を呼んだとき、何が起きるか

```sql
-- 物理削除（SoftDeletes なし）
DELETE FROM wish_memos WHERE id = 1;

-- 論理削除（SoftDeletes あり）
UPDATE wish_memos SET deleted_at = '2026-06-11 10:00:00' WHERE id = 1;
```

`DELETE` ではなく `UPDATE` が発行される。レコードはDBに残る。

### 特殊メソッド

```php
WishMemo::withTrashed()->get();  // 削除済み＋通常を全件取得
WishMemo::onlyTrashed()->get();  // 削除済みだけ取得
$wishMemo->restore();            // deleted_at を NULL に戻す（復元）
```

### 外部キー制約との注意点

ソフトデリートは `UPDATE` 文のため、DBの `ON DELETE CASCADE` は**発火しない**。

```
guest_accounts（deleted_at を記録するだけ）
  ↓ CASCADEは発火しない
guest_account_sales_cars（レコードが残り続ける → ズレが生じる）
```

対処はLaravelのイベントで手動処理する：

```php
protected static function booted()
{
    static::deleting(function ($account) {
        $account->salesCars()->detach(); // 中間テーブルのレコードを削除
    });
}
```

---

## 12. Observer

### Observerとは

モデルの作成・更新・削除を監視して、イベント発生時に自動で処理を実行する仕組み。

Observerはモデルに紐づくため、**どこで作成・更新・削除されても必ず発火する**。

### イベントの種類

| イベント | タイミング |
|---|---|
| `creating` / `created` | 新規作成の前/後 |
| `updating` / `updated` | 更新の前/後 |
| `saving` / `saved` | 作成・更新どちらでも前/後 |
| `deleting` / `deleted` | 削除の前/後 |

`-ing`（進行形）は保存前。`-ed`（過去形）は保存後。

```php
public function creating(GuestAccount $account)
{
    // 保存前 → データを加工できる（IDはまだない）
    $account->uuid = Str::uuid();
}

public function created(GuestAccount $account)
{
    // 保存後 → IDが確定している。外部への通知に使う
    Mail::to($account->email)->send(new WelcomeMail());
}
```

### 注意点

**バルク操作では発火しない**

```php
// これはObserverが発火しない（UPDATE文1本で全件更新するため）
GuestAccount::where('is_active', false)->update(['deleted_at' => now()]);

// これは発火する（1件ずつモデルを経由するため）
GuestAccount::where('is_active', false)->each(fn($a) => $a->delete());
```

### Observerに書くべき処理・書くべきでない処理

| Observerに向いている | サービス層に書くべき |
|---|---|
| 全ケースで必ず実行される処理 | 特定の操作・条件でだけ実行される処理 |
| ステータス変更時の更新日時自動追記 | 特定画面からだけ送るメール |
| 作成時のUUID自動セット | テストでスキップしたい処理 |
| 削除時の関連データ整理 | 呼び出し元によって挙動が変わる処理 |

---

## まとめ

- `$fillable` は入口の番人。ユーザーが入力してくるカラムだけを明示する。マスアサインメント脆弱性を防ぐ
- `$hidden` は出口の番人。JSONや配列に変換するときにパスワードなどの機密カラムを自動で除外する
- `$casts` はDBから取り出した値の型変換を保証する。PDOがstringで返す値を、指定した型に自動変換する
- スコープはwhere条件に名前をつけて再利用できるようにしたもの。再利用できることと、意図を伝えることが利点
- アクセサは取り出すときの加工をモデルが持つ。ミューテタは保存するときの加工をモデルが持つ。どちらも使う側は知らなくていい、という関心の分離を実現する
- サービスコンテナは `new` を自動でやってくれる仕組み。コントローラーがインスタンスの生成責任を持たなくて済む
- サービスプロバイダーはサービスコンテナへの特別な指示を書く場所。シングルトンなど特別な要件があるときだけ登録する
- Facadeはサービスコンテナからインスタンスを取り出す便利な窓口。気軽に使えるが、乱用すると依存関係が隠れる
- Middlewareはコントローラーの前後に割り込んでいる処理を挟む仕組み。`$next($request)` で通過、`return redirect()` で止める
- N+1問題は「1回のクエリ + データの数だけクエリ」が発行されるパフォーマンス問題。Eager Loading（`with()`）で解決する
- CollectionはEloquentで取得済みデータの入れ物。DBで絞れるものはDBに任せ、取得後の加工をCollectionで行うのが基本
- マイグレーションはDBのスキーマ変更をコードとして定義しGitで管理する仕組み。本番での`NOT NULL`カラム追加は3ステップで安全に行う
- Query BuilderはSQLに近い書き方でクエリを組み立てる仕組み。集計・複雑なJOINに向いている。結果はstdClassなのでモデルメソッドは使えない
- ソフトデリートは `deleted_at` に削除日時を記録して論理的に削除済みとして扱う仕組み。論理削除はUPDATE文のためON DELETE CASCADEは発火しない点に注意
- Observerはモデルの作成・更新・削除を監視して自動処理を実行する仕組み。全ケースで必ず実行される処理に使う。バルク操作では発火しない点に注意

---

*対話から整理。2026-06-10 / 2026-06-11*
