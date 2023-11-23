---
title: "cache:prune-stale-tagsは何をしているのか"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [laravel, php]
published: true
---

[アップグレードガイド 10.x Laravel](https://readouble.com/laravel/10.x/ja/upgrade.html)　に記載のRedisキャッシュタグについて、Artisanコマンドで`cache:prune-stale-tags`を定期実行してください、と記載があります。  
これを読み落としていて、実行しなかったらどうなるのでしょうか？  
という、失敗の記録です。

## Laravelキャッシュファサードのタグ機能

[undocument cache tags due to complexity of implementation · laravel/docs@63dde39](https://github.com/laravel/docs/commit/63dde39470588ab1ad39cea9a592f0838ca5ad47)
Taylor御大が自らドキュメントを削除しています...。おいおい。  

タグ機能は、複数のキーに目印(タグ)をつける、というもので、まとめて削除する、みたいなときに便利な機能です。  
Laravel10へのアップグレード時に、[大きな改修](https://github.com/laravel/framework/pull/45690)が入って性能が向上したらしいのですが、その改修で、タイトルの`cache:prune-stale-tags`の定期実行が必要になりました。

## 複数のタグをつけたときの奇妙な挙動

[Flushing entries with 2 tags in RedisTaggedCache doesn't clean up properly · laravel/framework · Discussion #46106](https://github.com/laravel/framework/discussions/46106)

```php
> Cache::tags(['liveId', 'channelId'])->put(1, 10, 20);
= true

> Cache::tags(['liveId', 'channelId'])->put(2, 20, 40);
= true

> Cache::tags(['liveId', 'channelId'])->put(3, 30, 60);
= true

> $redis = Cache::getRedis();
= Illuminate\Redis\RedisManager {#1686}

> $redis->zrange('laravel_cache:tag:channelId:entries', 0, 10, 'withscores')
= [
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:1" => "1700631234",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:2" => "1700631265",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:3" => "1700631292",
  ]

> $redis->keys('*')
= [
    "laravel_cache:tag:liveId:entries",
    "laravel_cache:tag:channelId:entries",
    "laravel_cache:85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]
```

```php
> Cache::tags(['liveId', 'channelId'])->put(1, 10);
= true

> Cache::tags(['liveId', 'channelId'])->put(2, 11);
= true

> Cache::tags(['liveId', 'channelId'])->put(3, 12);
= true

> clear
> Cache::tags(['liveId', 'channelId'])->put(1, 10);
= true

> Cache::tags(['liveId', 'channelId'])->put(2, 11);
= true

> Cache::tags(['liveId', 'channelId'])->put(3, 12);
= true

> $redis = Cache::getRedis();
= Illuminate\Redis\RedisManager {#1686}

> $redis->keys('*')
= [
    "laravel_cache:85142c9a6bd23d13c2ff4925786ac610a07f9992:1",
    "laravel_cache:tag:liveId:entries",
    "laravel_cache:tag:channelId:entries",
    "laravel_cache:85142c9a6bd23d13c2ff4925786ac610a07f9992:2",
    "laravel_cache:85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]

> $redis->zrange('laravel_cache:tag:liveId:entries', 0, 10)
= [
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:1",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:2",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]

> $redis->zrange('laravel_cache:tag:channelId:entries', 0, 10)
= [
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:1",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:2",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]

> Cache::tags(['liveId'])->flush();
= true

> $redis->keys('*')
= [
    "laravel_cache:tag:channelId:entries",
  ]

> $redis->zrange('laravel_cache:tag:channelId:entries', 0, 10)
= [
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:1",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:2",
    "85142c9a6bd23d13c2ff4925786ac610a07f9992:3",
  ]

```
