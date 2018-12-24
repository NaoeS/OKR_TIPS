- 目次
  * [Rails ガイドメモ](#rails-guide-memo)
  * [Rails リーディング](#rails-reading)

<a name="rails-guide-memo"></a>

# 🛤️ Rails ガイドメモ

## Active Record

### 基礎
- `Acrive Record` パターンが基礎となっている
- 複数化メカニズムが強力。例えば `Person` クラスは `people` テーブルになる
  * `Mouse Class` -> `mice table`
- `update_attributes` は非推奨となってガイドから削除された
- テーブルに自動追加されるカラム例
  * `lock_version` : モデルに `optimistic locking` を追加する
  * `type` : モデルで `Single Table Inheritance` を使用する場合に指定する
  * `テーブル名_count` : 所属しているオブジェクトの数をキャッシュする
- `set.table_name` を使用した場合は、テスト側で `set_fixture_class` を使う必要がある
- `create` はオブジェクト作成 & 永続化
- `save!` , `update!` でバリデーションに失敗すると、 `ActiveRecord::RecordInvalid` が発生する

### コールバック
- `before_xxx` で　`false or 例外` を発生させると、以降の処理を止めることができる
- 作成時
  * b_vali -> a_vali -> b_save -> around_save -> b_create -> around_create -> a_create -> after_save
- 更新時
  * 作成時の create が update に変わるだけ
- 削除時
  * b_destroy -> around_destroy -> a_destroy
    + `dependent: :destroy` より前に定義する。依存モデルが削除された時に `dependent` より前に発火すべきだから

### マイグレーション
- 逆方向に実行 (ロールバック) する方法が推測できない場合は `reversible` を使う
- `rails g model` すると勝手にマイグレーションファイルもできる
- `:primary_key` オプションで主キーを変えれる。 `id: false` で自動生成をやめれる
- `create_join_table` メソッドで中間テーブルを作成できる。 `table_name:` で名前を変えれる
- `change_column` は可逆ではない
- `remove_column` は 3 番目の引数でカラムの型を指定すればロールバック可能
- `reversible` を使う際、down ブロックで `ActiveRecord::IrreversibleMigration` を `raise` できる
- `revert` メソッド

## Action View

### テンプレート
- `<% %>` Ruby コードを実行するが出力はしない。 `<%= %>` は出力を行う
- 出力結果からホワイトスペースを取り除きたい場合は、`<%- -%>` を使う
- Builder テンプレートは `.builder` ファイルで、XML に特化している
  * `Jbuilder` は JSON 生成用の gem
  
### パーシャル
- 呼び出す際はアンダースコアは不要。ファイル名には付ける
- パーシャルはデフォルトで自身と同じ名前のローカル変数を持つ
  * 変数名を変えたい場合は、as オプションを使用する
- `collection` オプションに配列などを渡せば、いちいちブロックを書かずに繰り返しが実現できる
  * モデルとパーシャル名が同じならかなり短縮できる `<%= render partial: "product", collection: @products %>` -> `<%= render @products %>`

### レイアウト
- 多くのコントローラで共通して使用されるテンプレートのこと
- `app/views/layouts` に現在のコントローラ名のファイルがあるか探す⇢無ければ `application` が付くファイルを使用する
- コントローラで `layout` メソッドを使用するとレイアウトを上書きできる

### FormHelper
- `f.text_field :hoge` のように `input type`, 項目名となる
  * 出力された HTML の `name` には `#{modelname}[#{hoge}]` が出力される
  * label, password_field, radio_button, text_area, email_field, url_field なども同様
- `check_box('modelname', 'hoge')`でチェックボックスが出力できる
- `fields_for` を使えばフォームタグが作成されないため、既存のフォーム内に別モデルを追加できる

### FormOptionsHelper
- 様々な種類のコンテナを1つのオプションタグにまとめる役割を持つ
- TODO

### その他
- `<%= raw hoge %>` でエスケープされていない文字列を描画する

## ルーティング
- `resources` メソッドで 7 つのルーティングが作成される
  * index, new, create, show, edit, update, destroy
- 上からの記載順にマッチする 
- `_path` ヘルパーの名前はアクション名が先にくる
  * new_photo_path -> photos/new, edit_photo_path(10) -> photos/10/edit
- `_url` は `_path` を完全な URL にしたもの
- `to: 'users#show'` = `action: :show, controller: :users`
- `scope module: 'admin' do ~ end` を使用すると、`admin` でネームスペースを切らないルーティングとなる
- `scope '/admin' do ~ end` で `Admin::` でモジュールを切っていないコントローラを指定できる
- `:shallow` オプションでリソースのネストからリソースのみのルーティングを切り分けることができる。ネストの親でも指定できる
  * `:shallow_prefix` オプションでヘルパー名だけに接頭辞を付けれる
- concern でリソースの共通化ができる
- `get 'preview', on: :member` で member オプションのブロックが省略できる
- `on: :new` などで指定のアクションを挟める
- `photos(/:id)` で id が必須でないことを表せる
- Rails では GET で CSRF 対策が取られていないため、GET で永続化処理は行わない！
- `constraints: { id: /[A-Z]/}` で正規表現でパスをマッチさせれる
- Unicode 文字列をそのままルーティングで使用することもできる。使うのか？
  * `get 'おはよう', to: 'goodmorning#index'`
- `:member` は単一データに対して、`:collection` は全体に対してアクションを追加する時に使用する
- リソースのネストは1回に留める。親リソースが必要ないアクションは切り出してしまうと良い。
```
# 例
resources :articles do
  resources :comments, only: [:index, :new, :create]
end
respirces :comments, only: [:show, :edit, :update, :destroy]
```

## Unit Test

### 基礎
- `ActiveSupport::TestCase` のスーパークラスは `Minitest::Test`
- テストメソッドの実行順はすべてランダム
  * `config.active_support.test_order` オプションで設定可能
- `bin/rails test` で実行
  * `-n` , `--name` でメソッド単位で指定可能
  * `hoge_test.rb:5` で行を指定可能
- `Rails` では `rake test` で全テスト実行する
  * [参考](http://railsdoc.com/test)
- コントローラのテストをする際は、HTTP メソッドに対応するメソッドを使用する `get :show, :id => @user.to_param`

### 忘れそうなマッチャ（Minitest::Assertions）
- `assert_not_in_delta`
  * 引数の個数差を調べる
- `assert_throws`
  * 与えられたブロックが指定されたシンボルをスローすることを調べる
- `assert_kind_of`
  * サブクラスのインスタンスかどうか調べる
- `assert_predicate`
  * 与えたオブジェクトのメソッドが `true` であるか調べる
- `flunk`
  * 必ずテストが失敗することを明示する

### 忘れそうなマッチャ（Rails固有）
- `assert_recognizes(expected_options)`
  * 渡されたパスのルーティングが正しく扱われ、オプションが一致することを調べる
- `assert_generates(expected_path)`
  * 渡されたオプションは渡されたパスの生成に使用できるか調べる

### フィクスチャ
- テストデータの定義とカスタマイズか可能
- 一つのモデルに一つのフィクスチャファイルがあり、YAML 形式で記述する
- `ERB` も対応しているため、動的なデータも作成できる
- `ActiveRecord` のインスタンスなので、`users(:hoge)` で取得ができる。メソッドも使用可能

### システムテスト
- バックグラウンドでは `Capybara` が動いている
- 要素の内容を検証したいときは `assert_selector('h1', text: 'hoge')` のようにする
- `click_on` メソッドでリンクを探して押下する
- ドライバを変更したい時は `application_system_test_case.rb` の　　`driven_by` を変更する
  * Selenium, Poltergeist などがある
  * オプション例: `:using` (ブラウザ指定)、`:screen_size` (スクリーンショットのサイズ)、`:options` (ドライバでサポートされるオプション)
- `take_screenhot` メソッドを実行すると実行した時点の SS が撮影できる

## Action Mailer
### 基礎
- メール内容のビューは `app/views/#{class_name}` に作成される

### 添付ファイル
- `attachments` を使用する
  * `attachments['filename.jpg'] = File.read('/path/to/filename.jpg')`
  * 自動的に Base64 変換を行う
  * `attachments.inline` を使用すれば、ビュー側で URL を参照したりできる

### TO
- 複数の相手に送信するには、`to:` にアドレスの配列やカンマ区切りの文字列を渡せばいい
- メールアドレスではなく名前を表示させたい場合は、`to:` に `"#{name} <#{mail_address}>"` とする

## Active Support

### Numeric 拡張
- 数値には以下のメソッドが応答する
  * `bytes, kilobytes, megabytes, gigabytes, terabytes, petabytes, exabytes`

### フォーマッティング
- 数値は電話番号、通貨などにフォーマットできる
  * `5551234.to_s(:phone) # 555-1234`

### String 拡張
- 文字列に `html_safe` を使用すると、安全な文字列としてマークされる。マークを単純に付加するためなので注意
  * 普通は `raw` メソッドを使用するため、`html_safe` は使用しない
  * 安全にした文字列に `+` で文字列を足すと、エスケープされた上で結合される


## 模擬1
- 70問
  * [Rails技術者認定ブロンズ試験 模擬問題](https://www.school.ctc-g.co.jp/ruby/training_rails_bronze_01_10.html)

### 1回目
- 44/70 : 62%
  * 7, 11, 16, 19, 23, 24, 26, 31, 35, 36, 37, 38, 42, 43, 48, 50, 51, 53, 56, 57, 58, 59, 60, 65, 66, 69

### 2回目
- 55/70 : 78%
  * 17, 23, 28, 32, 35, 43, 44, 50, 53, 55, 56, 57, 58, 66, 69
  
### 3回目
- 64/70 : 91%
  * 19, 33, 35, 44, 55, 70

<a name="rails-reading"></a>

# 🛤️ Rails リーディング
- 取り敢えずあたりだけ。一つに絞るつもり

## ActiveRecord::Persistence

### save

## ActiveModel::Validations::ClassMethods

### validate

## ActiveSupport::Cache::Store

### fetch
