+++
title="foreverで動作しているNode.jsのアプリケーションログをCloudWatchに送る"
[taxonomies]
tags=["AWS", "CloudWatch", "Node.js", "forever"]
+++

forever を利用して daemon で起動しているアプリケーションのログを CloudWatch に流す方法です。

### forever のログファイル

forever はアプリケーション起動時にデフォルトで `~/.forever` 以下に `a93c.log` のようなランダムなファイル名のログファイルを出力します。

今回は CloudWatch Agent で取得するためファイル名を固定します。

## forever の config ファイルを作成

forever は起動引数または設定ファイルを用いることにより起動時の設定を細かく設定することができます。
[https://github.com/foreversd/forever#usage](https://github.com/foreversd/forever#usage)

今回はプロジェクトのルートに `.forever/config.json` ファイルを作成し、起動時に読み込むようにします。
json のファイルは以下のようにします。

```json
{
  "uid": "app",
  "append": true,
  "script": "server.js",
  "sourceDir": "dist/src/server",
  "workingDir": ".",
  "logFile": "logs/forever.log",
  "outFile": "/home/ec2-user/.forever/logs/out.log",
  "errFile": "/home/ec2-user/.forever/logs/error.log"
}
```

uid やディレクトリはお好みでよいのですが、forever では 3 つのログファイルを出力することができます。
それぞれは以下の特徴があり、cwd が微妙に異なります。

- `logFile` ... log も error も含んだ総合的なログ。デフォルトだと `~/.forever/[id].log` に出力される。(つまり、cwd が.forever になっている)
- `outFile` ... log のみが出力される。相対的になパスに出力される(つまり、cwd はプロジェクトのルートとなる)。記載したパスのディレクトリがない場合は、エラーになりなにも出力されないので注意。
- `errFile` ... error のみが出力される。パスの位置は outFile と同じ

なので、outFile, errFile は絶対パスにしています。

これで、`~/.forever/logs` ディレクトリに `forever.log` , `out.log`,`error.log`の 3 つのログファイルが出力されるようになります。forever を以下のコマンドで起動し、ログファイルが表示されていることを確認します。

```sh
$ forever start .forever/production.json
```

## EC2 事前準備

続いて EC2 側の設定します。

### IAM ロールに CloudWatch Agent 用の Policy をアタッチ

EC2 インスタンスの IAM ロールに `CloudWatchAgentServerPolicy`,`CloudWatchAgentAdminPolicy`をアタッチします。

### EC2 サーバに CloudWatch Agent をインストール

次にサーバにログインし、CloudWatch Agent をインストールします。Amazon Linux 2 を使っている場合は yum を実行するだけで、インストールできます。他の OS の場合は手動でインストールもできます。詳細は[こちら](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Agent-on-EC2-Instance-fleet.html#start-CloudWatch-Agent-EC2-fleet)を参照ください。

```sh
$ sudo yum install amazon-cloudwatch-agent
```

初回の設定を行う場合は以下のウィザードを実行する

```sh
$ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

### CloudWatch Agent の設定ファイルを修正

CloudWatch Agent の設定で`logs`(ログファイル)の設定を以下のようにします。
今回は info 系のログと error 系のログで別々に表示したいので、それぞれを別のログストリームに表示します。まとめて取得する場合は"forever.log"の方を参照してもよいです。またロググループ、ログストリーム名はお好みで問題ないと思います。

```json
// /opt/aws/amazon-cloudwatch-agent/bin/config.json

{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/ec2-user/.forever/logs/out.log",
            "log_group_name": "web-app",
            "log_stream_name": "{instance_id}_app_out_log"
          },
          {
            "file_path": "/home/ec2-user/.forever/logs/error.log",
            "log_group_name": "web-app",
            "log_stream_name": "{instance_id}_app_error_log"
          }
        ]
      }
    }
  },
  "agent": {...},
  "metrics": {...}
}

```

### CloudWatch Agent の起動

設定ファイルができたら、CloudWatch Agent を起動（もしくは再起動）します。

```sh
$ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```

以上で CloudWatch Agent の設定ができました。設定ができたら node.js のアプリケーションをデプロイし、forever でアプリケーションを起動します。

AWS コンソール上の CloudWatch > ロググループにて設定したログが表示されるようになります。

これで ssh でログを見に行かなくてもブラウザでログが見られるようになりました 🎉
また、エラーログが流れればそれに応じたアラートを設定することができます 🙌

---

### 参考

[https://tech.fusic.co.jp/posts/2020-12-12-cloudwatch-agent/](https://tech.fusic.co.jp/posts/2020-12-12-cloudwatch-agent/)
