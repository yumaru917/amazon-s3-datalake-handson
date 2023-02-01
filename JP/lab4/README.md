------------------------------------------------------------------------------------
Copyright <first-edit-year> Amazon.com, Inc. or its affiliates. All Rights Reserved.  
SPDX-License-Identifier: MIT-0

------------------------------------------------------------------------------------


# Lab4：アプリケーションログの永続化と長期間データの分析と可視化
ストリームデータを Amazon Kinesis Data Firehose（以降、Kinesis Data Firehose）に送信後、 Amazon S3（以降、S3）に保存することで長期保存します。その後、 Amazon Athena（以降、Athena）を用いて、アドホックな分析を行い、 Amazon QuickSight（以降、QuickSight）で可視化します。


## Section1：S3, Kinesis Data Firehose の設定
### Step1：S3 バケットの作成

 1. AWS マネジメントコンソールのサービス一覧から **S3** を選択し、画面の **[バケットを作成]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.15.17.png"> 
 <img src="images/Screen Shot 2023-01-26 at 15.15.29.png">  
 
 2. バケット名を以下のルールに従って入力し、画面左下の **[バケットを作成]** をクリックします。  

    - バケット名：[YYYYMMDD]-handson-minilake-[Your Name][Your Birthday]
    - [YYYYMMDD]：ハンズオン実施日
    - [Your Name]：ご自身のお名前
    - [Your Birthday]：ご自身の誕生日の日にち


    **Note：** S3 バケット名はグローバルで一意である必要がありますが、バケット作成ができればバケット名は任意でも構いません。
    <img src="images/Screen Shot 2023-01-26 at 15.16.47.png"> 

  

### Step2：Kinesis Data Firehose の作成

 1. AWS マネジメントコンソールのサービス一覧から **Kinesis** を選択し、 Kinesis Data Firehose の **[配信ストリームを作成]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.17.26.png">
 
 2. **[ソース]** に **[Direct PUT]** に **[送信先]** に「 Amazon S3」、そして **[配信ストリーム名]** に「 **minilake1**（任意）」と入力します。
 
    **Note：** 「 **minilake1**（任意）」を異なる名前に指定した場合、後続の手順において、「 **/etc/td-agent/td-agent.conf** 」のファイルにある「 **delivery_stream_name minilake1** 」の指定を合わせて変更いただく必要があります。
    <img src="images/Screen Shot 2023-01-26 at 15.17.56.png">
    <img src="images/Screen Shot 2023-01-26 at 15.18.22.png">
 
 3. **[データ変換]** が **[無効]** 、 **[レコード形式を転換]** が **[無効]** のままになっているか確認します。
 
 4. **[送信先の設定]** で、**[S3 バケット]** に **Step1** で作成したバケットを選択します。 **[S3 バケットプレフィックス]** に「 **minilake-in1/** 」を入力します。
 
    **Note：** **[S3 バケットプレフィックス]** の最後の「 **/** 」を忘れないように注意してください。 S3 への出力時のディレクトリとなり、デフォルトの場合、指定プレフィックス配下に「 **YYYY/MM/DD/HH** 」が作られます。
    <img src="images/Screen Shot 2023-01-26 at 15.18.38.png">
    <img src="images/Screen Shot 2023-01-26 at 15.18.46.png">
    <img src="images/Screen Shot 2023-01-26 at 15.19.07.png">
 
 5. 画面右下の **[配信ストリームを作成]** をクリックします。

    **Note：** 今回は設定しませんが、データの圧縮、暗号化も可能です。大規模データやセキュリティ要件に対して、有効に働きます。 
 
 6. **[Status]** が「 **Creating** 」となります。数分で「 **Active** 」になるので次の手順に進めてください。


## Section2：EC2 の設定変更
### Step1：IAM ロールのポリシー追加

作成済の「 **handson-minilake**（任意）」の IAM ロールに以下のようにポリシーを追加します。 

 1. AWS マネジメントコンソールのサービス一覧から **IAM** を選択し、 **[Identity and Access Management (IAM)]** 画面の左ペインから **[ロール]** を選択し、「 **handson-minilake-role**（任意）」のロール名をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.19.58.png">
 <img src="images/Screen Shot 2023-01-26 at 15.20.09.png">
 
 
 2. **[許可]** タブを選択し、 **[許可を追加]** をクリックし、 **[ポリシーをアタッチ]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.20.25.png">
 
 3. 検索窓で「 **amazonkinesisfirehose** 」と入れ検索し、  **[AmazonKinesisFirehoseFullAccess]** にチェックを入れ、 **[ポリシーをアタッチ]** をクリックします。  
    **Note：** **[AmazonKinesisAnalyticsFullAccess]** ではなく、 **[AmazonKinesisFirehoseFullAccess]** となります。
    <img src="images/Screen Shot 2023-01-26 at 15.20.43.png">
 
 4. 変更実施したロールの **[許可]** タブを選択し、 **[AmazonKinesisFirehoseFullAccess]** がアタッチされたことを確認します。
 <img src="images/Screen Shot 2023-01-26 at 15.21.13.png">

### Step2：Fluentd の設定
Fluentd から Kinesis Data Firehose にログデータを送信するための設定を行います。  

   1. Kinesis Data Firehose のプラグインのインストール状況を確認します。

      **Asset** 資料：[4-cmd.txt](asset/ap-northeast-1/4-cmd.txt)

 ```
 # td-agent-gem list | grep plugin-kinesis
 ```
   **[実行結果例]**  
   
   ```
  fluent-plugin-kinesis (2.1.0)
   ```
   <img src="images/Screen Shot 2023-01-26 at 15.21.44.png">
 
   3. 本手順については、どの Lab から開始したかによって、適用する設定ファイルが異なる為、ご自身が実施された手順に応じて、 Fluentd の設定を変更してください。
#### (a) Lab1, 2, 3 から続けて、 Lab4 を実施している場合（**Asset** 資料：[4-td-agent1.conf](asset/ap-northeast-1/4-td-agent1.conf)）
 
 3. 「 **/etc/td-agent/td-agent.conf** 」の中身を **Lab4** 向けに変更するために、あらかじめ用意しておいた **asset** に 以下の cp コマンドを用いて置き換えます。その際、ファイルを上書きするかの確認が入る為、 **yes** と入力します。 

 ```
 # cp -p /root/asset/4-td-agent1.conf /etc/td-agent/td-agent.conf
 ```   

 #### (b) Lab1 を実施し、その後　Lab4 を実施している場合（**Asset** 資料：[4-td-agent2.conf](asset/ap-northeast-1/4-td-agent2.conf) ）

 3. 「 **/etc/td-agent/td-agent.conf** 」の中身を **Lab4** 向けに変更するために、あらかじめ用意しておいた **asset** に 以下の cp コマンドを用いて置き換えます。その際、ファイルを上書きするかの確認が入る為、 **yes** と入力します。 

 ```
 # cp -p /root/asset/4-td-agent2.conf /etc/td-agent/td-agent.conf
 ```    
 <img src="images/Screen Shot 2023-01-26 at 15.22.09.png">

 #### 以下の手順からは、上記両方の場合において実施します。
 
   4. Fluentd を再起動します。
 
       **Asset** 資料：[4-cmd.txt](asset/ap-northeast-1/4-cmd.txt)
 
 ```
 # /etc/init.d/td-agent restart
 ```
   <img src="images/Screen Shot 2023-01-26 at 15.22.24.png">
 
   5. S3 にデータが出力されていることを確認します。  
   
      **Note：** 数分かかります。（S3 のパスの例：20190927-handson-minilake-test01/minilake-in1/2019/09/27/13）
      <img src="images/Screen Shot 2023-01-26 at 15.27.54.png">

   6. Kinesis Data Firehose の画面において、作成した **配信ストリーム** の「 **minilake1**（任意）」を選択し、 **[Monitoring]** タブをクリック、表示に時間がかかる為、次の手順に進みます。


## Section3：Glue Crawler, Athena の設定変更
### Step1：IAM ロールのポリシー追加
作成済の「 **handson-minilake**（任意）」の IAM ロールにポリシーを追加します。  

 1. AWS マネジメントコンソールのサービス一覧から **IAM** を選択し、 **[Identity and Access Management (IAM)]** 画面の左ペインから **[ロール]** を選択し、「 **handson-minilake**（任意）」のロール名をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.28.39.png">
 
 2. **[許可]** タブを選択し、 **[許可を追加]** をクリックし、 **[ポリシーをアタッチ]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.28.48.png">
 
 3. 検索窓で「 **awsglue** 」と入れ検索し、 **[AWSGlueServiceRole]** にチェック、検索窓で「 **amazons3** 」と入れ検索し、 **[AmazonS3ReadOnlyAccess]** にチェックを入れ、 **[ポリシーをアタッチ]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.29.10.png">
 <img src="images/Screen Shot 2023-01-26 at 15.29.44.png">
 
 4. **[AWSGlueServiceRole]** と **[AmazonS3ReadOnlyAccess]** がアタッチされたことを確認します。
 
 5. **[信頼関係]** タブをクリックし、 **[信頼ポリシーを編集]** ボタンをクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.30.02.png">
 
 6. **[信頼ポリシーを編集]** 画面において、**”Service”: “ec2.amazonaws.com”** の箇所に **glue** を追記します。 **[]**でくくり、 **カンマ** で区切り、 **glue.amazonaws.com** を追記し、**[ポリシーを更新]** をクリックします。
  
    **Asset** 資料：[4-policydocument.txt](asset/ap-northeast-1/4-policydocument.txt) 
 
 **[記入例]**
 
 ```
 {
 		"Version": "2012-10-17",
 		"Statement": [
			{
      		"Effect": "Allow",
      		"Principal": {
        		"Service": [             
          		"glue.amazonaws.com",
          		"ec2.amazonaws.com"
        		]                        
      		},
      		"Action": "sts:AssumeRole"
    		}
  		]
}
 ```
  <!-- <img src="images/Screen Shot 2023-01-26 at 15.31.06.png"> -->
 
 
### Step2：Glue Crawler を使ったスキーマの自動作成

 1. AWS マネジメントコンソールのサービス一覧で **AWS Glue** を選択し、 **[AWS Glue]** 画面の左ペインにおいて、 **[クローラ]** を選択し、 **[クローラの追加]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.34.53.png">


      **Note：** この時、画像のように最新版の英語のページが表示される場合があります。
      <img src="images/Screen Shot 2023-01-26 at 15.34.04.png">
      その際は **[Crawlers]** を選択し、画面上部に出る青いポップアップ内の **[old console]** を選択することによって、日本語のページを表示できます。
      <img src="images/Screen Shot 2023-01-26 at 15.34.25.png">

 2. **[クローラの名前]** に「 **minilake-in1**（任意）」と入力し、 **[次へ]** をクリックし、続いての画面もそのまま **[次へ]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.35.13.png">
 <img src="images/Screen Shot 2023-01-26 at 15.35.22.png">

 3. **[データストアの追加]** 画面において、 **[インクルードパス]** に作成した「 **s3://[S3 BUCKET NAME]/minilake-in1**（任意）」を入力し、 **[次へ]** をクリックします。

	**Note：** **[S3 BUCKET NAME]** には、ご自身で作成された S3 バケットの名前を入力ください。
   <img src="images/Screen Shot 2023-01-26 at 15.36.02.png">
 
 4. **[別のデータストアの追加]** 画面においても、 **[次へ]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.36.13.png">
 
 5. **[IAM ロールの選択]** 画面において、 **[既存の IAM ロールを選択]** にチェックを入れ、作成したロール「 **handson-minilake-role**（任意）」を選択し、 **[次へ]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.36.31.png">
 
 6. 続いての画面も **[頻度]** も **[オンデマンドで実行]** のままにし、 **[次へ]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.36.41.png">

 7. **[クローラの出力を設定する]** 画面において、 **[データベースの追加]** をクリックし、ポップアップした画面で **[データベース名]** に「 **minilake**（任意）」と入力し、 **[作成]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.36.52.png">
 <img src="images/Screen Shot 2023-01-26 at 15.37.08.png">

 8. **[クローラの出力を設定する]** 画面に戻り、 **[次へ]** をクリックします。

 9. 続いての画面の内容を確認し、 **[完了]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.37.25.png">

 10. **[クローラ]** の画面において、作成したクローラの「 **minilake-in1**（任意）」にチェックを入れ、 **[クローラの実行]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.37.45.png">
 ステータスが **[Starting]** になってから、数分待ちます。ステータスが再度 **[Ready]** に戻ったら、左ペインの **[テーブル]** をクリックします。
 <img src="images/Screen Shot 2023-01-26 at 15.37.52.png">
 <img src="images/Screen Shot 2023-01-26 at 15.38.36.png">
 <img src="images/Screen Shot 2023-01-26 at 15.39.55.png">

 11. 「 **minilake_in1**（任意）」のテーブルが作成されていることを確認し、テーブル名の箇所をクリックし、スキーマ定義を確認します。
 <img src="images/Screen Shot 2023-01-26 at 15.40.34.png">
 <img src="images/Screen Shot 2023-01-26 at 15.40.42.png">
 

### Step3：Athena でクエリ実行

 1. AWS マネジメントコンソールのサービス一覧で、 **Athena** を選択します。
 <img src="images/Screen Shot 2023-01-26 at 15.40.54.png">

 2. **[クエリエディタ]** を選択し、お使いの AWS アカウントで初めて Athena を利用する場合、 **[設定を編集]** を選択し **[クエリの結果の場所]** に「 **s3://[S3 BUCKET NAME]/result/**（任意）」を入力し、**[保存]** をクリックします。

	**Note：** **[S3 BUCKET NAME]** には、ご自身で作成された S3 バケットの名前を入力ください。該当の AWS アカウントで、 Athena の画面を初めて開く場合は、「最初のクエリを実行する前に、 Amazon S3 でクエリ結果の場所を設定する必要があります。詳細はこちら」と画面上に出ている可能性があります。その場合は、「Amazon S3 でクエリ結果の場所を設定する」のリンクに飛ぶことで設定可能です。
   <img src="images/Screen Shot 2023-01-26 at 15.41.06.png">
   <img src="images/Screen Shot 2023-01-26 at 15.41.20.png">
   <img src="images/Screen Shot 2023-01-26 at 15.41.41.png">
   <img src="images/Screen Shot 2023-01-26 at 15.42.24.png">

 3. **[データベース]** において「 **minilake**（任意）」を選び、テーブルは先程作成した「 **minilake_in1**（任意）」を選び、テーブル名の右端の **[点マーク]** をクリックし、 **[テーブルをプレビュー]** をクリックします。

 4. クエリ結果が画面下部に表示されることを確認します。

 5. クエリエディタで下記 SQL を実行します。
 
    **Asset** 資料：[4-cmd.txt](asset/ap-northeast-1/4-cmd.txt)
 
 ```
 SELECT * FROM "minilake"."minilake_in1";

 ```
 <img src="images/Screen Shot 2023-01-26 at 15.43.00.png">

 **[実行結果例]**
 
 ``` 
 (実行時間: 4.84 秒, スキャンしたデータ: 135.22 KB)

 ```
 <img src="images/Screen Shot 2023-01-26 at 15.43.13.png">

 6. Where 句をつけたクエリを実行してみます。

    **Asset** 資料：[4-cmd.txt](asset/ap-northeast-1/4-cmd.txt)

 ```
 SELECT * FROM "minilake"."minilake_in1" where partition_0 = '2019' AND partition_1 = '09' AND partition_2 = '27' AND partition_3 = '14';
 ```

   **Note：** Where 句の日付はデータが存在するものを入力してください。

**参考**：[Athena におけるクエリ実行の補足説明](additional_info_lab4.md)


### Step4：QuickSight の設定

 1. AWS マネジメントコンソールのサービス一覧から **QuickSight** を選択します。 QuickSight を初めて使う方はサインアップがまだされていない為、サインアップの画面が出るため、 **[Sign up for QuickSight]** をクリックします。  

    **Note：** すでに東京リージョン以外で登録されている場合、 **[QuickSight の管理]** → **[お客様のサブスクリプション]** で、所持しているサブスクリプションを選択し **[サブスクリプション削除]** をクリックします。実施後、数分待つと、再度 Sign up することが可能になります。 
    <img src="images/Screen Shot 2023-01-26 at 15.43.52.png">
    <img src="images/Screen Shot 2023-01-26 at 15.44.03.png">

 2. 画面右上のアイコンをクリックし、 **[日本語]** に変更します。  

 3. **[QuickSight アカウントの作成]** で **[エンタープライズ版]** を選び、 **[続行]** をクリックします。

    **Note：** 1GB までは無料利用枠ですが、無料利用期間が終わってる場合は、1ヶ月単位で $24 かかるので、費用が気になる場合、 QuickSight の手順は飛ばしていただいても構いません。 
    <img src="images/Screen Shot 2023-01-26 at 15.44.22.png">
    <img src="images/Screen Shot 2023-01-26 at 15.44.26.png">

 
 4. **[リージョンを選択]** で **[Asia Pacific (Tokyo)]** を選択し、 **[QuickSight アカウント名]** に任意の名前、 **[通知のEメールアドレス]** にご自身のメールアドレスを入力します。
 <img src="images/Screen Shot 2023-01-26 at 15.46.14.png">  

 5. **[これらのリソースへのアクセスと自動検知を許可する]** の項目で、 **[Amazon Athena]** にチェックを入れます（すでにチェックが入っている場合はそのままとします）。 
 <img src="images/Screen Shot 2023-01-26 at 15.46.37.png">

 6. **[S3 バケットを選択する]** をクリックします。
   
 7. **Section1** の **Step1** で作成したS3のバケット名にチェックを入れ（すでにチェックが入っている場合はそのままとします）、 **[完了]** をクリックします。  
 <img src="images/Screen Shot 2023-01-26 at 15.46.46.png">

 8. **[完了]** をクリックします。  
 <img src="images/Screen Shot 2023-01-26 at 15.46.54.png">

 9. **[Amazon QuickSight に移動する]** をクリックし、初回ログイン時のみ表示されるダイアログを消去します。
 <img src="images/Screen Shot 2023-01-26 at 15.47.27.png">

 10. **[新しい分析]** をクリックします。  
 <img src="images/Screen Shot 2023-01-26 at 15.47.55.png">

 11. **[新しいデータセット]** をクリックします。  
 <img src="images/Screen Shot 2023-01-26 at 15.48.04.png">

 12. **[Athena]** をクリックし、ポップアップ画面で **[データソース名]** に「 **minilake1**（任意）」と入力し、 **[接続を検証]** をクリックし、成功したら **[データソースを作成]** をクリックします。  
 <img src="images/Screen Shot 2023-01-26 at 15.48.31.png">
 <img src="images/Screen Shot 2023-01-26 at 15.48.54.png">
 <img src="images/Screen Shot 2023-01-26 at 15.49.01.png">

 13. **[データベース]** を「 **minilake**（任意）」、 **[テーブル]** を「 **minilake-in1**（任意）」を選び、 **[選択]** をクリックし、 **[迅速な分析のために SPICE へインポート]** を選択し、 **[Visualize]** をクリックします。  
 <img src="images/Screen Shot 2023-01-26 at 15.49.15.png">
 <img src="images/Screen Shot 2023-01-26 at 15.49.31.png">

 14. **[インポートの完了]** がポップアップされたら準備完了です。 **[フィールドリスト]** や **[ビジュアルタイプ]** を適当に選び、データが可視化されていることを確認してください。  
 <img src="images/Screen Shot 2023-01-26 at 15.49.54.png">

 **[完了想定画面]**
 <img src="images/quicksight_capture01.png">  
 

## Section4：まとめ

ストリーミングデータを直接データストアに永続化し、長期間の保存を可能にした上で、アドホックな分析・可視化を行う基盤ができました。

<img src="../images/architecture_lab4.png" >

Lab4 は以上です。選択されているパターンに合わせて次の手順を実施ください。

（1） ニアリアルタイムデータ分析環境（スピードレイヤ）の構築：[Lab1](../lab1/README.md) → [Lab2](../lab2/README.md) → [Lab3](../lab3/README.md)  
（2） 長期間のデータをバッチ分析する環境（バッチレイヤ）の構築と、パフォーマンスとコストの最適化：[Lab1](../lab1/README.md) → [Lab4](../lab4/README.md) or [Lab5](../lab5/README.md) → [Lab6](../lab6/README.md)  
（3） すべて実施：[Lab1](../lab1/README.md) → [Lab2](../lab2/README.md) → [Lab3](../lab3/README.md) → [Lab4](../lab4/README.md) → [Lab5](../lab5/README.md) → [Lab6](../lab6/README.md) 

環境を削除される際は、[こちら](../clean-up/README.md)の手順をご覧ください。
