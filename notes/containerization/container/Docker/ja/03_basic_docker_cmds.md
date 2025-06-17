# Docker 基本コマンド演習

[English](../en/03_basic_docker_cmds.md) | [繁體中文](../zh-tw/03_basic_docker_cmds.md) | [日本語](../ja/03_basic_docker_cmds.md) | [インデックスに戻る](../README.md)

## 環境セットアップ
まず、Amazon Linux 2 環境であることを確認します：
```bash
$ cat /etc/os-release
NAME="Amazon Linux"
VERSION="2"
ID="amzn"
ID_LIKE="centos rhel fedora"
VERSION_ID="2"
PRETTY_NAME="Amazon Linux 2"
ANSI_COLOR="0;33"
CPE_NAME="cpe:2.3:o:amazon:amazon_linux:2"
HOME_URL="https://amazonlinux.com/"
SUPPORT_END="2026-06-30"
```

## Docker のインストールと設定
1. Docker のインストール：
```bash
$ sudo yum install docker -y
$ sudo docker --version
```

2. 現在のユーザー確認と Docker サービスの起動：
```bash
# 現在のユーザー名が ssm-user と仮定
$ whoami
$ sudo service docker start
$ sudo service docker status | grep Active
```

3. Docker の権限設定：
```bash
$ ls -l /var/run/docker.sock
# 毎回 sudo を使用せずに済むようにユーザーを docker グループに追加
# usermod -aG：ユーザーをグループに追加（-a は追加、-G はグループ指定）
# newgrp：ログアウトせずに新しいグループに即時切り替え
$ sudo usermod -aG docker ssm-user && newgrp docker
```

## 基本的な Docker 操作
1. コンテナの状態確認と Docker のテスト：
```bash
$ docker ps
$ docker pull hello-world
$ docker images
$ docker run hello-world
```

2. Docker サービスの停止：
```bash
$ sudo service docker stop
$ sudo service docker status | grep Active
```

3. イメージの削除：
```bash
# 関連するすべてのコンテナを停止して削除してから実行してください
$ docker rmi alpine:3.16 
```

## カスタムイメージの作成
1. Dockerfile の作成：
```bash
$ vi Dockerfile
```

Dockerfile の内容：
```dockerfile
FROM alpine:3.16
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
ENTRYPOINT ["httpd", "-D", "FOREGROUND"]
```

2. カスタムイメージのビルドと実行：
```bash
$ docker build -t myimage .
$ docker images 
$ docker run -d -p 8081:80 --name container002 myimage 
$ docker container ls 
```
- 完了後、ブラウザで `[ec2_instance_public_id]:8081` にアクセスして結果を確認できます
- アクセスできない場合は、通常 EC2 セキュリティグループのインバウンドルールが設定されていないためです。下図のように設定してください
<img src="../images/03_sg_settings.jpg" width="600" />

## Docker Hub の操作
1. イメージを Docker Hub にプッシュ：
```bash
$ docker build -t your_docker_hub_account/myimage .
$ docker images
$ docker logout
$ docker login
$ docker push your_docker_hub_account/myimage
```

完了後、[Docker Hub](https://hub.docker.com/) でアップロードしたイメージを確認できます 

## Docker Volume の操作
1. Volume の作成と確認：
```bash
# すべての volume を一覧表示
$ docker volume ls

# 新しい volume を作成
$ docker volume create v001

# volume の一覧を再確認
$ docker volume ls

# volume の詳細情報を確認
$ docker volume inspect v001
```

2. コンテナへの Volume マウント：
```bash
# volume をコンテナの /var/www/localhost/htdocs/ ディレクトリにマウントして起動
$ docker run -d -p 8081:80 --name c001 -v v001:/var/www/localhost/htdocs/ myimage 
```

3. コンテナ内での Volume 操作：
```bash
# コンテナに入る
$ docker exec -it c001 /bin/sh

# マウントされたディレクトリに移動
$ cd /var/www/localhost/htdocs/

# ディレクトリの内容を確認
$ ls

# index.html の内容を確認
$ cat index.html

# index.html の内容を変更
$ echo "<h2>learning docker now!</h2>" >> index.html

# 変更内容を確認
$ cat index.html

# コンテナから出る
$ exit
```
ブラウザで結果を確認できます

4. Volume の永続性テスト：
```bash
# コンテナを停止して削除
$ docker container stop c001
$ docker container rm c001

# 同じ volume でコンテナを再起動
$ docker run -d -p 8081:80 --name c001 -v v001:/var/www/localhost/htdocs/ myimage 
$ docker ps
```
ブラウザでデータが保持されていることを確認できます

5. リソースのクリーンアップ：
```bash
# コンテナを停止して削除
$ docker container stop c001
$ docker container rm c001

# volume を削除
$ docker volume rm v001

# volume が削除されたことを確認
$ docker volume ls
``` 