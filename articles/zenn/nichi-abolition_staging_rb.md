## はじめに

Ruby on Railsでは`config/environments`配下に環境設定ファイルが用意されています。
デフォルトでは以下3つのファイルが用意されています。

- production.rb
- development.rb
- test.rb

これらのファイルを使用して、`RAILS_ENV`環境変数に環境名を指定することでそれぞれの環境設定でアプリケーションを立ち上げることが出来ます。
個別に環境設定ファイルを作成することが出来るため、Staging環境だけstaging.rbで動かすような運用が可能です。`config/environments`配下にstaging.rbを作成し`RAILS_ENV=staging`を指定してRailsを起動するだけで、Staging環境固有の設定でアプリケーションを動かすことが出来ます。また`Rails.env.staging?`という判定を使用してStaging環境の時だけ〇〇をするという動作を簡単に記述出来ます。

一見便利そうですが...

## staging.rbはアンチパターン

staging.rbが存在することで、当然のことながらStaging環境とProduction環境が異なる設定で動きます。一見環境ごとの制御が出来て便利そうなのですが、Staging環境とProduction環境が乖離する原因となります。これはThe Twelve-Factor Appでも言及されている方法論「開発/本番一致」に反します。

The Twelve-Factor Appとは、Herokuプラットフォーム上で開発・運用・スケールした幾千ものアプリケーションから得られた知見を、モダンなWebアプリケーションのあるべき姿としてまとめた12の方法論です。

https://12factor.net/

Herokuのガイドにも同様の記述があります。
>Heroku では、開発環境と本番環境をできるだけ似せておくことを強くおすすめします。これを開発/本番パリティ​と呼びます。staging​をできるだけ production​ と似せておくこともお勧めします。これらが違い過ぎると、ステージングアプリをデプロイしてテストしても本番インスタンスが機能することを検証できません。

https://devcenter.heroku.com/ja/articles/deploying-to-a-custom-rails-environment

環境の差異が残っている状態では以下に代表されるような事象を引き起こしたり、またプロダクトの品質に悪影響を及ぼす可能性があります。

- Staging環境では正常に動いたがProduction環境で想定外の動きをしてしまう
- Production環境で見つかったバグがStaging環境で再現しない
- Staging環境での品質評価プロセスの信頼度が下がる

実際にLRMではStaging環境にて staging.rb を用いた構成で動作確認を行っていました。そのため、ライブラリのバージョンアップやソースコードの変更時に、環境差異起因でStaging環境では再現されない事象が発生するケースがありました。
このようなリスクを取り除くため、staging.rbを廃止しました。

## staging.rbを廃止する

staging.rbを削除し、Staging環境でも`RAILS_ENV=production`を使用します。
今まで`RAILS_ENV=staging`で動かしていたStaging環境を変更するにあたり行った作業を簡単に紹介します。

### staging.rbの削除

staging.rbはもう使わないので削除します。

### Rails.env.staging?の記述を検知する

rubocopには、定義されていない環境が呼び出されている場合に警告を出してくれる `Rails/UnknownEnv`というルールがあります。このルールを有効化しておくことで、`Rails.env.staging?`の記述に気づくことができ、漏れなく修正できます。

```ruby
Rails/UnknownEnv:
  Enabled: true
```

### Rails.env.production?/Rails.env.staging?での制御を修正

下記はBasic認証を有効化するか無効化するかを制御しているサンプルコードです。

```ruby
  def basic_auth_required?
    if Rails.env.production?
      true
    elsif Rails.env.staging?
      false
    end
  end
```

`Rails.env`を見て処理を変えていますが、このような場合は`Rails.env`で分岐させるのではなく環境変数によって制御するべきです。この処理を有効化するか、無効化するか
`BASIC_AUTH_ENABLED=true`のような環境変数で制御します。

```ruby
  def basic_auth_required?
    ENV['BASIC_AUTH_ENABLED'] == 'true'
  end
```

### production.rbの修正

どうしてもStaging環境だけ変更したい値は環境変数で入れます。
例えばログレベルは本番環境は`info`、検証環境は`debug`で動かしたいので、以下の様に`ENV.fetch`を使用して環境変数を取得します。

```ruby
Rails.application.configure do
  ~~省略~~
  config.log_level = ENV.fetch('RAILS_LOG_LEVEL', 'info').to_sym
  ~~省略~~
end
```

この時、環境変数の設定ミスでProduction環境の動作に影響が出ないよう、デフォルトの値はProduction環境で元々設定されていた値にしました。

## 実際に廃止してみた所感

環境変数の設定が増えて心配しながらも、Production環境の動作には一切の不具合なくリリースを終えました。廃止以前はデプロイ時にStaging環境では再現しない不具合に遭遇する不安がありましたが、その不安が緩和され、Staging環境で信頼度の高いQAを行うことができています。
