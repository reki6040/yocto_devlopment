# Dockerを使用した開発と運用環境

## アジェンダ

- Docker環境の構築
- 開発環境構築
- 運用環境構築
- CI/CD環境構築

## Docker環境の構築

- Dockerのメリットは以下
- アプリケーションごとの動作環境の分割[いろいろなアプリケーション入れていると何がそのアプリケーションに必要なのか資料を残しわすれるとわからなくなってくる]
- 開発者ごとの環境差分の吸収[いわゆる、私の環境では動作するのになぜXXXさんの環境では動作しないの...がなくなる]
- 複数のアプリケーションの環境を壊さないでいられる。
- ほかのマシンへ移行しやすい
- マシンリソースが仮想化より小さい
- 仮想化より構築時間が短くできる
- コマンドライン、テキストファイルでも構築なのでCI/CDの環境構築やほかのTOOL類と連携しやすい
- まったく同じ環境が別マシンで実現可能

- Dockerのデメリットは以下
- コマンドライン、テキストファイルでの構築なので知識が必要[GUIでも知識は必要だが人によっては毛嫌いする人がいる...]
- コンテナ上のTerminalで変更は再起動時保持されない[これは環境の構築のメリットでもあるが、知っておかないと仮想マシンではできていたことができないとかいわれる可能性あり]
- プロキシの環境が慣れていないとドはまりする可能性あり。

### 動作環境

- OS:Ubuntsu 20.04 LTS
- マシン:VirutulBox 6.1.22
- Memory : 32GB[用途によっては減らしてOK]
- ストレージ:200GB[DockerのImageとレジストリがあるので多めにとっている。]

### OSのパッケージのインストール

```shell:Linux Command
sudo apt-get update -y && sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common \
&& curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - \
&& sudo apt-key fingerprint 0EBFCD88 \
&& sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable" \
&& sudo apt-get update \
&& sudo apt-get install docker-ce docker-ce-cli containerd.io
```

### 動作確認

- DockerのHerroworldで動作確認
  
```shell:Linux Command

reki@reki-VM:~$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
b8dfde127a29: Pull complete
Digest: sha256:df5f5184104426b65967e016ff2ac0bfcd44ad7899ca3bbcf8e44e4461491a9e
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

reki@reki-VM:~$
```

↑が表示さればOK

### Dockerをsudo使用しなくてもよいことにする

```shell:Linux Command
$ sudo groupadd docker
groupadd: グループ 'docker' は既に存在します
$ sudo usermod -aG docker $USER
$newgrp docker
一度抜ける
$exit
再度ログイン後はdockerコマンドがsudoなしで可能。
$docker run hello-world

```

### 複数のコンテナを使用する場合

```shell:Linux Command
sudo apt install docker-compose
```

### DockerのImageをほかのディレクトリにする場合[主にサーバマシンの起動DiskとデータDiskを分ける場合]

docker infoのDocker Root Dirに現在のImageの場所がある。

```shell:Linux Command
$ docker info | grep  "Docker Root"
 Docker Root Dir: /var/lib/docker

以下変更方法
$ sudo service docker stop
$ mkdir /foo/bar/docker
もしくは以下のように別ドライブをシンボリックリンクで参照してもOK！
Virtulboxの共有ディレクトリにするなら、権限とかもいるので以下参照[これは試したが、NTFSの都合上Dockerの書くファイルの文字[小文字大文字はWindows識別できない]や権限の都合でエラー出まくるのでNG。WindowsでLinuxのディスクをマウントできるまで待つ必要がある。]また、Virtulboxの共有ディレクトリの設定で/etc/docker/daemon.jsonに設定を残して置くと、dockerのdemonが起動しないのでトラブルになるので、削除しておこう。

sudo gpasswd -a $USER vboxsf
sudo chmod 777 -R /mnt/Share
sudo chmod 777 -R /mnt/Share/Docker
ln -s /mnt/Share/Docker /home/reki/docker
ln -s /foo/bar/docker /mnt/bar/docker
いったんReboot
$ sudo vi /etc/docker/daemon.json

/etc/docker/daemon.json
{
    "graph": "/foo/bar/docker"
}

データ移行が必要な場合は移動先へコピー
/var/lib/docker/　-> /foo/bar/docker
$ sudo rsync -aqxP /var/lib/docker/ /foo/bar/docker
sudo rsync -aqxP /var/lib/docker /home/reki/docker/varlibdocker

$ sudo service docker start
$ docker info | grep 'Docker Root Dir'
Docker Root Dir: /foo/bar/docker

```

参考URL

[Docker Root Dir を /var/lib/docker から /foo/bar/docker へ変更する方法](https://qiita.com/moperon/items/cda5b7d6e9ad39f796d5#:~:text=2017%2D05%2D07-,Docker%20Root%20Dir%20%E3%82%92%20%2Fvar%2Flib%2Fdocker%20%E3%81%8B%E3%82%89%20%2F,docker%20%E3%81%B8%E5%A4%89%E6%9B%B4%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95&text=linux%E3%81%AE%E5%A0%B4%E5%90%88%20%2Fetc%2Fdocker,%E8%A8%AD%E5%AE%9A%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E7%B7%A8%E9%9B%86%E3%81%99%E3%82%8B%E3%80%82)

### DockerのBase Imageを作成する場合[Ubntsu]

- 目的:オリジナルのimageを作成するためや公式imageを取れないケース、カスタムしたい場合などが考えられる。あえて一からBase Imageを作成してみることも良い勉強になるのでやってみる。

```shell:Linux Command

cd　/home/reki
sudo debootstrap --verbose --variant=minbase --arch=amd64 focal focal > /dev/null
↑はamd64のアーキのminibase[最小イメージ]のDocker Imageファイル

cdのディレクトリ:focalをtarで固めてdockerへimport[2021/07/27では120MBくらいある]
sudo tar -C focal -c . | docker import - focal

試しにdocker起動してOSのバージョンなど表示
docker run focal cat /etc/lsb-release

アレンジ
sudo tar -C focal -c . | docker import  --message "New image imported from tarball" - focal:new

アレンジ2[gz圧縮している場合]
sudo tar xf focal.tgz && sudo tar -C focal -c . | docker import  --message "New image imported from tarball" - focal:new && sudo rm -rf focal

reki@reki-VM:~$ docker image ls
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
focal        new       a50dbf79d092   5 minutes ago   113MB

mkdir -p /home/reki/Dck/focal_test
cd　/home/reki/Dck/focal_test
以下のエラー出力:どうやらdockerのグループない模様。
usermod: group 'docker' does not exist
ERROR: Service 'jenkins' failed to build: The command '/bin/sh -c usermod -aG docker jenkins' returned a non-zero code: 6

以下は実行可能なので、調査に使用する。
docker run --name focal_test_focal_test_1 -i -t focal:new /bin/bash

↑を実行後、パッケージも足りないことが判明したので
まず、aptのsource.listに何が設定されているか確認する。
root@7576730898f5:/# cat /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu focal main

focalのデスクトップ版は以下なので同じように変更する。
reki@reki-VM:~$ cat /etc/apt/sources.list
deb http://jp.archive.ubuntu.com/ubuntu/ focal main restricted
deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates main restricted
deb http://jp.archive.ubuntu.com/ubuntu/ focal universe
deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates universe
deb http://jp.archive.ubuntu.com/ubuntu/ focal multiverse
deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates multiverse
deb http://jp.archive.ubuntu.com/ubuntu/ focal-backports main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu focal-security main restricted
deb http://security.ubuntu.com/ubuntu focal-security universe
deb http://security.ubuntu.com/ubuntu focal-security multiverse

install
echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal main restricted" > /etc/apt/sources.list \
&& echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates main restricted" >> /etc/apt/sources.list \
&& echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal universe" >> /etc/apt/sources.list \
&& echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates universe" >> /etc/apt/sources.list \
&& echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal multiverse" >> /etc/apt/sources.list \
&& echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates multiverse" >> /etc/apt/sources.list \
&& echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-backports main restricted universe multiverse" >> /etc/apt/sources.list \
&& echo "deb http://security.ubuntu.com/ubuntu focal-security main restricted" >> /etc/apt/sources.list \
&& echo "deb http://security.ubuntu.com/ubuntu focal-security universe" >> /etc/apt/sources.list \
&& echo "deb http://security.ubuntu.com/ubuntu focal-security multiverse" >> /etc/apt/sources.list

apt update -y
apt upgrade -y

dockerのグループ追加のためにdockerもインストールする。
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    lsb-release \
    && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
    && echo \
      "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io



Dockerでユーザを追加したい場合[アプリケーションによってはrootではなく一般ユーザで動作することを期待する場合もあるため]
以下はuserというユーザを追加する例
パスワードはuser1
groupadd -g 1000 user && \
    useradd -m -s /bin/bash -u 1000 -g 1000 -G sudo user && \
    echo user:user1 | chpasswd && \
    echo "user   ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

以下のコマンドでログイン
su user

Dockerfileの作成
vi Dockerfile
# ベースイメージ[ローカルで作成したイメージを指定]
FROM focal:new

# aptのパッケージのリストを設定+Dockerのグループ追加のためにパッケージのインストール
# Dockerのインストールは「https://docs.docker.com/engine/install/ubuntu/」参照
RUN echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal main restricted" > /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates main restricted" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal universe" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates universe" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal multiverse" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates multiverse" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-backports main restricted universe multiverse" >> /etc/apt/sources.list \
    && echo "deb http://security.ubuntu.com/ubuntu focal-security main restricted" >> /etc/apt/sources.list \
    && echo "deb http://security.ubuntu.com/ubuntu focal-security universe" >> /etc/apt/sources.list \
    && echo "deb http://security.ubuntu.com/ubuntu focal-security multiverse" >> /etc/apt/sources.list \
    && apt-get update -y && apt-get upgrade -y && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    lsb-release \
    && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
    && echo \
      "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

#ユーザ追加
ARG USERNAME=user
ARG GROUPNAME=user
ARG UID=1000
ARG GID=1000
ARG PASSWORD=user
RUN groupadd -g $GID $GROUPNAME && \
    useradd -m -s /bin/bash -u $UID -g $GID -G sudo $USERNAME && \
    echo $USERNAME:$PASSWORD | chpasswd && \
    echo "$USERNAME   ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

#ユーザをdockerに追加
RUN usermod -aG docker $USERNAME

#ここに実行させたいコマンドを追記[CMDはDockerfileに1つだけ有効]
CMD ["/bin/bash"]

#作成したユーザに切り替え[デフォルトのログインはrootではなくユーザになる]
USER $USERNAME

とりあえず、作成したイメージを常駐させる
vi docker-compose.yml
version: "3"

services:
    focal_test:
        build: .
            
ls
docker-compose.yml  Dockerfile

docker-compose.ymlとDockerfileからイメージを作成
docker-compose up --build -d

作成したイメージを確認
docker image ls
REPOSITORY              TAG       IMAGE ID       CREATED          SIZE
focal_test_focal_test   latest    c8a1eacc35e3   34 seconds ago   773MB
focal                   new       a50dbf79d092   3 days ago       113MB

dockerコンテナの起動結果[まだ起動していないのでExited]
docker ps -a
CONTAINER ID   IMAGE                   COMMAND       CREATED          STATUS                      PORTS     NAMES
45350665616d   focal_test_focal_test   "/bin/bash"   37 seconds ago   Exited (0) 36 seconds ago             focal_test_focal_test_1

コンテナ:focal_test_focal_test_1を削除
docker rm focal_test_focal_test_1

コンテナイメージ:focal_test focal_test_focal_test:latestからコンテナ名:ocal_testでstopするまでとりあえず起動[pseudo-tty付きのBashを走らせればOK]
docker run -itd --name focal_test focal_test_focal_test:latest bash -c "bash --rcfile <(echo \"trap 'exit 0' TERM\")"

常駐確認
docker ps -a
CONTAINER ID   IMAGE                          COMMAND                  CREATED         STATUS         PORTS     NAMES
8ad85143a3c1   focal_test_focal_test:latest   "bash -c 'bash --rcf…"   2 minutes ago   Up 2 minutes             focal_test

dockerへログイン[root]
docker exec -i -t -u root focal_test bash

dockerへログイン[defaultユーザ]
docker exec -i -t focal_test bash

dockerのコンテナを自動起動させる
docker update --restart=always focal_test
dockerのコンテナを自動起動をやめさせる
docker update --restart=no focal_test

```

参考URL
[ベースイメージを作成する](https://docs.docker.com/develop/develop-images/baseimages/)
[dockerのカスタムベースイメージを作成する](https://blog.n-z.jp/blog/2013-12-13-docker-custom-base-image.html)
[UbuntuにDockerエンジンをインストールする](https://docs.docker.com/engine/install/ubuntu/)
[ただ動くだけのdockerコンテナを作る](https://doloopwhile.hatenablog.com/entry/2014/09/12/184725)
[Dockerコンテナの自動起動設定を変更する方法](https://www.pressmantech.com/tech/6522)

### 実験:DockerでGo環境で動作確認

- DockerでGo環境作成し、ARM[32bit]クロスコンパイルと動作確認

- CGOでクロスコンパイル先の機能を使用したクロスコンパイルと動作確認

- buildrootへ統合
  
## 開発環境構築

- Dockerコンテナのレジストリ
  Dockerコンテナのイメージを格納しておくストレージ兼構成管理サーバ、自前のサーバをDockerレジストリにもできるが、近年GithubやGitlabなどでも利用できるようになっている。
  本家はDocker Hubだけど、無料枠は一アカウント一レジストリのみなので無料ではできる枠に制限がある状態。

### Dockerコンテナのレジストリの構築

#### プライベートDockerレジストリサーバーの構築方法

- 目的：インターネット上の外部公開ではなく会社内で開発する時だけの構築をしたい場合、プライベートクラウド上で自前で行う場合を想定している

##### 手段1:Docker上でそのままプライベートDockerレジストリサーバをコンテナ起動する

- 目的:ローカル環境用にDockerレジストリサーバを立ててImageバックアップできるようにする。

- やり方:Dockerレジストリ用のimageをDLしてローカルへ展開...

##### 手段2:NAS[ASUSTOR]上でそのままプライベートDockerレジストリサーバをコンテナ起動する

- 目的:お手軽にローカルのプライベートリポジトリ作成する。
  しかし、常時通電させるのは電気代もったいないので、基本的にはDocker用のＶＭ上でそのまま立てて、共有ディレクトリしてＨｏｓｔ側から直接アクセスできるようにしておいたほうが良いかも,バックアップ面倒。

- やり方:あらかじめ対応しているNASを購入する。
例.
[Asustor AS1002Tv2-DockerとPortainer](https://www.reddit.com/r/asustor/comments/kcz744/asustor_as1002t_v2_issues_with_docker_and/)
↑の記事で存在を確認、あとはNASのNASのWebUIからDownload＆Installして起動。

#### Gitlab上でコンテナレジストリを構築する方法

### Dockerコンテナのファイル作成

### docker-composeのymlファイルの作成

- Dockerコンテナのファイルとの違い
  docker-composeは複数のDockerコンテナを生成と制御できる
　例えば,Dockerコンテナで開発環境で1つ,試験環境で1つ,開発と試験のシナリオの制御・ログサーバで1つの計:3つでソフト開発環境も作成できる。
また、Webサービスで複数環境の連携させるような構築もできる。
 ECサイトのように顧客から注文受けたら、卸売り業者のサーバへ見積書発注、受注後配送業者へ自動発注といったサービス構成もできるが、従来はコンテナではなく仮想マシンベースでやっていた処理をリソースの少ないコンテナベースにもできる。

## 運用環境構築

## CI/CD環境構築
