---
title: "Redisをざっくり理解する"
emoji: "⚡"
type: "tech"
topics: ["redis", "cache", "laravel", "php"]
published: true
---

## 一言で言うと

RAM にデータを一時保存することで DB クエリの回数を減らし、アプリのパフォーマンスを上げる仕組み。

## 問い（復習時にここだけ見て答える）

- Redis が MySQL より速い理由は？
- Cache Hit / Cache Miss とは何か？
- TTL が必要な理由を2つ言えるか？
- キャッシュを使うべきでないデータの条件は？

## ノート

### Redisとは
- Remote Dictionary Server の略
- キャッシュ（一時的なデータ）を RAM に保存するインメモリデータベース

### なぜ速いのか

| 種類 | 保存場所 | 特徴 |
|------|---------|------|
| MySQL | HDD/SSD | 電源を切っても残る。遅い |
| Redis | RAM | 電源を切ると消える。速い |

### Cache Hit / Cache Miss

```
リクエスト
    ↓
Redis を確認
    ├── データあり（Cache Hit）  → Redis から返す ✅ 速い
    └── データなし（Cache Miss） → DB から取得
                                    ↓
                                Redis に保存（次回から Hit する）
                                    ↓
                                レスポンスを返す
```

### TTL（Time To Live）

保存時に「何秒後に自動で消す」という期限を設定できる。

**必要な理由：**
1. RAM の容量節約
2. データの鮮度維持（DB が更新されても古いキャッシュが残り続けるのを防ぐ）

### いつ使う・使わないか

| 向いている | 向いていない |
|-----------|------------|
| あまり変わらないデータ（商品マスタ、顧客基本情報など） | 頻繁に更新されるデータ（在庫数、リアルタイム価格など） |
| 何度も同じクエリが走るデータ | 機密性の高いデータ（パスワード、決済情報など） |

### Laravelでの使われ方の例

```php
// 保存（TTL付き）
protected function setRedis($id, $data, $period = null)
{
    Redis::set($id, $data, 'EX', $period);
}

// 取得（あればデータ、なければ null）
protected function returnRedis($id)
{
    $redis_data = $this->getRedis($id);
    if (!is_null($redis_data)) {
        return $redis_data; // Cache Hit
    }
    // null を返す = Cache Miss
}
```

## 自分の言葉で説明する

Redis は「よく使うデータを手元に置いておく棚」のようなもの。毎回倉庫（DB）まで取りに行く代わりに、棚（RAM）から即座に取り出せるので速い。ただし棚は狭いので、古くなったものは定期的に捨てる（TTL）。

## なぜそうなっているのか（Why）

RAM は CPU に近い位置にあるため物理的にアクセスが速い。HDD/SSD は大容量だが、アクセス時間がかかる。Redis はこの物理的な速度差を利用している。

## 詰まったポイント・誤解していたこと

- ファイルキャッシュ・DBキャッシュも「キャッシュ」と呼ばれるが、結局 HDD/SSD へのアクセスが発生するため Redis より遅い。名前だけで同じ速さだと思わないこと。
