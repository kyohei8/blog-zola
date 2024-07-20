+++
title="Code Deployを使ったnode.jsアプリケーションデプロイ方法"
[taxonomies]
tags=["AWS", "CodeDeploy", "EC2", "Node.js"]
+++

node.js のアプリケーションを EC2 へ自動デプロイする方法です

# EC2 側の設定

## EC2 に CodeDeploy-Agent をインストール

[https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install-linux.html](https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install-linux.html)

こちらを参考に CodeDeploy-Agent をインストールする。途中の wget の url は東京リージョンの場合は、以下になると思います。

```bash
wget https://aws-codedeploy-ap-northeast-1.s3.ap-northeast-1.amazonaws.com/latest/install
```

CodeDeploy-agent の起動など

```bash
sudo service codedeploy-agent status
sudo service codedeploy-agent start
sudo service codedeploy-agent restart
```

CodeDeploy クライアントのログ確認方法

```bash
tail -F /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

## EC2 の IAM ロールに CodeDeploy のパーミッションを追加

EC2 から S3 のアーティファクトを取りにいけるようにするため、サービスロールを作成し、EC2 にアタッチします。

[https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/getting-started-create-iam-instance-profile.html](https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/getting-started-create-iam-instance-profile.html)

こちらの手順を順にやれば OK

**IAM ロール 設定後は CodeDeploy-agent をリスタートすること！**

```bash
# ログにこのようなエラーがでていた場合、リスタートする。
# ERROR [codedeploy-agent(3582)]: InstanceAgent::Plugins::CodeDeployPlugin::CommandPoller: Missing credentials - please check if this instance was started with an IAM instance profile
sudo service codedeploy-agent restart
```

# CodeCommit の設定

## CodeCommit へ push するためのユーザを追加します

※ Github を利用する場合は不要。

CodeCommit の Git に push できるよう、ユーザと ssh 周りを設定します。

[https://docs.aws.amazon.com/ja_jp/codecommit/latest/userguide/setting-up-ssh-unixes.html](https://docs.aws.amazon.com/ja_jp/codecommit/latest/userguide/setting-up-ssh-unixes.html)

ssh の config ファイルに以下のような記述を追加し、ssh が通れば OK です。

```bash
Host git-codecommit.*.amazonaws.com
  User APXXXXXXXXXXX
  IdentityFile ~/.ssh/codecommit_rsa
```

```bash
ssh git-codecommit.ap-northeast-1.amazonaws.com
# You have successfully authenticated over SSH...
```

## リポジトリの作成

CodeCommit の画面からリポジトリを作成します。作成された git のリモート先をローカルの git に追加し、push します。

# CodeDeploy の設定

## CodeDeploy のサービスロールを作成

CodeDeploy から EC2 等のサービスへアクセスできるようにするため、ロールを作成します。

[https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/getting-started-create-service-role.html](https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/getting-started-create-service-role.html)

## CodeDeploy のアプリケーションの作成

CodeDeploy の画面から [アプリケーションの作成] を選択しますた。

アプリケーション名を入力し、コンピューティングプラットフォームは「EC2/オンプレミス」を選択します。

## CodeDeploy のデプロイグループの作成

作成したアプリケーションの詳細画面から [デプロイグループの作成] を選択。

サービスロールは上記で作成したロール(CodeDeployServiceRole 等）を選択する。その他は適宜設定する。

# CodePipeline の設定

CodeCommit に push したファイルを元に自動的にデプロイが実行されるようにします。

## CodePipeline の作成

CodePipeline の画面から [パイプラインの作成] を選択。パイプライン名を設定し、「新しいサービスロール」を選択し、サービスロール名は `CodePipeline-Role` 等とします。

ソースは CodeCommit のリポジトリ、ブランチを選択。

ビルドステージは「スキップ」を選択。

デプロイプロバイダーは上記の CodeDeploy のアプリケーション、デプロイグループを設定します。

これでプロジェクトファイルのルートディレクトリに `appspec.yml` があれば自動的にその内容でデプロイされます。
