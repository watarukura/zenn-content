---
title: "cache:prune-stale-tagsは何をしているのか"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [laravel, php]
published: true
publication_name: "torana_tech"
---

[アップグレードガイド 10.x Laravel](https://readouble.com/laravel/10.x/ja/upgrade.html)　に記載のRedisキャッシュタグについて、Artisanコマンドで`cache:prune-stale-tags`を定期実行してください、と記載があります。  
何だこれ、ということで調べてみました。

## Laravelキャッシュファサードのタグ機能

[undocument cache tags due to complexity of implementation · laravel/docs@63dde39](https://github.com/laravel/docs/commit/63dde39470588ab1ad39cea9a592f0838ca5ad47)
Taylor御大が自らドキュメントを削除しています...。おいおい。  

タグ機能は、複数のキーに目印(タグ)をつける、というもので、まとめて削除する、みたいなときに便利な機能です。  
Laravel10へのアップグレード時に、[大きな改修](https://github.com/laravel/framework/pull/45690)が入って性能が向上したらしいです。  
その改修の影響で、タイトルの`cache:prune-stale-tags`の定期実行が必要になった模様。

## 複数のタグをつけたときの奇妙な挙動

[Flushing entries with 2 tags in RedisTaggedCache doesn't clean up properly · laravel/framework · Discussion #46106](https://github.com/laravel/framework/discussions/46106)
↑のIssueで記載がありますが、複数タグをセットしている場合に、片側でflush()を呼んだ後で、flushを読んでいない側の参照が残ってしまうようです。  
tinkerを使って調べてみました。

```php
> Cache::tags(['a', 'b'])->put(1, 10);
= true

> Cache::tags(['a', 'b'])->put(2, 11);
= true

> Cache::tags(['a', 'b'])->put(3, 12);
= true

> $redis = Cache::getRedis();
= Illuminate\Redis\RedisManager {#1686}

> $redis->keys('*')
= [
    "laravel_cache:85142c9a6bd23d13c2ff4925786ac610a07f9992:1",
    "laravel_cache:tag:a:entries",
    "laravel_cache:tag:b:entries",
    "laravel_cache:85142c9a6bd23d13c2ff4925786ac610a07f9992:2",
    "laravel_cache:85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]

> $redis->zrange('laravel_cache:tag:a:entries', 0, 10)
= [
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:1",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:2",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]

> $redis->zrange('laravel_cache:tag:b:entries', 0, 10)
= [
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:1",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:2",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]

> Cache::tags(['a'])->flush();
= true

> $redis->keys('*')
= [
    "laravel_cache:tag:b:entries",
  ]

> $redis->zrange('laravel_cache:tag:b:entries', 0, 10)
= [
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:1",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:2",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]

```

ここで、`cache:prune-stale-tags`の呼び出し元を手繰っていくと、`\Illuminate\Cache\RedisTagSet::flushStaleEntries` にたどり着きます。

```php
     /**
     * Remove the stale entries from the tag set.
     *
     * @return void
     */
    public function flushStaleEntries()
    {
        $this->store->connection()->pipeline(function ($pipe) {
            foreach ($this->tagIds() as $tagKey) {
                $pipe->zremrangebyscore($this->store->getPrefix().$tagKey, 0, Carbon::now()->getTimestamp());
            }
        });
    }
```

zremrangebyscoreは、ソート済みセットから特定の範囲のスコアを持つエントリを削除する機能です。
ゼロから実行時点のUnixtimeまで、ということですね。  
で、スコアを見てみます。

```php
> $redis->zrange('laravel_cache:tag:b:entries', 0, 10, 'withscores')
= [
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:1" => "1700631234",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:2" => "1700631265",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:3" => "1700631292",
  ]

> $redis->keys('*')
= [
    "laravel_cache:tag:a:entries",
    "laravel_cache:tag:b:entries",
    "laravel_cache:85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]
```

これで、ようやく`cache:prune-stale-tags`の挙動がわかりました。  
TTLを超過していて、かつ削除されていないエントリを削除する、というごみ掃除の機能ということです。  
...もしかして、複数タグを使ってなければ実行しなくて良いのでは？
