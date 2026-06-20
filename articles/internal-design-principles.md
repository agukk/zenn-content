---
title: "内部設計の原則"
emoji: "⚙️"
type: "tech"
topics: ["design", "backend", "database", "laravel"]
published: true
---

## 1. 内部設計とは何か

**外部設計が決めた契約を、DBとコードで実現する方法を決める工程。**

外部設計が「ユーザーにどう見せるか・どう振る舞うか（How it looks）」を決めるフェーズなら、内部設計は「それをどうやって実現するか（How it works）」を決めるフェーズ。

```
要件定義
 「ゲストが顧客登録できるようにしたい」
 ↓
外部設計
 「どの画面に・どの項目を・どんな条件で表示するか」を定義する
 ↓
内部設計
 「どのテーブルで・どのクラスで・どう処理して実現するか」を決める
 ↓
実装
```

内部設計が決める主な対象は以下の3つ。

- **コードの構造** — どの層に何を書くか
- **DBの設計** — テーブルやカラムをどう持つか
- **処理の設計** — トランザクション・認可・エラー処理をどう実現するか

---

## 2. 外部設計との関係

外部設計が内部設計に渡すのは「**外側から見た振る舞いの契約**」。内部設計はその契約を守りながら、実現方法を自由に選べる。

### 具体例: 「自分が登録した顧客のみ編集できること」

外部設計の受け入れ条件にこうあった。

> 自分が登録した顧客とそれ以外の顧客で、編集可否を区別できること

この契約を受け取った内部設計は、2段階で実現方法を決めた。

**① DBの設計**
「誰が登録したか」をデータとして持つために、`customers` テーブルに `created_by_guest_account_id` カラムを追加する。

**② コードの設計**
`CustomerPolicy::update()` でそのカラムとログイン中のゲストIDを比較して判定する。

```php
// CustomerPolicy
public function update($user, Customer $customer)
{
    if ($user instanceof GuestAccount) {
        return (int) $customer->created_by_guest_account_id === (int) $user->id;
    }
    // ...
}
```

外部設計が「何を実現するか」を決め、内部設計が「どうやって実現するか」を決める。外部設計の契約が明確であるほど、内部設計の判断がしやすくなる。

---

## 3. 根本原則: 不要な複雑さを作らない

内部設計のあらゆる判断の根底にある原則は一つ。

> **変更するとき、考えることがシンプルになる状態を作ること。**

これが「保守性が高い」の中身。保守性が高いとは、将来の変更や拡張が容易にできることであり、そのためには不要な複雑さを取り除いておく必要がある。

複雑さが増すと何が起きるか。

| 状態 | 変更時に何が起きるか |
|---|---|
| シンプル | 変更したい箇所だけ考えればいい |
| 複雑 | 関係ない部分まで理解してから変更しなければならない |
| 複雑（混在） | 変更の影響が予測しにくく、意図しない場所が壊れるリスクがある |

---

## 4. 関心の分離

不要な複雑さを作らないための最も基本的な手段が「**関心の分離**」。

**あるべき場所に書く。1つのものが1つの役割だけを持つ。**

関心の分離がもたらすのは2つの効果。

**① 探しやすい**
変更したいものがどこにあるか迷わない。メールのロジックを変えたければサービス層を開けばいい。

**② 壊しにくい**
変更の影響範囲が限定される。メール文面を変えようとしたとき、同じファイルに無関係な処理が混在していると、うっかりその処理の閉じ括弧を消してしまうだけで別の機能が壊れる。あるべき場所に分離されていれば、壊せるものが少ない。

### コードとDBの両方に現れる原則

関心の分離はコードの構造だけでなく、DBの設計にも現れる。

| レベル | 混在している状態 | 分離された状態 |
|---|---|---|
| コード | コントローラーにビジネスロジックが混在 | サービス層に分離されている |
| DB | 同じ事実を複数のカラムで持っている | 1つの事実は1か所のカラムだけが持つ |

---

## 5. コード層の設計

Laravelでは、責務ごとに層を分けることが保守性の基盤になる。

| 層 | 役割 | 書くもの |
|---|---|---|
| Controller | HTTPの受け渡し | リクエストの受け取り・レスポンスの返却・次の画面へのリダイレクト |
| Service | ビジネスロジック | 業務上の判断・複数モデルにまたがる処理・メール送信の判断 |
| Repository | DBとのやりとり | クエリの構築・データの取得・保存 |

### 混在するとどうなるか

コントローラーにビジネスロジックが混在すると：

- メールの仕様を変えたいとき、HTTPの処理が書かれたファイルを触ることになる
- 関係ない処理を誤って変更するリスクが生まれる
- 同じロジックを別の場所から呼び出したいときに再利用できない

### 判断の基準

「これはどの層に書くか」で迷ったときの判断基準はシンプル。

> **そのコードが担っている「役割」は何か。**

HTTPを受け取って返すことが役割ならController、業務上の判断が役割ならService、DBを操作することが役割ならRepository。

### Repositoryを独立させる3つの理由

**① 修正が1か所で済む**

同じクエリを複数か所に直書きされていると、仕様変更のたびに全か所を探して直さなければならない。1か所修正を忘れると「ある画面では新仕様、別の画面では旧仕様」という矛盾が生まれる。Repositoryにまとめておけば変更は1か所だけ。

```php
// CustomerRepository
public function findByGuestId(int $salesCarId, int $guestId): Collection
{
    return Customer::ofSalesCar($salesCarId)
        ->where('created_by_guest_account_id', $guestId)
        ->whereNull('deleted_at')  // ←仕様追加されたらここだけ直せばいい
        ->orderBy('created_at', 'desc')
        ->get();
}
```

**② HTTPに依存せず、再利用できる**

ControllerにSQLを直書きすると、そのロジックはHTTPリクエスト経由でしか呼べない。ArtisanコマンドからはControllerは呼べないため、同じロジックをコピーしなければならなくなる。ServiceからRepositoryを呼ぶ構造にすれば、どの入口からでも同じRepositoryを使い回せる。

```
Controller（HTTP） ───┐
                      ├──▶ CustomerService ▶ CustomerRepository（DB）
Artisan Command ──────┘
```

**③ テストができる（DIで偽物に差し替えられる）**

ServiceとControllerがRepositoryをコンストラクタで受け取る構造（DI）になっていると、テスト時に、本物のRepositoryではなくDBに触らない偽物に差し替えられる。

```php
// テスト用の偽Repository（DBに触らず決まったデータを返すだけ）
$fakeRepo = new class {
    public function findByGuestId(int $salesCarId, int $guestId): Collection {
        return collect([(object)['name' => 'テスト太郎', 'status' => 1]]);
    }
};

$service = new CustomerService($fakeRepo);  // 偽物を注入
```

これにより、DBを用意しなくてもビジネスロジックだけをテストできる状態になる。Controllerに直書きされていると差し替えができないため、テストのたびにDBへの実データ挿入が必要になる。

### Repositoryが必要な場面・不要な場面

| 場面 | 判断 |
|---|---|
| 同じクエリが複数か所から呼ばれる（または呼ばれる可能性が高い） | 作るべき |
| クエリロジックが複雑でServiceに混ぜたくない | 作るべき |
| テストでDBを差し替えたい | 作るべき |
| 1か所しか呼ばれないシンプルなクエリ | 不要なことも |
| 使い捨てのスクリプト・プロトタイプ | 不要 |

---

## 6. 依存性注入（DI）

### DIとは何か

クラスが必要とする他のクラス（依存物）を、**内部で `new` するのではなく外から渡してもらう**設計パターン。

- **Dependency（依存性）** = そのクラスが必要としているもの
- **Injection（注入）** = それを外から渡すこと

### 内部で `new` するとどう困るか

```php
// 問題のある書き方（内部で new している）
public function hasInvalidSelectedSalesCarIds(array $selectedIds, ?int $agencyId): bool
{
    $repository = new GuestAccountRepository();  // ←外から差し替えられない
    // ...
}
```

- テストで偽物（モック）に差し替えられない → DB必須になる
- 実装を変えたいとき（DB→Redisなど）、このクラスも修正が必要になる

### コンストラクタ注入（推奨）

```php
class GuestAccountService
{
    private $repository;

    public function __construct(GuestAccountRepository $repository)
    {
        $this->repository = $repository;  // 外から受け取って保存
    }

    public function hasInvalidSelectedSalesCarIds(array $selectedIds, ?int $agencyId): bool
    {
        $selectableSalesCarIds = $this->repository->getSelectableSalesCars($agencyId)
                                                  ->pluck('id')->all();
        return !empty(array_diff($selectedIds, $selectableSalesCarIds));
    }
}
```

コンストラクタで受け取ることで、`new GuestAccountService($repository)` した瞬間から、依存物が確実に存在する。渡し忘れる構造上できない。

### 注入の3種類と使い分け

| 種類 | 書き方 | いつ使う |
|---|---|---|
| **コンストラクタ注入** | `__construct` の引数 | クラス全体で常に必要なもの（主役） |
| **メソッド注入** | メソッドの引数 | その呼び出しの瞬間だけ必要なもの |
| **セッター注入** | `setXxx()` メソッド | デバッグ用・オプションなもの |

### Laravelのサービスコンテナ

型ヒントに書かれたクラス名を見て、自動でインスタンスを作って渡してくれる仕組み。おかげで引数に依存関係を全部解決してくれる。

```
// サービスコンテナがない場合、手動で連鎖的にnewしなければならない
$repository = new GuestAccountRepository();
$service    = new GuestAccountService($repository);
$controller = new GuestAccountController($service);
```

具体的なクラスを型ヒントに書けば登録不要で自動解決される。インターフェースを使う場合のみ `AppServiceProvider` での登録が必要。

### DIを使う場面・使わない場面

> **他のクラスに依存するかどうかが、判断の鍵。**

| 場面 | 判断 |
|---|---|
| Service・Repository など、他クラスが必要なロジックを持つクラス | DI |
| Eloquentモデル（`GuestAccount` など）、データを入れる箱として使うクラス | 都度 `new` |

### 依存性逆転の原則（SOLID の D）との関係

通常の依存方向：上位（Service）→ 下位（具体的なRepositoryクラス）に依存。

インターフェースを介すことで：

```
GuestAccountService（上位） → GuestAccountRepositoryInterface（抽象）
                                          ↑ implements
                               GuestAccountRepository（下位・具体クラス）
```

下位の具体クラスが上位の決めたインターフェースに従う形になり、**依存の向きが逆転する**。これを「依存性逆転の原則（DIP）」と呼ぶ。

---

## 7. DB設計の原則

### 単一の情報源

同じ事実はDBの1か所だけに持つ。

`guest_accounts` テーブルに `status` カラムと `stopped_at` カラムの両方を持つ設計を例に考える。

| カラム | 内容 |
|---|---|
| `status` | `'stopped'` などの固定値 |
| `stopped_at` | 停止した時刻 |

どちらも「停止しているかどうか」という同じ事実を表している。停止処理を実装するとき、両方を更新し忘れると矛盾が生まれる。「`stopped_at` はセットされているのに `status` は `'active'` のまま」という状態では、どちらが正しいか判断できない。

`status` カラムを持たず、`stopped_at` に値があるかどうかだけで判定することで、同じ事実は1か所に集約できる。

### 状態は導出する

状態（招待中・有効・停止済みなど）は、固定値のカラムではなく複数のカラムから導出する設計にする。

```php
// GuestAccount モデル
public function isStopped(): bool
{
    return !empty($this->stopped_at);
}

public function isExpired(): bool
{
    return !empty($this->end_date) && $this->end_date->endOfDay()->isPast();
}

public function getStatusLabelAttribute(): string
{
    if ($this->isStopped()) return '停止済み';
    if ($this->isExpired()) return '期限切れ';
    // ...
}
```

状態を固定値カラムで持つと「カラムの更新漏れ」という複雑さが生まれる。導出する設計にすることで、更新する場所を最小限にできる。

---

## 8. トランザクション設計

複数のテーブルにまたがる操作は、**原子性**を保証する必要がある。

原子性とは「全部成功するか、全部なかったことにするか」のどちらかしかない状態。

### なぜ必要か

ゲストアカウント発行処理では、2つのテーブルに書き込む必要があった。

1. `guest_accounts` にアカウントを保存する
2. `guest_account_sales_cars` に担当号車を紐づける

この2つをトランザクションでまとめない場合、`guest_accounts` の保存は成功したが `guest_account_sales_cars` の保存でエラーが起きると、**担当号車がないゲストアカウントがDBに残る**。そのゲストがログインしても業務が開始できない中途半端な状態になる。

```php
DB::beginTransaction();
try {
    $guestAccount->save();                           // guest_accounts
    $guestAccount->salesCars()->sync($salesCarIds);  // guest_account_sales_cars
    DB::commit();
} catch (\Exception $exception) {
    DB::rollBack();
}
```

### トランザクションが必要な判断基準

> **複数のテーブルへの書き込みが、一連の操作として意味を持つ場合。**

どちらか一方だけ成功した状態がビジネス上ありえない場合は、トランザクションでまとめる。

---

## 9. ドメイン知識の役割

内部設計ではドメイン知識を「**コードの構造がビジネスの概念を正しく反映しているか**」の判断に使う。

要件定義でドメイン知識を「暗黙の前提を引き出すため」に使うのとは、使い方が異なる。

### 判断の例

どのポータルに機能を置くかの検討で、2つの案があった。

| 案 | 内容 | 実装コスト |
|---|---|---|
| 案A | 既存の `sales_partner` ポータルにゲストを寄せる | 低い |
| 案B | ゲスト専用の構造を作る | 高い |

案Aは実装コストが低く、実装のしやすさで言えば有利だった。しかし「ゲストは販売パートナーの一種ではなく、権限が強く制限された別の主体」というドメイン知識があったため、案Bを選んだ。

### なぜコードの構造がビジネスの概念を反映すべきか

コードの構造とビジネスの概念がずれていると、将来の変更時に「ゲストの制限を変えたいのに、販売パートナーのロジックも考慮しなければならない」という状況が生まれる。

> コードの構造がビジネスの概念を正しく反映していると、変更したい概念のコードだけを変えれば済む。

技術的に正しい実装より、ビジネスとして正しい構造の方が、長期的な保守性を生む。

---

## 10. 設計原則の名前

今日の対話で導き出した考え方には、業界で広く使われている名前がある。

| 対話で出てきた言葉 | 原則の名前 |
|---|---|
| 不要な複雑さを作らない | **KISS**（Keep It Simple, Stupid） |
| あるべき場所に書く・1つの役割だけ持つ | **SRP**（Single Responsibility Principle） |
| 今必要ないものを作らない | **YAGNI**（You Aren't Gonna Need It） |

名前を知る前に本質を理解していたということ。原則の名前は、他のエンジニアと話すときの共通言語として機能する。

### YAGNIについて

今必要ないものを作ると2つの問題が起きる。

- **スコープが際限なく広がる** — あれもこれもと増えてキリがなくなる
- **コードが不必要に複雑になる** — 使われない抽象化や設計が保守の邪魔になる

### SOLIDについて

SRPはSOLIDという5つの原則の頭文字の「S」にあたる。

| 頭文字 | 原則名 | 一言 |
|---|---|---|
| **S** | Single Responsibility Principle（単一責任の原則） | 1つのクラスは1つの責任だけ持つ |
| **O** | Open/Closed Principle（開放閉鎖の原則） | 新機能はコードを追加して実現し、既存コードを触らない |
| **L** | Liskov Substitution Principle（リスコフの置換原則） | サブクラスは親の代わりに使われても呼び出し側の期待を裏切らない |
| **I** | Interface Segregation Principle（インターフェース分離の原則） | 使わないメソッドの実装を強制しない。インターフェースは小さく |
| **D** | Dependency Inversion Principle（依存性逆転の原則） | 上位が下位の具体クラスではなく抽象（インターフェース）に依存する |

---

### O: 開放閉鎖の原則（Open/Closed Principle）

> **拡張に対して開いており、修正に対して閉じている。**

新しい機能を追加するとき、既存の動いているコードを変更しなくても済む構造にする。

**違反の例 — ステータスラベルのif文**

```php
public function getStatusLabelAttribute(): string
{
    if ($this->isStopped()) return '停止済み';
    if ($this->isExpired()) return '期限切れ';
    // 新ステータスを追加するたびに、ここを変更 → 既存コードを壊すリスク
}
```

**準拠の例 — メール送信処理**

```php
// MailUtil は変更せず、新しいMailableクラスを追加するだけで拡張できる
MailUtil::sendMailAndCheckDelivery($email, new GuestAccountResumed($account), $portal);
```

`MailUtil` は一切変更せず、`GuestAccountResumed` というクラスを追加するだけで新しいメール種別に対応できる。

**適用の判断基準**

原則は常に適用すべきではない。**変更が繰り返し起きる箇所**に絞る。ステータスが5・6種類で増える見込みがなければifのままでもいい（YAGNI）。

---

### L: リスコフの置換原則（Liskov Substitution Principle）

> **サブクラスは、親クラスの代わりに使われても、呼び出し側の期待を裏切ってはいけない。**

**違反の例**

```php
class TrialGuestAccount extends GuestAccount
{
    public function isExpired(): bool
    {
        return false;  // 常にfalse → 「期限切れでも通知が飛ばなければならない」が起きる
    }
}

// 呼び出し側は「期限切れなら通知を飛ばす」と信じている。裏切られる。
$service->sendExpiryWarning(new TrialGuestAccount());
```

**修正の方針**

LSP違反が起きるとき、本当の問題は継承関係そのものが間違っていることが多い。親クラスの全メソッドが期待通りに動かないような継承をするくらいなら、別クラスにする。

---

### I: インターフェース分離の原則（Interface Segregation Principle）

> **使わないメソッドへの依存を強制してはいけない。**

インターフェースに全メソッドを詰め込むと、実装クラスは不要なメソッドを全て実装しなければならない。インターフェースは役割ごとに小さく分ける。

**準拠の例**

```php
// 役割ごとに分割する
interface GuestAccountReaderInterface
{
    public function findById(int $id): ?GuestAccount;
}

interface GuestAccountWriterInterface
{
    public function save(GuestAccount $account): void;
}
// 各クラスは必要なインターフェースだけを実装する
```

インターフェース自体のYAGNI判断が重要。実装が1つだけで差し替える見込みがなければ作らなくていい。

---

### D: 依存性逆転の原則（Dependency Inversion Principle）

> **上位モジュールは下位の具体実装に依存してはいけない。両方が抽象（インターフェース）に依存すべき。**

DI（依存性注入）と組み合わせることで真価を発揮する。詳細は「6. 依存性注入（DI）」を参照。

**準拠の例**

```php
// インターフェース（抽象）に依存する
public function __construct(GuestAccountRepositoryInterface $repository)
```

```
GuestAccountService（上位） → 依存 → GuestAccountRepositoryInterface（抽象）
                                                  ↑ implements
                                     GuestAccountRepository（下位・具体クラス）
```

下位の具体クラスが上位の決めたインターフェースに従う形になり、**依存の向きが逆転する**。主導権が上位に移ったことで、下位の実装を差し替えても上位は影響を受けない。

---

## まとめ

- 内部設計の本質は「外部設計の契約を守りながら、DBとコードで実現方法を決めること」。契約が明確であるほど、実現方法の判断がしやすくなる。
- 根本原則は「不要な複雑さを作らない」こと。保守性が高いとは「変更するとき、考えることがシンプルになる」状態を維持できていること。
- 関心の分離（あるべき場所に書く）は、コードの層にもDBの設計にも現れる普遍的な原則。探しやすさと壊しにくさを生む。
- コードの層はController・Service・Repositoryそれぞれの役割を明確に分ける。混在すると変更の影響が読めなくなる。
- Repositoryを独立させる理由は3つ: ①修正が1か所で済む（散らばり防止）、②HTTP非依存で再利用できる、③DIで偽物に差し替えてテストができる。
- 依存性注入（DI）とは、クラスが必要とする他のクラスを内部で `new` せず外から渡してもらう設計。コンストラクタ注入が基本。他のクラスに依存するロジックはDI、データを入れる箱（Eloquentモデルなど）は都度 `new`。
- DB設計では同じ事実を1か所だけに持つ（単一の情報源）。状態は固定値カラムより導出する設計の方が更新漏れを防げる。
- 複数テーブルにまたがる操作はトランザクションで原子性を保証する。中途半端な状態をDBに残さない。
- ドメイン知識は「コードの構造がビジネスの概念を正しく反映しているか」の判断に使う。技術的に正しい実装より、ビジネスとして正しい構造が長期的な保守性を生む。
- 今日導き出した原則には名前がある: KISS・SRP・YAGNI・SOLID。名前は他のエンジニアと話すときの共通言語になる。
- SOLIDの残り4原則: O（変更せず拡張で新機能を追加）・L（サブクラスは親の約束を守る）・I（インターフェースは役割ごとに小さく分ける）・D（具体クラスではなく抽象に依存する）。それぞれ常に適用すべきではなく、適用すべき場面を判断して使う。

---

*対話から整理。最終更新 2026-06-13（リポジトリパターン・依存性注入・SOLID O/L/I/D 追記）*
