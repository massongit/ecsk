[English](https://github.com/yukiarrr/ecsk/blob/main/README.md) / [日本語](https://github.com/yukiarrr/ecsk/blob/main/README.ja.md)

# ecsk

**EC**S + Ta**sk** = **ecsk** 😆

ecskは、インタラクティブにAmazon ECS API（run-task, execute-command, stop-task）を呼び出したり、ECSとローカルの間でファイルをコピーしたり、ログを表示したりできるCLIツールです。

![ecsk](https://github.com/yukiarrr/ecsk/raw/main/docs/images/ecsk.gif)

ecskはコンテナ（タスク）を取り扱うことに特化しているので、

- ECSサービスやタスク定義の管理 → CDKやTerraformなどを使用
- デバッグ → **ecsk**を使用 😁

などの利用を想定しています。

## インストール

### MacOS

```sh
brew install yukiarrr/tap/ecsk
```

### Linux

```sh
wget https://github.com/yukiarrr/ecsk/releases/download/v0.9.3/ecsk_Linux_x86_64.tar.gz
tar zxvf ecsk_Linux_x86_64.tar.gz
chmod +x ./ecsk
sudo mv ./ecsk /usr/local/bin/ecsk
```

### Windows

[Releases](https://github.com/yukiarrr/ecsk/releases)からダウンロードしてください。

## 使い方

ここでは、よく使うコマンドを紹介します。  
詳しいフラグについては、`ecsk [command] --help`を実行して確認してください。

### `ecsk run`

```sh
ecsk run
```

フラグを一切指定しない場合は、インタラクティブにタスク情報を入力した後、`docker run`と同じようにタスクが起動し終了するまでログが流れ続けます。
<br>
<br>

```sh
ecsk run -e -i --rm -c [container_name] -- /bin/sh
```

タスクを起動させた後、`execute-command`で指定したコマンドを実行します。  
合わせて`--rm`を指定することで、セッション終了時に自動でタスクが終了するため、踏み台サーバーのように運用することが可能になります。
<br>
<br>

```sh
ecsk run -d
```

インタラクティブにタスク情報を入力した後、タスクの開始・終了を待たずにコマンドを終了させます。

### `ecsk exec`

```sh
ecsk exec -i -- /bin/sh
```

インタラクティブにタスク・コンテナを選択し、コマンドを実行します。

### `ecsk cp`

```sh
ecsk cp ./ [container_name]:/etc/nginx/
```

インタラクティブにタスクを選択し、ローカルからリモートへファイルをコピーします。  
内部的にはS3 Bucketを使用してファイルを転送しているので、[タスクロールに該当Bucketのアクセス許可を追加する必要があります。](#ecsk-cpを使う場合)

なお、コンテナもインタラクティブに選択する場合は、`ecsk cp ./ :/etc/nginx/`としてください。
<br>
<br>

```sh
ecsk cp [container_name]:/var/log/nginx/access.log ./
```

リモートからローカルにファイルを転送します。

### `ecsk logs`

```sh
ecsk logs
```

インタラクティブにタスクを選択し、ログを表示します。  
このタスクは複数指定することができます。

なお、ログ表示は[knqyf263/utern](https://github.com/knqyf263/utern)を利用させていただいています。

### `ecsk stop`

```sh
ecsk stop
```

インタラクティブにタスクを選択し、終了します。

### `ecsk describe`

```sh
ecsk describe
```

インタラクティブにタスクを選択し、詳細情報を表示します。  
また、タスク一覧を確認する用途としても利用できます。

## 前提条件

### `ecsk exec`を使う場合

内部で`execute-command`を実行しているため、いくつかの前提条件があります。  
ここでは、[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/ecs-exec.html)を参考に、必須項目を紹介します。

#### Session Manager pluginをインストール

下記を参考にしてください。

https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

#### SSMのアクセス許可を追加

SSMエージェントとSSMサービス間の通信に必要なアクセス許可を追加する必要があります。

```json
{
   "Version": "2012-10-17",
   "Statement": [
       {
       "Effect": "Allow",
       "Action": [
            "ssmmessages:CreateControlChannel",
            "ssmmessages:CreateDataChannel",
            "ssmmessages:OpenControlChannel",
            "ssmmessages:OpenDataChannel"
       ],
      "Resource": "*"
      }
   ]
}
```

#### ECS Execの有効化

すでに作成されているサービスのタスクで`execute-command`するためには、ECS Execを有効化する必要があります。  
AWS CLIであれば`--enable-execute-command`フラグを、CFnであれば`EnableExecuteCommand`を追加してください。

なお、`ecsk run`で起動するタスクに関しては、`-e`か`--enable-execute-command`フラグを使用してください。

#### 補足

これらのように、前提条件が多めとなっているので、ecskではエラー時に[aws-containers/amazon-ecs-exec-checker](https://github.com/aws-containers/amazon-ecs-exec-checker)を実行するようにしています。

### `ecsk cp`を使う場合

ファイルの受け渡しにS3 Bucketを用いているため、タスクロールに該当Bucketのアクセス許可を追加する必要があります。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::[bucket_name]",
                "arn:aws:s3:::[bucket_name]/ecsk_*"
            ]
        }
    ]
}
```
