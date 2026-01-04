## はじめに

ご無沙汰しております。LRMでエンジニアをしている大場です。

先日、弊社が開発しているセキュリティ教育SaaS「セキュリオ」のRailsアプリケーションにYJITを導入し、本番環境での運用を開始しました。結果として大きくパフォーマンスを向上させることに成功しましたので、YJIT導入手順とその結果についての記事をお届けします。

YJIT導入時のRuby、Railsのバージョンは以下の通りです。
・Ruby 3.4.1
・Rails 7.1

## 概要

YJITはJITコンパイラの一種で、コード実行の速度向上を図るものです。下記の記事でも触れているので参考にして頂ければ幸いです。

https://zenn.dev/lrm/articles/5ab63eff5360ab

YJITの導入事例はここ数年で増えています。調査した内容は参考文献に記載しました。

導入結果に関して調査したところ、ほとんどの企業でパフォーマンスが向上していました。しかし、導入前後のサーバーのリソース消費量変化には結果にばらつきがありました。

多くの場合、メモリ消費量が若干増加していましたが、著しく増加した例もあり、中にはむしろメモリ使用量が減少したという例もありました。導入にはサーバーのリソース消費に配慮が必要そうです。

今回は、リソース消費に配慮した導入手順と具体的な取り組み、実際の導入結果に触れます。

## 作業手順

まずは、`config/initializers/yjit.rb`に以下の変更を加えました。

```config/initializers/yjit.rb
if defined? RubyVM::YJIT.enable
  Rails.application.config.after_initialize do
    RubyVM::YJIT.enable(stats: true)
  end
end
```

YJITのスタッツに関するログ出力が不要であれば、`(stats:true)`は不要です。今回はローカル環境、検証環境、本番環境全てでログを確認するため、一時的に付与しました。

実際のログ出力処理は`application_controller.rb`に追記しました。

```application_controller.rb
class ApplicationController < ActionController::Base
  before_action :logging_yjit_stats

・・・　　省略　　　・・・

  def logging_yjit_stats
    return if defined?(RubyVM::YJIT).nil? || !RubyVM::YJIT.enabled?

    # TODO: 本番環境では1/5000の確率でYJITの統計情報をログに出力
    # return unless Random.rand(5_000) < 1

    stats = RubyVM::YJIT.runtime_stats
    code_region_size, yjit_alloc_size, ratio_in_yjit = stats.values_at(:code_region_size, :yjit_alloc_size, :ratio_in_yjit)
    total_memory_usage = code_region_size + yjit_alloc_size
    Rails.logger.info(" [YJIT]: code_region_size: #{code_region_size}, yjit_alloc_size: #{yjit_alloc_size}, ratio_in_yjit: #{ratio_in_yjit}")
    Rails.logger.info(" [YJIT]: total memory usage: #{total_memory_usage.to_f / 1024 / 1024} MB")
  end
end
```

変更を加えた後にローカルで動作確認を行い、出力されたYJITのスタッツを確認しました。

```bash
I, [2025-06-09T11:59:37.012105 #7]  INFO -- : [192.168.65.1]  [YJIT]: code_region_size: 46084096, yjit_alloc_size: 40567943, ratio_in_yjit: 97.02192876498957
I, [2025-06-09T11:59:37.012992 #7]  INFO -- : [192.168.65.1] total memory usage:85.64040088653564 MB
```

JIT領域のコミッターである[k0kubunさんのブログ](https://k0kubun.hatenablog.com/entry/ruby-3-3-yjit#:~:text=%E3%81%BE%E3%81%9F%E3%80%81RubyVM%3A%3AYJIT.runtime_stats%5B%3Aratio_in_yjit%5D%20%E3%81%8C%20%2D%2Dyjit%2Dstats%20%E6%99%82%E3%81%AB%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88%E3%81%AE%E3%83%93%E3%83%AB%E3%83%89%E3%81%A7%E3%82%82%E6%8F%90%E4%BE%9B%E3%81%95%E3%82%8C%E3%82%8B%E3%82%88%E3%81%86%E3%81%AB%E3%81%AA%E3%81%A3%E3%81%9F%E3%80%82%20%E3%81%93%E3%82%8C%E3%81%AFRuby%20VM%E4%B8%8A%E3%81%A7%E5%AE%9F%E8%A1%8C%E3%81%95%E3%82%8C%E3%82%8B%E5%91%BD%E4%BB%A4%E3%81%AE%E3%81%86%E3%81%A1%E4%BD%95%25%E3%81%8CYJIT%E3%81%A7%E5%AE%9F%E8%A1%8C%E3%81%95%E3%82%8C%E3%81%9F%E3%81%8B%E3%82%92%E7%A4%BA%E3%81%99%E3%82%82%E3%81%AE%E3%81%A7%E3%80%81%20%E7%90%86%E6%83%B3%E7%9A%84%E3%81%AB%E3%81%AF%E6%9C%80%E5%A4%A799%25%E3%81%8F%E3%82%89%E3%81%84%E3%81%8C%E6%9C%9B%E3%81%BE%E3%81%97%E3%81%84%E3%81%8C%E3%80%81%E3%82%A2%E3%83%97%E3%83%AA%E3%81%AB%E3%82%88%E3%81%A3%E3%81%A6%E3%81%AF%E3%81%93%E3%82%8C%E3%81%8C%E5%B9%B3%E5%9D%8790%25%E3%81%A8%E3%81%8B%E3%81%A7%E3%82%8218%25%E9%AB%98%E9%80%9F%E5%8C%96%E3%81%97%E3%81%9F%E3%82%8A%E3%81%99%E3%82%8B%E3%80%82%20%E9%80%9F%E5%BA%A6%E3%82%92%E5%A6%A5%E5%8D%94%E3%81%97%E3%81%A6%20%2D%2Dyjit%2Dexec%2Dmem%2Dsize%20%E3%82%92%E4%B8%8B%E3%81%92%E3%82%8B%E5%A0%B4%E5%90%88%E3%81%AF%E3%80%81%E3%81%93%E3%82%8C%E3%81%8C%E3%81%82%E3%81%BE%E3%82%8A%E4%B8%8B%E3%81%8C%E3%82%8A%E9%81%8E%E3%81%8E%E3%81%AA%E3%81%84%E3%82%88%E3%81%86%E3%81%AB%E6%B0%97%E3%82%92%E3%81%A4%E3%81%91%E3%82%8B%E3%81%A8%E8%89%AF%E3%81%84*1%E3%80%82)や[YJITのREADME](https://docs.ruby-lang.org/en/3.4/yjit/yjit_md.html)によると、YJITのカバー率`ratio_in_yjit`は99%以上が推奨とのことでした。ローカルでの動作確認時では1000~1500リクエスト近くを処理させて97%でした。

97%到達時点でメモリ使用量は85MB程度だったため[YJITに割り当てるメモリ使用量を指定できる`--yjit-exec-mem-size`](https://docs.ruby-lang.org/en/3.4/yjit/yjit_md.html#:~:text=%2D%2Dyjit%2Dmem%2Dsize%3DN%3A%20soft%20limit%20on%20YJIT%20memory%20usage%20in%20MiB%20(default%3A%20128).%20Tries%20to%20limit%20code_region_size%20%2B%20yjit_alloc_size)はデフォルトの128MiBのままで、カバー率は理想値の99%に到達すると判断しました。

次に、インフラ側のリソース使用状況も確認しました。

調査時点では2GB割り当てたうちの35%である700MB程度を使用しており、未使用のメモリが1300MB程度ありました。このことから、YJITが追加で128MiB(134MB)を消費しても問題ないと判断しました。導入後のメモリ使用率は`134÷2000=0.067`の式より約7%増加する予想を立てました。

![](https://storage.googleapis.com/zenn-user-upload/779b8830bacc-20250709.png)

また、慎重を期すためにRailsサーバーのプロセスを実行しているコンテナのリソース消費を直接確認しました。具体的には、ECSタスクによって起動した`rails server`のプロセスを実行中のコンテナ内で`free -m`コマンドによる確認と`/sys/fs/cgroup/memory/memory.stat`の確認を行いました。

```bash
$ aws ecs execute-command --cluster prd-seculio --task <タスクID> --container rails-puma --interactive --command "free -m"

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.


Starting session with SessionId: ecs-execute-command-aiqiiljysnr4qddloj5k5arpny
               total        used        free      shared  buff/cache   available
Mem:            3850        1274         127           0        2770        2615
Swap:              0           0           0
```

```bash
{
  "read": "2025-06-10T04:41:26.290527762Z",
  "preread": "2025-06-10T04:41:16.290947033Z",
  "pids_stats": {},

　　　　・・・　　　省略　　　・・・

  "memory_stats": {
    "usage": 710815744,
    "max_usage": 711073792,
    "stats": {
      "active_anon": 0,
      "active_file": 10121216,
      "cache": 44199936,
      "dirty": 0,
      "hierarchical_memory_limit": 2147483648,
      "hierarchical_memsw_limit": 9223372036854772000,
      "inactive_anon": 635019264,
      "inactive_file": 34263040,
      "mapped_file": 4866048,
      "pgfault": 369303,
      "pgmajfault": 66,
      "pgpgin": 299838,
      "pgpgout": 133938,
      "rss": 635019264,
      "rss_huge": 0,
      "total_active_anon": 0,
      "total_active_file": 10121216,
      "total_cache": 44199936,
      "total_dirty": 0,
      "total_inactive_anon": 635019264,
      "total_inactive_file": 34263040,
      "total_mapped_file": 4866048,
      "total_pgfault": 369303,
      "total_pgmajfault": 66,
      "total_pgpgin": 299838,
      "total_pgpgout": 133938,
      "total_rss": 635019264,
      "total_rss_huge": 0,
      "total_unevictable": 0,
      "total_writeback": 0,
      "unevictable": 0,
      "writeback": 0
    },
    "limit": 9223372036854772000
  },
　　　・・・　　　省略　　　・・・
```

1つ目の`free -m`実行結果は`1274(userd) / 3854(total) =0.331`となりました。メモリ消費率が約33%となっていることがわかります。

2つ目の`/sys/fs/cgroup/memory/memory.stat`の値も`712964571(usage) / 2147483648(hierarchical_memory_limit) =0.332`で約33%であり、
実際の測定でもメモリ使用率はAWS上のコンソールとほぼ同じ値が得られました。

コンソール上はスケールアウトした後の全コンテナの平均値が表示されています。そのため1つのコンテナ内で取得したメモリ使用率とコンソール上で閲覧できるメモリ使用率に若干の差があります。調査時は4台までスケールアウトしていたため4台全てでメモリ使用率を測定し、コンソール上の平均値とほぼ同値であることを確認しました。

上記変更と調査を行い、実際にYJITでの本番環境運用を開始しました。

## 運用結果

### レスポンス速度

導入前2週間と導入後2週間でのレスポンス速度を計測し、以下の通りになりました。

![](https://storage.googleapis.com/zenn-user-upload/2aa85b4a0ad2-20250708.png)

![](https://storage.googleapis.com/zenn-user-upload/b9dedf206429-20250708.png)

1枚目のグラフがp90(90th percentile)で、2枚目がp50(50th percentile)です。どちらも濃く着色された折れ線が導入後の数値になります。

| | 導入前 | 導入後 |
| ---- | ---- | ---- |
| p90 | 98.2(ms) | 81.5(ms) |
| p50 | 18.8(ms) | 16.5(ms) |

p90の平均値は、導入前が平均98.2(ms)だったのに対して、導入後は81.5(ms)となりました。
割合にして約20%近くレスポンス速度が向上しています。また、全日で導入後の方がレスポンス速度が早く、顕著に差が出ました。

p50の平均値は導入前が18.8(ms)で、導入後が16.5(ms)となりました。こちらも約13%程レスポンス速度が向上しています。

実際に、ホーム画面のレスポンス速度は下記グラフの通りに変化しました。

![](https://storage.googleapis.com/zenn-user-upload/2f7a8596d480-20250711.png)

6月18日のYJIT導入を境にレスポンス速度が改善されていることがわかります。

### YJITカバー率

メモリ使用量100MBほどで、YJITのカバー率は目標としていた99%に到達していました。
導入前の試算から乖離することなく、狙い通りの挙動になりました。

```bash
I, [2025-06-18T17:17:03.636129 #8]  INFO -- : [YJIT]: code_region_size: 31096832, yjit_alloc_size: 52412181, ratio_in_yjit: 98.34052255946375
I, [2025-06-18T17:17:03.637203 #8]  INFO -- : [YJIT]: total memory usage: 79.64040088653564 MB
I, [2025-06-18T17:34:11.245573 #8]  INFO -- : [YJIT]: code_region_size: 33193984, yjit_alloc_size: 55700037, ratio_in_yjit: 98.54103535795394
I, [2025-06-18T17:34:11.245777 #8]  INFO -- : [YJIT]: total memory usage: 84.77594470977783 MB
I, [2025-06-18T19:19:13.931748 #8]  INFO -- : [YJIT]: code_region_size: 37072896, yjit_alloc_size: 62140809, ratio_in_yjit: 99.08002440102673
I, [2025-06-18T19:19:13.931854 #8]  INFO -- : [YJIT]: total memory usage: 94.61756229400635 MB
I, [2025-06-19T03:00:10.293765 #8]  INFO -- : [YJIT]: code_region_size: 38858752, yjit_alloc_size: 64812521, ratio_in_yjit: 99.1960748272391
I, [2025-06-19T03:00:10.293938 #8]  INFO -- : [YJIT]: total memory usage: 98.86863040924072 MB
I, [2025-06-19T08:30:29.143663 #8]  INFO -- : [YJIT]: code_region_size: 39399424, yjit_alloc_size: 65636670, ratio_in_yjit: 99.21875472489396
I, [2025-06-19T08:30:29.143763 #8]  INFO -- : [YJIT]: total memory usage: 100.17022514343262 MB
```

### メモリ使用率

6月18日の17時に変更を入れ、メモリ消費率は7~8%増加しました。導入前は30%台前半だったメモリ消費率が、導入後は40%前後で推移しています。

![](https://storage.googleapis.com/zenn-user-upload/b85eb1a25f9a-20250711.png)

19日~22日で一時的にメモリ消費率が急増していますが、該当日にアプリケーション内で非常に重い処理が繰り返し実行された履歴をNew Relic上で確認しています。このメモリ消費率の急増はYJITの導入起因ではありませんでした。

YJIT導入によるメモリ消費率の増加は、概ね事前の見積もりに準じた結果となりました。

## おわりに

調査段階ではメモリ消費量の急増など、いくつか懸念点があったためかなり慎重に作業を進めました。
良い結果となって非常に満足しています。Rubyのメンテナー、コミッターの方々に感謝を忘れず、引き続きRubyの最新動向をキャッチアップおよび実践していきたいと思います。

## 参考文献
https://tech.timee.co.jp/entry/2025/03/26/132435
https://techblog.zozo.com/entry/effects-and-considerations-of-ruby3-yjit-on-wear
https://tech.smarthr.jp/entry/2025/03/28/123020
https://zenn.dev/dely_jp/articles/094f3084e9fa2c
https://zenn.dev/pharmax/articles/add57693adf021
https://tech.medpeer.co.jp/entry/2024/07/05/165000
https://k0kubun.hatenablog.com/entry/ruby-3-3-yjit
https://docs.ruby-lang.org/en/3.4/yjit/yjit_md.html#:~:text=%2D%2Dyjit%2Dmem%2Dsize%3DN%3A%20soft%20limit%20on%20YJIT%20memory%20usage%20in%20MiB%20(default%3A%20128).%20Tries%20to%20limit%20code_region_size%20%2B%20yjit_alloc_size