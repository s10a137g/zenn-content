---
title: "BigQuery Data Transfer Service の「全量投入」と「差分投入」の違いを、Salesforce 連携で試した"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bigquery", "salesforce", "googlecloud"]
published: true
---

[BigQuery Data Transfer Service（DTS）の Salesforce 連携ドキュメント](https://cloud.google.com/bigquery/docs/salesforce-transfer)を読んでいたら、転送モードに「全量投入」と「差分投入（プレビュー）」の2つがあることに気づきました。

差分があるなら実行もコストも効率化できそうですが、実際どう挙動が違うのか気になったので、無料の Salesforce Developer Edition を使って手元で試してみました。

## 用語：全量投入と差分投入

DTS の Salesforce コネクタには転送モードがあり、ざっくり次の違いがあります。

| モード | 仕組み | 必要なもの |
| --- | --- | --- |
| 全量投入（洗い替え） | 毎回データセット全体を取り直す | なし |
| 差分投入 Append | ウォーターマーク以降の行を追加（既存行は更新せず、更新行も重複として追加されうる） | 作成時のみ更新されるウォーターマーク列 |
| 差分投入 Upsert | 主キーで更新 or 新規挿入 | 毎更新で変わるウォーターマーク列（SystemModstamp 等）＋一意な主キー（Id 等） |

言葉だけだと分かりにくいので、実際に動かして「2回目の転送で何が起きるか」を見てみます。

## 検証環境

- **Salesforce：Developer Edition（無料）**。検証用の Salesforce 環境は、有料エディションに付属する Sandbox ではなく、無料の Developer Edition で用意できます。API（Bulk API）もデフォルトで有効です
- **GCP：BigQuery + Data Transfer Service**
- 対象オブジェクト：Account（取引先）

まず Salesforce 側に検証用の取引先データ（架空のもの）を用意しました。

![Salesforce に登録した取引先データ](/images/salesforce-accounts.png)

DTS の Salesforce 連携は OAuth の Client ID / Secret で認証するため、事前に Salesforce 側で外部クライアントアプリ（接続アプリ）を作成し、クライアントログイン情報フローを有効化したうえで、My Domain・Consumer Key・Consumer Secret を控えておきます。

![外部クライアントアプリの作成](/images/salesforce-external-client-app.png)

クライアントログイン情報フローを使う場合は、「(ユーザー名) として実行」に実行ユーザーを指定するのを忘れないようにします（ここが未設定だとトークン取得で失敗します）。

![OAuth ポリシーと実行ユーザーの指定](/images/salesforce-oauth-policy-run-as.png)

### ハマりどころ：My Domain の指定

最初の転送実行で、次のエラーが出ました。

```
java.net.UnknownHostException: orgfarm-xxxx-dev-ed.my.salesforce.com
```

原因は My Domain のホスト名でした。Developer Edition の正しいホスト名には `.develop` が含まれます（`orgfarm-xxxx-dev-ed.develop.my.salesforce.com`）。設定 → My Domain で実際の URL を確認し、`.develop` まで含めて指定したら解決しました。

## デモ①：差分投入（Upsert）

転送を作成し、ソースに Salesforce を選び、My Domain・Client ID・Client secret を入力します。Ingestion type で **Incremental（プレビュー）** を選ぶと、差分投入になります。

![DTS の転送作成（データソースの詳細）](/images/dts-create-transfer-source.png)

転送対象は Account を選択します。

![転送対象に Account を選択](/images/dts-select-asset-account.png)

差分投入では、変更を検知するためのウォーターマーク列と主キーを指定します。今回はウォーターマーク列に Salesforce の `SystemModstamp`（レコードの最終更新時刻を表す標準フィールド）、主キーに `Id` を指定しました。

![ウォーターマーク列に SystemModstamp を指定](/images/dts-watermark-systemmodstamp.png)

### 1回目の実行（初回）

作成した転送を「今すぐ転送を実行」で動かします。

![転送の詳細と手動実行](/images/dts-transfer-detail-upsert.png)

初回はまるごと取り込まれます。BigQuery 側にテーブルができ、データが入りました。

![BigQuery に取り込まれた Account テーブル](/images/bigquery-account-preview.png)

> 補足：Developer Edition には標準のサンプル取引先（Edge Communications など）が最初から入っているため、自分で登録した10件に加えてそれらも一緒に転送されました。初回の転送件数は **23 件**（自作10件＋標準サンプル13件）です。

### Salesforce 側で1件だけ変更

ここで取引先名を `Cyberdyne Systems` → `Update Systems` に、**1 件だけ**変更します。これにより、その行の `SystemModstamp` が新しい時刻に書き換わります。

![1 件だけ名前を変更（Update Systems）](/images/salesforce-account-updated.png)

### 2回目の実行（変更後）

同じ転送をもう一度実行します。差分投入の本領はここで、前回以降に更新された行だけが対象になります。

実行ログを見ると、処理されたのは **1 件だけ**でした。

![差分実行のログ（Number of records: 1）](/images/dts-incremental-run-log.png)

残りのレコードには触れず、変更分のみが Upsert されています。BigQuery 側のテーブルも、対象の行だけが新しい名前に更新されました。つまり差分投入では、**2回目以降は変更分だけ**が転送されます。

## デモ②：全量投入（洗い替え）との対比

比較のため、別の転送を **全量（Full）** モードで作り、同じように実行します。

![Ingestion type に Full を選択](/images/dts-ingestion-type-full.png)

全量モードでは、Salesforce 側を何も変更していなくても、2回目の実行でも**全レコードが再度転送**されました。全量投入は、変更の有無にかかわらず毎回データセット全体を取り直します。

## 比較：2回目の転送で何が起きたか

| 観点 | 全量投入 | 差分投入（Upsert） |
| --- | --- | --- |
| 2回目の転送行数 | 全件 | 変更した 1 件のみ |
| 既存行の更新 | 洗い替えで反映 | 主キーで特定して更新 |
| 必要な設定 | なし | ウォーターマーク列＋主キー |
| 向くデータ | 小規模・常に最新で十分 | 状態が変わる CRM データ・大きめのデータ |

Salesforce の商談のように**既存レコードが更新される**データは、更新を追えてコストも抑えられる差分（Upsert）が基本的に相性が良いです。一方、データ量が小さく毎回取り直しても負担が小さいなら、設定がシンプルな全量投入も十分選択肢になります。

注意点として、差分の Append は既存行をチェックせず追加するため、主キーの一意性に不安があると重複の原因になります。確実に更新も追いたいなら Upsert が無難です。

## コスト面のひとこと

差分投入が効くのはコスト面でも同じです。DTS のサードパーティコネクタは、2025年9月25日以降、GA 到達したものからコンピュートリソース（スロット時間）に応じた従量課金になります（執筆時点・2026年6月の情報。最新の料金体系は[BigQuery の料金ページ](https://cloud.google.com/bigquery/pricing)を確認してください）。毎回全件を取り直す全量投入より、変更分だけの差分投入のほうが転送量が少なく、その分有利です。

なお、Salesforce の増分取り込みは現在 Preview で、プレビュー中のコネクタ利用は無料です。個人で試す範囲では、Salesforce（Developer Edition）も無料、BigQuery も毎月10GB のストレージと1TB のクエリ処理の無料枠に収まるため、ほぼ0円で一通り体験できます。

## まとめ

- **全量投入は毎回全件**を洗い替え。設定はシンプルだが転送量が大きい
- **差分投入は2回目以降は変更分だけ**。ウォーターマーク列と主キーが必要だが、更新を追えてコストも抑えられる
- 状態が変わる CRM データなら **差分（Upsert）が基本**。小規模なら全量も可

「1件だけ変更して再実行する」と違いが一目で分かるので、気になったら手元の Developer Edition で試してみるのがおすすめです。

今回は数十件ほどの小さなデータで挙動を見ただけなので、次は本番規模（数十万件）のデータで、全量と差分で実行時間やコストがどれくらい変わるのかを確かめてみたいと思います。