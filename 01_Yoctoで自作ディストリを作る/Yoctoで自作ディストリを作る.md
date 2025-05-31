
# 1.Yoctoで自作ディストリを作る

- 目的:Yocto Projectは組み込みLinuxを作成する基盤であるOpenSourceですが、PC用のディストリビューションの作成も可能です。

## 1.1.アジェンダ

- [1.Yoctoで自作ディストリを作る](#1yoctoで自作ディストリを作る)
  - [1.1.アジェンダ](#11アジェンダ)
    - [1.1.1.YoctoのInstall](#111yoctoのinstall)
      - [1.1.1.1.Yocto環境をDocker Imageで作成する](#1111yocto環境をdocker-imageで作成する)
      - [1.1.1.2.Docker Imageをコンテナとして起動し、Hostのディレクトリをマウントし、ssh接続できるようにYocto環境を構築する](#1112docker-imageをコンテナとして起動しhostのディレクトリをマウントしssh接続できるようにyocto環境を構築する)
    - [1.1.2.YoctoのBuild](#112yoctoのbuild)
    - [1.1.2.1.パターン1 x86-64のQEMU用のイメージ作成\[Yoctoバージョン:styhead\]](#1121パターン1-x86-64のqemu用のイメージ作成yoctoバージョンstyhead)
    - [1.1.2.2.パターン2 x86-64の汎用イメージ作成\[Yoctoバージョン:styhead\]](#1122パターン2-x86-64の汎用イメージ作成yoctoバージョンstyhead)
    - [1.1.2.3.パターン3 x86-64の汎用イメージ作成\[Yoctoバージョン:Scarthgap\]](#1123パターン3-x86-64の汎用イメージ作成yoctoバージョンscarthgap)
    - [1.1.2.4.パターン4 rasberry piの汎用イメージ作成\[Yoctoバージョン:Scarthgap\]](#1124パターン4-rasberry-piの汎用イメージ作成yoctoバージョンscarthgap)
    - [1.1.3.Yoctoの初期設定](#113yoctoの初期設定)
      - [1.1.3.1.初期ユーザパスワード設定とOpenSSHサーバのInstall](#1131初期ユーザパスワード設定とopensshサーバのinstall)
      - [1.1.3.2.Network設定](#1132network設定)
      - [1.1.3.3.USB-WIFIをNetwork追加する](#1133usb-wifiをnetwork追加する)
      - [1.1.3.4.USB-WIFIとLAN Network BrigeしてEthernet Converter化](#1134usb-wifiとlan-network-brigeしてethernet-converter化)
        - [1.1.3.4.1.WIFIで日本用の設定可能にする](#11341wifiで日本用の設定可能にする)
      - [1.1.3.5.不揮発性ディスク→ramdisk化して起動する](#1135不揮発性ディスクramdisk化して起動する)

### 1.1.1.YoctoのInstall

- 目的：最新のYocto Projectをインストールする
- Yocto Projectのインストール方法は[こちら>GET STARTEDを押下](https://www.yoctoproject.org/)

やり方

- 1.Yocto環境をDocker Imageで作成する
- 2.Docker Imageをコンテナとして起動し、Hostのディレクトリをマウントし、ssh接続できるようにYocto環境を構築する

#### 1.1.1.1.Yocto環境をDocker Imageで作成する

まずDocker Imageの元を作成する

Yoctoの環境をサポートするディストリビューションは以下を参照する。

- [1.サポートされているLinuxディストリビューション](https://docs.yoctoproject.org/ref-manual/system-requirements.html#supported-linux-distributions)

↑より今回は[Ubuntu 24.04 (LTS)]を選択した。

Docker Imageを今回自作するが、公式Imageを利用するすべもあるのでいかに記載しておく
以下のDockerfileの先頭に記載する

``` Dockerfile
FROM ubuntu:24.04
```

以下が自作するパターンです。
まずDockerのHostマシンへログインした状態で、以下のコマンドを実行する

``` bash
#必要なツールinstall
#debootstrap をホストマシンにインストールします（Ubuntu環境の場合
sudo apt update && sudo apt install -y debootstrap

#管理者になる
su

# 最小限のUbuntu環境を構築
#次に debootstrap を使って、Ubuntu 24.04 のルートファイルシステム (rootfs) を作成します
#最小限のUbuntu環境を構築
debootstrap --variant=minbase --arch=amd64 noble ./ubuntu_noble_rootfs_mini http://archive.ubuntu.com/ubuntu/

#不要なファイルの削除
sudo rm -rf ./ubuntu_noble_rootfs_mini/var/cache/apt/*
sudo rm -rf ./ubuntu_noble_rootfs_mini/usr/share/doc/*
sudo rm -rf ./ubuntu_noble_rootfs_mini/usr/share/man/*

#実行結果
I: Retrieving InRelease
I: Checking Release signature
I: Valid Release signature (key id F6ECB3762474EDA9D21B7022871920D1991BC93C)
・
・略
・
I: Configuring libc-bin...
I: Configuring ca-certificates...
I: Base system installed successfully.

#意味
#- noble は Ubuntu 24.04 のコードネームです。
#- ./ubuntu-rootfs は作成するルートファイルシステムの保存先。
#- http://archive.ubuntu.com/ubuntu/ は公式のUbuntuパッケージリポジトリ。

#Downloadしたサイズ確認
# du -sh ubuntu_noble_rootfs_mini
94M     ubuntu_noble_rootfs_mini

#ルートファイルシステムを Docker イメージ化
cd ubuntu_noble_rootfs_mini

tar -cvf - . | gzip -9 > ../ubuntu_noble_rootfs_mini.tar.gz

#結果
ubuntu_noble_rootfs_mini/
ubuntu_noble_rootfs_mini/srv/
ubuntu_noble_rootfs_mini/bin.usr-is-merged/
・
・略
・
ubuntu_noble_rootfs_mini/mnt/
ubuntu_noble_rootfs_mini/bin
ubuntu_noble_rootfs_mini/home/

cd ../

gunzip -c ubuntu_noble_rootfs_mini.tar.gz | docker import - ubuntu-noble-rootfs-mini:v1.0

#結果
sha256:a26bc93c5c70beb9883abb16afaba58307d9f8f279ad57010b7b1f234c7a7ff3

#イメージの確認
docker images
ubuntu-noble-rootfs-mini   v1.0             008716d5e023   12 seconds ago   89.7MB

#イメージの動作確認
docker run -it ubuntu-noble-rootfs-mini:v1.0 /bin/bash

#これで debootstrap で作成した軽量なUbuntu環境がDockerコンテナとして動作します！ 

```

#### 1.1.1.2.Docker Imageをコンテナとして起動し、Hostのディレクトリをマウントし、ssh接続できるようにYocto環境を構築する

続いてDockerfileへ記載する

``` Dockerfile
cd /home/reki/Dck
mkdir -p yocto_wk
cd yocto_wk
cat <<EOF > Dockerfile
# 開発用のYocto3.3のDockerマシン
# ベースイメージ[ローカルで作成したイメージを指定]
FROM ubuntu-noble-rootfs-mini:v1.0
# 各環境変数を設定
ENV USER reki
ENV USER_PASSWD a278354
ENV HOME /home/\${USER}
ENV SHELL /bin/bash
# aptのパッケージinstall
#[https://docs.yoctoproject.org/ref-manual/system-requirements.html#ubuntu-and-debian]参照
RUN apt update -y && apt upgrade -y && apt autoremove --purge -y && \\
apt install -y software-properties-common && \\
add-apt-repository universe && \\
    apt install -y openssh-server sudo \\
    build-essential chrpath cpio debianutils diffstat file gawk gcc git iputils-ping libacl1 liblz4-tool locales python3 python3-git python3-jinja2 python3-pexpect python3-pip python3-subunit socat texinfo unzip wget xz-utils zstd

#&& \
#apt-get clean && rm -rf /var/lib/apt/lists/*

# Locales
RUN echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen && \\
DEBIAN_FRONTEND=noninteractive dpkg-reconfigure locales && \\
DEBIAN_FRONTEND=noninteractive locale-gen en_US.UTF-8 && \\
DEBIAN_FRONTEND=noninteractive update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

ENV LC_ALL   en_US.UTF-8 && \\
    LANG     en_US.UTF-8 && \\
    LANGUAGE en_US.UTF-8

# Create User
RUN useradd -m \${USER} && \\
    gpasswd -a \${USER} sudo \\
    && echo "\${USER}:\${USER_PASSWD}" | chpasswd \\
    && usermod -aG sudo \${USER} \\
    && chsh -s \${SHELL} \${USER} \\
    && sed -i.bak -e "\${HOME}:\${HOME}:\${SHELL}" /etc/passwd \\
    && echo 'Defaults visiblepw'             >> /etc/sudoers \\
    && echo '\$HOME ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

#SSH
RUN mkdir -p /var/run/sshd \\
    && echo 'root:root' | chpasswd \\
    && sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config \\
    && sed -i 's/#PasswordAuthentication/PasswordAuthentication/' /etc/ssh/sshd_config \\
    && sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd \\
    && ssh-keygen -t rsa -N "" -f /etc/ssh/ssh_host_rsa_key -y \\
    && ssh-keygen -t dsa -N "" -f /etc/ssh/ssh_host_dsa_key \\
    && ssh-keygen -t ecdsa -N "" -f /etc/ssh/ssh_host_ecdsa_key -y \\
    && ssh-keygen -t ed25519 -N "" -f /etc/ssh/ssh_host_ed25519_key -y \\
    && mkdir -p /root/.ssh \\
    && mkdir -p \$HOME/.ssh

EXPOSE 22

ENTRYPOINT [ "/usr/sbin/sshd", "-D"]

# 以降の作業ディレクトリを指定
WORKDIR \${HOME}

EOF

```

docker-compose.ymlへ記載する

``` docker-compose.yml
cd yocto_wk
cat <<EOF > docker-compose.yml
#実行するコマンド:$ docker-compose up --build -d
#
# 注意:テキストの保存はUTF-8をすること。
version: "3"

services:
    yocto_wk:
        build: .
# コンテナ名を明示的に指定する
        container_name: yocto_wk
# コンテナのhostnameを明示的に設定する
        hostname: 'yocto_wk'
#公開用のPort
        ports:
            - "2221:22"
        tty: true
#ボリューム[ストレージ]
        volumes:
            #作業用のワークディレクトリ
            - "./vol:/home/reki/vol"
            #Hostと時刻同期
            #- "/etc/timezone:/etc/timezone:ro"
            #- "/etc/localtime:/etc/localtime:ro"

EOF

```

docker コンテナを作成する

``` bash
cd yocto_wk
docker-compose up --build -d

#起動後のsshログインをhost上で試す
ssh -p 2221 reki@127.0.0.1
#結果
reki@yocto_wk:~$

```

### 1.1.2.YoctoのBuild

- [Gitを使ってPokyをクローンする](https://docs.yoctoproject.org/5.1.4/brief-yoctoprojectqs/index.html)

### 1.1.2.1.パターン1 x86-64のQEMU用のイメージ作成[Yoctoバージョン:styhead]

``` sh

#build用のWork Directoryを作成
$mkdir -p /home/reki/vol/x86_64
#build用のWork Directoryへ移動
$cd /home/reki/vol/x86_64
#git cloneする
$git clone https://git.yoctoproject.org/poky
#cloneしたDirectoryへ入る
$ cd poky
#gitのbranchを確認する今回はYocto 5.1.4のstyheadを見る
$ git branch -a
#styheadをチェックアウトしてmy-styheadとしてbranchを切る。
$git checkout -t origin/styhead -b my-styhead

#pokyDirectoryから抜ける
cd ../

#oe-init-build-env 環境セットアップ スクリプトを実行
$source poky/oe-init-build-env

#ちなみに通常TOP_DIRはbuildだけど、以下のようにBUILDDIRを変更するとTOPDIRの変数を変更できる。
export BUILDDIR=/home/reki/vol/x86_64/build
source poky/oe-init-build-env $BUILDDIR


$vi conf/local.conf
SSTATE_DIR ?= "${TOPDIR}/../sstate-cache"
DL_DIR ?= "${TOPDIR}/../downloads"

BB_HASHSERVE_UPSTREAM = "wss://hashserv.yoctoproject.org/ws"
SSTATE_MIRRORS ?= "file://.* http://cdn.jsdelivr.net/yocto/sstate/all/PATH;downloadfilename=PATH"
BB_HASHSERVE = "auto"
BB_SIGNATURE_HANDLER = "OEEquivHash"

$bitbake core-image-minimal
#エラー対応.1
ERROR: quilt-native-0.68-r0 do_populate_lic_setscene: Fetcher failure: Unable to find file file://65/cb/sstate:quilt-native::0.68:r0::14:65cbfd32678bec1bcee007013b7975430424c599233c0eb27d0457972acfaa24_populate_lic.tar.zst.siginfo;downloadfilename=65/cb/sstate:quilt-native::0.68:r0::14:65cbfd32678bec1bcee007013b7975430424c599233c0eb27d0457972acfaa24_populate_lic.tar.zst.siginfo anywhere to download to /home/reki/vol/x86_64/poky/build/../sstate-cache/65/cb/sstate:quilt-native::0.68:r0::14:65cbfd32678bec1bcee007013b7975430424c599233c0eb27d0457972acfaa24_populate_lic.tar.zst.siginfo. The paths that were searched were:
    /home/reki/vol/x86_64/poky/build/../sstate-cache
#対応内容
# 1️⃣ キャッシュをクリアして再試行 OK
bitbake -c cleanall quilt-native
bitbake quilt-native

# 2️⃣ sstate-cache をリフレッシュ 未
rm -rf /home/reki/vol/x86_64/poky/sstate-cache
bitbake quilt-native

# 3️⃣ fetch の設定を確認　未
grep -E 'BB_FETCH_PREMIRROR|BB_FETCH_MIRROR' conf/local.conf

#エラー対応.2
ERROR: zlib-native-1.3.1-r0 do_deploy_source_date_epoch_setscene: Fetcher failure: Unable to find file file://universal/0e/88/sstate:zlib-native:x86_64-linux:1.3.1:r0:x86_64:14:0e8813404e01ad4d8267d7d5c36ccdffdf7cfc98b6ce83aa65ad5c2baf0f520f_deploy_source_date_epoch.tar.zst;downloadfilename=universal/0e/88/sstate:zlib-native:x86_64-linux:1.3.1:r0:x86_64:14:0e8813404e01ad4d8267d7d5c36ccdffdf7cfc98b6ce83aa65ad5c2baf0f520f_deploy_source_date_epoch.tar.zst anywhere to download to /home/reki/vol/x86_64/poky/build/../sstate-cache/universal/0e/88/sstate:zlib-native:x86_64-linux:1.3.1:r0:x86_64:14:0e8813404e01ad4d8267d7d5c36ccdffdf7cfc98b6ce83aa65ad5c2baf0f520f_deploy_source_date_epoch.tar.zst. The paths that were searched were:
    /home/reki/vol/x86_64/poky/build/../sstate-cache
#対応内容 OK
bitbake -c cleanall zlib-native
bitbake zlib-native

#buildしたimageは以下参照
/home/reki/vol/x86_64/build/tmp/deploy/images/qemux86-64/
bzImage
bzImage--6.10.14+git0+af06ad75b8_bbe3d1be4e-r0-qemux86-64-20250430122550.bin
bzImage-qemux86-64.bin
core-image-minimal-qemux86-64.rootfs-20250430122550.spdx.json
core-image-minimal-qemux86-64.rootfs-20250430234021.ext4
core-image-minimal-qemux86-64.rootfs-20250430234021.manifest
core-image-minimal-qemux86-64.rootfs-20250430234021.qemuboot.conf
core-image-minimal-qemux86-64.rootfs-20250430234021.tar.bz2
core-image-minimal-qemux86-64.rootfs-20250430234021.testdata.json
core-image-minimal-qemux86-64.rootfs.ext4
core-image-minimal-qemux86-64.rootfs.manifest
core-image-minimal-qemux86-64.rootfs.qemuboot.conf
core-image-minimal-qemux86-64.rootfs.spdx.json
core-image-minimal-qemux86-64.rootfs.tar.bz2
core-image-minimal-qemux86-64.rootfs.testdata.json
modules--6.10.14+git0+af06ad75b8_bbe3d1be4e-r0-qemux86-64-20250430122550.tgz
modules-qemux86-64.tgz

```

### 1.1.2.2.パターン2 x86-64の汎用イメージ作成[Yoctoバージョン:styhead]

``` sh

#build用のWork Directoryを作成
$mkdir -p /home/reki/vol/x86_64
#build用のWork Directoryへ移動
$cd /home/reki/vol/x86_64
#git cloneする
$git clone https://git.yoctoproject.org/poky
#cloneしたDirectoryへ入る
$ cd poky
#gitのbranchを確認する今回はYocto 5.1.4のstyheadを見る
$ git branch -a
#styheadをチェックアウトしてmy-styheadとしてbranchを切る。
$git checkout -t origin/styhead -b my-styhead

#pokyDirectoryから抜ける
cd ../

#oe-init-build-env 環境セットアップ スクリプトを実行
$source poky/oe-init-build-env

#ちなみに通常TOP_DIRはbuildだけど、以下のようにBUILDDIRを変更するとTOPDIRの変数を変更できる。
export BUILDDIR=/home/reki/vol/x86_64/build_genx86_64
source poky/oe-init-build-env $BUILDDIR


$vi conf/local.conf
MACHINE = "genericx86-64"

DL_DIR ?= "${TOPDIR}/../downloads"
SSTATE_DIR ?= "${TOPDIR}/../sstate-cache"

BB_HASHSERVE_UPSTREAM = "wss://hashserv.yoctoproject.org/ws"
SSTATE_MIRRORS ?= "file://.* http://cdn.jsdelivr.net/yocto/sstate/all/PATH;downloadfilename=PATH"
BB_HASHSERVE = "auto"
BB_SIGNATURE_HANDLER = "OEEquivHash"

BB_NUMBER_THREADS ?= "8"
PARALLEL_MAKE = "-j 10"

$bitbake core-image-minimal

#buildしたimageは以下参照
ls /home/reki/vol/x86_64/build_genx86_64/tmp/deploy/images/genericx86-64
addonx64.efi.stub
bzImage
bzImage--6.6.21+git0+2b3144e072_06644f0d71-r0-genericx86-64-20250501000249.bin
bzImage-genericx86-64.bin
core-image-minimal.env
core-image-minimal-genericx86-64.rootfs-20250501000249.manifest
core-image-minimal-genericx86-64.rootfs-20250501000249.qemuboot.conf
core-image-minimal-genericx86-64.rootfs-20250501000249.spdx.json
core-image-minimal-genericx86-64.rootfs-20250501000249.testdata.json
core-image-minimal-genericx86-64.rootfs-20250501000249.wic
core-image-minimal-genericx86-64.rootfs-20250501000249.wic.bmap
core-image-minimal-genericx86-64.rootfs.manifest
core-image-minimal-genericx86-64.rootfs.qemuboot.conf
core-image-minimal-genericx86-64.rootfs.spdx.json
core-image-minimal-genericx86-64.rootfs.testdata.json
core-image-minimal-genericx86-64.rootfs.wic
core-image-minimal-genericx86-64.rootfs.wic.bmap
core-image-minimal-genericx86.wks
grub-efi-bootx64.efi
linuxx64.efi.stub
modules--6.6.21+git0+2b3144e072_06644f0d71-r0-genericx86-64-20250501000249.tgz
modules-genericx86-64.tgz
systemd-bootx64.efi

#wicを直接USB[例./dev/sdbとした場合]へWriteする
wic write /home/reki/vol/x86_64/build_genx86_64/tmp/deploy/images/genericx86-64core-image-minimal-genericx86-64.rootfs-20250501000249.wic /dev/sdb
#もしくはddでもOKです。
dd if=tmp/deploy/images/genericx86-64/core-image-minimal-genericx86-64.rootfs.wic of=/dev/sdb bs=4M status=progress
sync
#windowsの場合はWin32DiskImagerを使うと便利です。ちなみにXG135は起動しなかった。BIOSがefi未対応？→DefaultはLegacy=BIOSだった。
#再度UEFIに変更してUSBからBootさせると起動しました。NICも認識されている。

```

### 1.1.2.3.パターン3 x86-64の汎用イメージ作成[Yoctoバージョン:Scarthgap]

Scarthgapにする理由はLTS[Long Term Support(つまりはソフトウェアの特定のバージョンを長期間にわたって安定的にサポートすることを指す)]のため、こちらの方が長期に利用するにはうってつけです。
Scarthgapは下記より2028/4までSupportしています。
<https://www.yoctoproject.org/development/releases/>

``` sh

#build用のWork Directoryを作成
$mkdir -p /home/reki/vol/x86_64
#build用のWork Directoryへ移動
$cd /home/reki/vol/x86_64
#git cloneする
$git clone https://git.yoctoproject.org/poky
#cloneしたDirectoryへ入る
$ cd poky
#gitのbranchを確認する今回はYocto 5.0.9のscarthgapを見る
$ git branch -a
#scarthgapをチェックアウトしてmy-scarthgapとしてbranchを切る。
$git checkout -t origin/scarthgap  -b my-scarthgap 

#pokyDirectoryから抜ける
cd ../

#oe-init-build-env 環境セットアップ スクリプトを実行
$source poky/oe-init-build-env

#ちなみに通常TOP_DIRはbuildだけど、以下のようにBUILDDIRを変更するとTOPDIRの変数を変更できる。
export BUILDDIR=/home/reki/vol/x86_64/build_scarthgap_genx86_64
source poky/oe-init-build-env $BUILDDIR


$vi conf/local.conf
MACHINE = "genericx86-64"

DL_DIR ?= "${TOPDIR}/../downloads"
SSTATE_DIR ?= "${TOPDIR}/../sstate-cache"

BB_HASHSERVE_UPSTREAM = "wss://hashserv.yoctoproject.org/ws"
SSTATE_MIRRORS ?= "file://.* http://cdn.jsdelivr.net/yocto/sstate/all/PATH;downloadfilename=PATH"
BB_HASHSERVE = "auto"
BB_SIGNATURE_HANDLER = "OEEquivHash"

BB_NUMBER_THREADS ?= "8"
PARALLEL_MAKE = "-j 10"

$bitbake core-image-minimal

#buildしたimageは以下参照
ls /home/reki/vol/x86_64/build_genx86_64/tmp/deploy/images/genericx86-64
addonx64.efi.stub
bzImage
bzImage--6.6.21+git0+2b3144e072_06644f0d71-r0-genericx86-64-20250501000249.bin
bzImage-genericx86-64.bin
core-image-minimal.env
core-image-minimal-genericx86-64.rootfs-20250501000249.manifest
core-image-minimal-genericx86-64.rootfs-20250501000249.qemuboot.conf
core-image-minimal-genericx86-64.rootfs-20250501000249.spdx.json
core-image-minimal-genericx86-64.rootfs-20250501000249.testdata.json
core-image-minimal-genericx86-64.rootfs-20250501000249.wic
core-image-minimal-genericx86-64.rootfs-20250501000249.wic.bmap
core-image-minimal-genericx86-64.rootfs.manifest
core-image-minimal-genericx86-64.rootfs.qemuboot.conf
core-image-minimal-genericx86-64.rootfs.spdx.json
core-image-minimal-genericx86-64.rootfs.testdata.json
core-image-minimal-genericx86-64.rootfs.wic
core-image-minimal-genericx86-64.rootfs.wic.bmap
core-image-minimal-genericx86.wks
grub-efi-bootx64.efi
linuxx64.efi.stub
modules--6.6.21+git0+2b3144e072_06644f0d71-r0-genericx86-64-20250501000249.tgz
modules-genericx86-64.tgz
systemd-bootx64.efi

#wicを直接USB[例./dev/sdbとした場合]へWriteする
wic write /home/reki/vol/x86_64/build_genx86_64/tmp/deploy/images/genericx86-64core-image-minimal-genericx86-64.rootfs-20250501000249.wic /dev/sdb
#もしくはddでもOKです。
dd if=tmp/deploy/images/genericx86-64/core-image-minimal-genericx86-64.rootfs.wic of=/dev/sdb bs=4M status=progress
sync
#windowsの場合はWin32DiskImagerを使うと便利です。ちなみにXG135は起動しなかった。BIOSがefi未対応？→DefaultはLegacy=BIOSだった。
#再度UEFIに変更してUSBからBootさせると起動しました。NICも認識されている。

```

### 1.1.2.4.パターン4 rasberry piの汎用イメージ作成[Yoctoバージョン:Scarthgap]

SBCで有名なrasberry pi[B+]のイメージを作成します。

``` sh

#build用のWork Directoryを作成
$mkdir -p /home/reki/vol/rpi
#build用のWork Directoryへ移動
$cd /home/reki/vol/rpi
#git cloneする
$git clone https://git.yoctoproject.org/poky
#cloneしたDirectoryへ入る
$ cd poky
#gitのbranchを確認する今回はYocto 5.0.9のscarthgapを見る
$ git branch -a
#scarthgapをチェックアウトしてmy-scarthgapとしてbranchを切る。
$git checkout -t origin/scarthgap  -b my-scarthgap 

#pokyDirectoryから抜ける
cd ../

#oe-init-build-env 環境セットアップ スクリプトを実行
$source poky/oe-init-build-env

#ちなみに通常TOP_DIRはbuildだけど、以下のようにBUILDDIRを変更するとTOPDIRの変数を変更できる。
export BUILDDIR=/home/reki/vol/x86_64/build_scarthgap_gen_rpi
source poky/oe-init-build-env $BUILDDIR

#rasberry pi用のソース取得
bitbake-layers layerindex-fetch meta-raspberrypi -b scarthgap
#出力メッセージにはいかがでるので/home/reki/vol/x86_64/poky/meta-raspberrypiに格納されているようだ。
Cloning into '/home/reki/vol/x86_64/poky/meta-raspberrypi'...
git.yoctoproject.org[0: 38.108.68.216]: errno=Connection timed out

ERROR: Failed to download git://git.yoctoproject.org/meta-raspberrypi (scarthgap)
#↑の場合、gitプロトコルではだめでhttpsへアクセスする必要が環境上必要なので[Firewallの都合上]、git:の場合はhttps:へ置き換えるようにする。
git config --global url."https://".insteadOf "git://"
#設定確認
git config --global --list | grep url
#正しい出力結果
url.https://.insteadof=git://
#↑の状態で下記を再度実行すると成功する。
bitbake-layers layerindex-fetch meta-raspberrypi -b scarthgap
#結果
Adding layer "meta-lts-mixins" (/home/reki/vol/x86_64/poky/meta-lts-mixins/) to conf/bblayers.conf
Adding layer "meta-raspberrypi" (/home/reki/vol/x86_64/poky/meta-raspberrypi/) to conf/bblayers.conf

$vi conf/local.conf
MACHINE = "raspberrypi"

DL_DIR ?= "${TOPDIR}/../downloads"
SSTATE_DIR ?= "${TOPDIR}/../sstate-cache"

BB_HASHSERVE_UPSTREAM = "wss://hashserv.yoctoproject.org/ws"
SSTATE_MIRRORS ?= "file://.* http://cdn.jsdelivr.net/yocto/sstate/all/PATH;downloadfilename=PATH"
BB_HASHSERVE = "auto"
BB_SIGNATURE_HANDLER = "OEEquivHash"

BB_NUMBER_THREADS ?= "8"
PARALLEL_MAKE = "-j 10"

$time bitbake core-image-minimal

#buildしたimageは以下参照
ls /home/reki/vol/x86_64/build_scarthgap_gen_rpi/tmp/deploy/images/raspberrypi
at86rf233-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
at86rf233.dtbo
at86rf233-raspberrypi.dtbo
bcm2708-rpi-b-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2708-rpi-b.dtb
bcm2708-rpi-b-plus-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2708-rpi-b-plus.dtb
bcm2708-rpi-b-plus-raspberrypi.dtb
bcm2708-rpi-b-raspberrypi.dtb
bcm2708-rpi-b-rev1-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2708-rpi-b-rev1.dtb
bcm2708-rpi-b-rev1-raspberrypi.dtb
bcm2708-rpi-cm-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2708-rpi-cm.dtb
bcm2708-rpi-cm-raspberrypi.dtb
bcm2708-rpi-zero-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2708-rpi-zero.dtb
bcm2708-rpi-zero-raspberrypi.dtb
bcm2708-rpi-zero-w-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2708-rpi-zero-w.dtb
bcm2708-rpi-zero-w-raspberrypi.dtb
bcm2709-rpi-2-b-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2709-rpi-2-b.dtb
bcm2709-rpi-2-b-raspberrypi.dtb
bcm2710-rpi-2-b-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2710-rpi-2-b.dtb
bcm2710-rpi-2-b-raspberrypi.dtb
bcm2710-rpi-3-b-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2710-rpi-3-b.dtb
bcm2710-rpi-3-b-plus-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2710-rpi-3-b-plus.dtb
bcm2710-rpi-3-b-plus-raspberrypi.dtb
bcm2710-rpi-3-b-raspberrypi.dtb
bcm2710-rpi-cm3-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2710-rpi-cm3.dtb
bcm2710-rpi-cm3-raspberrypi.dtb
bcm2710-rpi-zero-2-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2710-rpi-zero-2.dtb
bcm2710-rpi-zero-2-raspberrypi.dtb
bcm2711-rpi-400-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2711-rpi-400.dtb
bcm2711-rpi-400-raspberrypi.dtb
bcm2711-rpi-4-b-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2711-rpi-4-b.dtb
bcm2711-rpi-4-b-raspberrypi.dtb
bcm2711-rpi-cm4-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2711-rpi-cm4.dtb
bcm2711-rpi-cm4-raspberrypi.dtb
bcm2711-rpi-cm4s-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
bcm2711-rpi-cm4s.dtb
bcm2711-rpi-cm4s-raspberrypi.dtb
bootfiles
core-image-minimal.env
core-image-minimal-raspberrypi.rootfs-20250505064525.ext3
core-image-minimal-raspberrypi.rootfs-20250505064525.manifest
core-image-minimal-raspberrypi.rootfs-20250505064525.spdx.tar.zst
core-image-minimal-raspberrypi.rootfs-20250505064525.tar.bz2
core-image-minimal-raspberrypi.rootfs-20250505064525.testdata.json
core-image-minimal-raspberrypi.rootfs-20250505064525.wic.bmap
core-image-minimal-raspberrypi.rootfs-20250505064525.wic.bz2
core-image-minimal-raspberrypi.rootfs.ext3
core-image-minimal-raspberrypi.rootfs.manifest
core-image-minimal-raspberrypi.rootfs.spdx.tar.zst
core-image-minimal-raspberrypi.rootfs.tar.bz2
core-image-minimal-raspberrypi.rootfs.testdata.json
core-image-minimal-raspberrypi.rootfs.wic.bmap
core-image-minimal-raspberrypi.rootfs.wic.bz2
disable-bt-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
disable-bt.dtbo
disable-bt-raspberrypi.dtbo
disable-wifi-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
disable-wifi.dtbo
disable-wifi-raspberrypi.dtbo
dwc2-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
dwc2.dtbo
dwc2-raspberrypi.dtbo
gpio-ir-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
gpio-ir.dtbo
gpio-ir-raspberrypi.dtbo
gpio-ir-tx-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
gpio-ir-tx.dtbo
gpio-ir-tx-raspberrypi.dtbo
gpio-key-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
gpio-key.dtbo
gpio-key-raspberrypi.dtbo
gpio-poweroff-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
gpio-poweroff.dtbo
gpio-poweroff-raspberrypi.dtbo
gpio-shutdown-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
gpio-shutdown.dtbo
gpio-shutdown-raspberrypi.dtbo
hifiberry-amp100-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-amp100.dtbo
hifiberry-amp100-raspberrypi.dtbo
hifiberry-amp-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-amp3-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-amp3.dtbo
hifiberry-amp3-raspberrypi.dtbo
hifiberry-amp4pro-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-amp4pro.dtbo
hifiberry-amp4pro-raspberrypi.dtbo
hifiberry-amp.dtbo
hifiberry-amp-raspberrypi.dtbo
hifiberry-dac-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-dac.dtbo
hifiberry-dacplus-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-dacplusadc-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-dacplusadc.dtbo
hifiberry-dacplusadcpro-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-dacplusadcpro.dtbo
hifiberry-dacplusadcpro-raspberrypi.dtbo
hifiberry-dacplusadc-raspberrypi.dtbo
hifiberry-dacplusdsp-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-dacplusdsp.dtbo
hifiberry-dacplusdsp-raspberrypi.dtbo
hifiberry-dacplus.dtbo
hifiberry-dacplushd-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-dacplushd.dtbo
hifiberry-dacplushd-raspberrypi.dtbo
hifiberry-dacplus-raspberrypi.dtbo
hifiberry-dac-raspberrypi.dtbo
hifiberry-digi-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-digi.dtbo
hifiberry-digi-pro-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
hifiberry-digi-pro.dtbo
hifiberry-digi-pro-raspberrypi.dtbo
hifiberry-digi-raspberrypi.dtbo
i2c-gpio-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
i2c-gpio.dtbo
i2c-gpio-raspberrypi.dtbo
i2c-rtc-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
i2c-rtc.dtbo
i2c-rtc-raspberrypi.dtbo
imx219-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
imx219.dtbo
imx219-raspberrypi.dtbo
imx477-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
imx477.dtbo
imx477-raspberrypi.dtbo
imx708-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
imx708.dtbo
imx708-raspberrypi.dtbo
iqaudio-dac-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
iqaudio-dac.dtbo
iqaudio-dacplus-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
iqaudio-dacplus.dtbo
iqaudio-dacplus-raspberrypi.dtbo
iqaudio-dac-raspberrypi.dtbo
justboom-both-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
justboom-both.dtbo
justboom-both-raspberrypi.dtbo
justboom-dac-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
justboom-dac.dtbo
justboom-dac-raspberrypi.dtbo
justboom-digi-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
justboom-digi.dtbo
justboom-digi-raspberrypi.dtbo
mcp2515-can0-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
mcp2515-can0.dtbo
mcp2515-can0-raspberrypi.dtbo
mcp2515-can1-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
mcp2515-can1.dtbo
mcp2515-can1-raspberrypi.dtbo
mcp3008-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
mcp3008.dtbo
mcp3008-raspberrypi.dtbo
miniuart-bt-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
miniuart-bt.dtbo
miniuart-bt-raspberrypi.dtbo
modules-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.tgz
modules-raspberrypi.tgz
overlay_map-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtb
overlay_map.dtb
overlay_map-raspberrypi.dtb
pitft22-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
pitft22.dtbo
pitft22-raspberrypi.dtbo
pitft28-capacitive-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
pitft28-capacitive.dtbo
pitft28-capacitive-raspberrypi.dtbo
pitft28-resistive-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
pitft28-resistive.dtbo
pitft28-resistive-raspberrypi.dtbo
pitft35-resistive-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
pitft35-resistive.dtbo
pitft35-resistive-raspberrypi.dtbo
pps-gpio-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
pps-gpio.dtbo
pps-gpio-raspberrypi.dtbo
rpi-ft5406-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
rpi-ft5406.dtbo
rpi-ft5406-raspberrypi.dtbo
rpi-poe-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
rpi-poe.dtbo
rpi-poe-raspberrypi.dtbo
uart0-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart0.dtbo
uart0-pi5-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart0-pi5.dtbo
uart0-pi5-raspberrypi.dtbo
uart0-raspberrypi.dtbo
uart1-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart1.dtbo
uart1-pi5-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart1-pi5.dtbo
uart1-pi5-raspberrypi.dtbo
uart1-raspberrypi.dtbo
uart2-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart2.dtbo
uart2-pi5-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart2-pi5.dtbo
uart2-pi5-raspberrypi.dtbo
uart2-raspberrypi.dtbo
uart3-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart3.dtbo
uart3-pi5-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart3-pi5.dtbo
uart3-pi5-raspberrypi.dtbo
uart3-raspberrypi.dtbo
uart4-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart4.dtbo
uart4-pi5-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart4-pi5.dtbo
uart4-pi5-raspberrypi.dtbo
uart4-raspberrypi.dtbo
uart5-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
uart5.dtbo
uart5-raspberrypi.dtbo
vc4-fkms-v3d-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
vc4-fkms-v3d.dtbo
vc4-fkms-v3d-pi4-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
vc4-fkms-v3d-pi4.dtbo
vc4-fkms-v3d-pi4-raspberrypi.dtbo
vc4-fkms-v3d-raspberrypi.dtbo
vc4-kms-dsi-7inch-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
vc4-kms-dsi-7inch.dtbo
vc4-kms-dsi-7inch-raspberrypi.dtbo
vc4-kms-v3d-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
vc4-kms-v3d.dtbo
vc4-kms-v3d-pi4-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
vc4-kms-v3d-pi4.dtbo
vc4-kms-v3d-pi4-raspberrypi.dtbo
vc4-kms-v3d-pi5-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
vc4-kms-v3d-pi5.dtbo
vc4-kms-v3d-pi5-raspberrypi.dtbo
vc4-kms-v3d-raspberrypi.dtbo
w1-gpio-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
w1-gpio.dtbo
w1-gpio-pullup-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
w1-gpio-pullup.dtbo
w1-gpio-pullup-raspberrypi.dtbo
w1-gpio-raspberrypi.dtbo
wm8960-soundcard-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.dtbo
wm8960-soundcard.dtbo
wm8960-soundcard-raspberrypi.dtbo
zImage
zImage-1-6.6.63+git0+52ff0d7571_e442e5c1ab-r0-raspberrypi-20250505064525.bin
zImage-raspberrypi.bin

#windowsの場合はcore-image-minimal-raspberrypi.rootfs.wic.bz2をbz2を解凍し、wicファイル→Win32DiskImagerへ渡すことで可能です。

```

### 1.1.3.Yoctoの初期設定

やること

- 1.初期ユーザパスワード設定とOpenSSHサーバのInstall

- 2.Network設定
  初期のIPアドレスを変更する。

- 3.USB-WIFIをNetwork追加する。
  Driverは「IO-DATA WN-G150UM」なので「rtl8192cu」となる。
  現時点ではLinuxの標準Driverで入っているため、そのまま利用できる認識。
  以下を追加してみよう。
  core-image-minimalを選択出来るが、カーネルモジュールが取り込まれず、GPIOなどの使用に支障が出る模様。
　conf/local.confに以下の設定を追加すればおおよそのモジュールは取り込まれる。
　MACHINE_ESSENTIAL_EXTRA_RRECOMMENDS += "kernel-modules"

- 4.USB-WIFIとLAN Network BrigeしてEthernet Converter化

- 5.不揮発性ディスク→ramdisk化して起動する

#### 1.1.3.1.初期ユーザパスワード設定とOpenSSHサーバのInstall

``` sh

#環境変数の設定をする BUILDDIRはbuildする場所によって対象が異なると思うので環境によって変えてほしい。
export BUILDDIR=/home/reki/vol/x86_64/build_scarthgap_gen_rpi
source poky/oe-init-build-env $BUILDDIR

#変更前にbackupする backupはbootOKだった時のもの
cp conf/local.conf conf/local.conf_bk_1_bootOK

#パスワード:a278354を平文のためそのまま設定するとセキュリティ的に推奨されないので、sha256などでハッシュ化します。
openssl passwd -5 a278354
#出力結果は以下でフォーマット:$5$salt$hashedpasswordのようになっている
$5$oaCS4yDHJuszlISw$918.sSq1gEdBEuMp5ETDzvwfr628PZkU2mvDYSbvd.C
#↑の出力結果をFile:conf/local.confへ以下を追記する 注意点は特殊文字である$をエスケープ[\の意味]する必要がある。
#conf/local.confへ以下を変更する
#追加内容はrootユーザのパスワード無効化するdebug-tweaksをremove,sshサーバのInstall,rootと一般ユーザ:rekiのパスワード追加および作成
+ EXTRA_IMAGE_FEATURES:remove = "debug-tweaks"
+ IMAGE_INSTALL:append = " openssh openssh-sftp-server"

+ EXTRA_USERS_PARAMS:append = "usermod -p '\$5\$oaCS4yDHJuszlISw\$918.sSq1gEdBEuMp5ETDzvwfr628PZkU2mvDYSbvd.C' root;"
+ EXTRA_USERS_PARAMS:append = " useradd -m -s /bin/sh -p '\$5\$oaCS4yDHJuszlISw\$918.sSq1gEdBEuMp5ETDzvwfr628PZkU2mvDYSbvd.C' reki;"
+ INHERIT += "extrausers"


#イメージのリビルド
bitbake core-image-minimal


#loop deviceを利用してリビルドしたイメージを確認するため、リビルドしたイメージをコピーしてHost OSで作業する。
cd tmp/deploy/images/raspberrypi

scp core-image-minimal-raspberrypi.rootfs.ext3 reki@10.10.10.1:/home/reki/download
ssh reki@10.10.10.1
su
cd /home/reki/download
mkdir -p /mnt/yocto_rootfs
losetup -fP core-image-minimal-raspberrypi.rootfs.ext3
#core-image-minimal-raspberrypi.rootfs.ext3の対象となるloop1を確認
losetup -a
/dev/loop1: [2050]:37487193 (/home/reki/download/core-image-minimal-raspberrypi.rootfs.ext3)

mount /dev/loop1 /mnt/yocto_rootfs

#確認
df -h /dev/loop1
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop1       14M  9.8M  3.4M  75% /mnt/yocto_rootfs

cd /mnt/yocto_rootfs
#結果は以下、正しいユーザとHash値が入っている
# cat etc/shadow | grep root
root:$5$oaCS4yDHJuszlISw$918.sSq1gEdBEuMp5ETDzvwfr628PZkU2mvDYSbvd.C:15069:0:99999:7:::

# cat etc/shadow | grep reki
reki:$5$oaCS4yDHJuszlISw$918.sSq1gEdBEuMp5ETDzvwfr628PZkU2mvDYSbvd.C:15069:0:99999:7:::

# cat etc/passwd | grep root
root:x:0:0:root:/home/root:/bin/sh

# cat etc/passwd | grep reki
reki:x:1000:1000::/home/reki:/bin/sh

#確認が終わったので、umount
cd 
umount /mnt/yocto_rootfs
#loopデバイスを削除
losetup -d /dev/loop1

#不要なファイルの削除
cd /home/reki/download
rm core-image-minimal-raspberrypi.rootfs.ext3

#戻る
exit
exit

#imageをSDカードにWriteして、起動するとroot or rekiでterminal or sshでログインできる
#windowsの場合はcore-image-minimal-raspberrypi.rootfs.wic.bz2をbz2を解凍し、wicファイル→Win32DiskImagerへ渡すことで可能です。

```

#### 1.1.3.2.Network設定

  初期のIPアドレスを変更する。

Yoctoの場合は以下のように変更する。

``` sh

#環境変数の設定をする BUILDDIRはbuildする場所によって対象が異なると思うので環境によって変えてほしい。
export BUILDDIR=/home/reki/vol/x86_64/build_scarthgap_gen_rpi
source poky/oe-init-build-env $BUILDDIR

#変更前にbackupする backupはbootOKだった時のもの
cp conf/local.conf conf/local.conf_bk_2_bootOK

#自身のlayerを作成し、加える。
cd /home/reki/vol/x86_64
bitbake-layers create-layer meta-myConfig -p 9
rm -rf ./meta-myConfig/recipes-example
cd build_scarthgap_gen_rpi
bitbake-layers add-layer ../meta-myConfig
cat conf/bblayers.conf  
#確認:/home/reki/vol/x86_64/meta-myConfigが追加されていること

```

今回、初期のIPアドレスを変更したいので、まずはraspberryパイへログインすると以下でした。

``` sh
cat /etc/network/interfaces
# /etc/network/interfaces -- configuration file for ifup(8), ifdown(8)
 
# The loopback interface
auto lo
iface lo inet loopback

# Wireless interfaces
iface wlan0 inet dhcp
	wireless_mode managed
	wireless_essid any
	wpa-driver wext
	wpa-conf /etc/wpa_supplicant.conf

iface atml0 inet dhcp

# Wired or wireless interfaces
auto eth0
iface eth0 inet dhcp
iface eth1 inet dhcp

# Ethernet/RNDIS gadget (g_ether)
# ... or on host side, usbnet and random hwaddr
iface usb0 inet static
	address 192.168.7.2
	netmask 255.255.255.0
	network 192.168.7.0
	gateway 192.168.7.1

# Bluetooth networking
iface bnep0 inet dhcp
```

↑を以下のように変更したいです。

``` sh
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
      address 172.16.16.20
      netmask 255.255.255.0
      network 172.16.16.1
      gateway 172.16.16.1
```

まずは「/etc/network/interfaces」が属するパッケージの確認します。

``` sh
oe-pkgdata-util find-path /etc/network/interfaces
init-ifupdown: /etc/network/interfaces　★この←側にある「init-ifupdown」がパッケージ名
左[パッケージ]:右[パス]
```

パッケージからどのbbファイル[yoctoにおける構成管理ファイル]なのか検索する。

``` sh
cd ../poky
find . -name init-ifupdown*.bb
./meta/recipes-core/init-ifupdown/init-ifupdown_1.0.bb
#bbファイルでfile://interfaces \を検索する。
cat ./meta/recipes-core/init-ifupdown/init-ifupdown_1.0.bb | grep file://interfaces

```

↑の「file://interfaces \」に「interfaces」があるので、これをbackupして書き換えようと思う。
↑のlsコマンドでpathを検索すると以下

``` sh
cd ./meta/recipes-core/init-ifupdown
find . -name interfaces
./init-ifupdown-1.0/interfaces
```

つまり./meta/recipes-core/init-ifupdown/init-ifupdown-1.0/interfacesになる。

以下はinit-ifupdownをコピーとユーザ独自定義を追加する。先ほどのbackupした「interfaces」を利用する

``` sh

cd /home/reki/vol/x86_64
mkdir -p meta-myConfig/recipes-core/init-ifupdown
touch meta-myConfig/recipes-core/init-ifupdown/init-ifupdown_%.bbappend
vi meta-myConfig/recipes-core/init-ifupdown/init-ifupdown_%.bbappend
#FILESEXTRAPATHS:prepend = "${THISDIR}/files:"　★単に=だと遅延評価されるつまりBitbake実行中に評価。
#- Yocto 3.4 以降の BitBake では、「環境変数が既にセットされている場合、prepend が期待通り動作しないことがある」 という変更が加えられました。
#このため、場合によっては FILESEXTRAPATHS の変更が 正しく優先順位を維持できなくなる という問題が発生します。
#- := を使うと、BitBake の環境変数設定時点で確定値として評価される。
#- これにより、FILESEXTRAPATHS の変更が 確実に優先される ようになる。
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://interfaces"
do_install:append() {
    rm -f ${D}${sysconfdir}/network/interfaces
    install -m 0644 ${WORKDIR}/interfaces ${D}${sysconfdir}/network/interfaces
}
FILES_${PN} += "${sysconfdir}/network/interfaces"
PACKAGE_WRITE_DEPS += "base-files"

mkdir -p meta-myConfig/recipes-core/init-ifupdown/files
cp poky/meta/recipes-core/init-ifupdown/init-ifupdown-1.0/interfaces meta-myConfig/recipes-core/init-ifupdown/files/interfaces
#以下のみ記載。
vi meta-myConfig/recipes-core/init-ifupdown/files/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
      address 172.16.16.20
      netmask 255.255.255.0
      network 172.16.16.1
      gateway 172.16.16.1

#vi build_scarthgap_gen_rpi/conf/local.conf
#IMAGE_INSTALL:append = "meta-myConfig"
#BBFILE_PRIORITY_meta-myConfig = "4"

#再度rootfsを作成しなおす。
#bitbake -c cleanall init-ifupdown && \
bitbake -c cleansstate init-ifupdown && \
bitbake init-ifupdown

bitbake core-image-minimal

#確認
ls build_scarthgap_gen_rpi/tmp/work/raspberrypi-poky-linux-gnueabi/core-image-minimal/1.0/rootfs/etc/network/interfaces
cat build_scarthgap_gen_rpi/tmp/work/raspberrypi-poky-linux-gnueabi/core-image-minimal/1.0/rootfs/etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
      address 172.16.16.20
      netmask 255.255.255.0
      network 172.16.16.1
      gateway 172.16.16.1

```

結果として、meta-myConfig/recipes-core/init-ifupdown/init-ifupdown_%.bbappendの定義を追加することで、rootfs配下の既存ファイルを書き換えることが可能となった。

#### 1.1.3.3.USB-WIFIをNetwork追加する

今回利用するUSB-WIFI用のアダプタは以下です。  
[IO-DATA WN-G150UM](https://www.iodata.jp/product/network/adp/wn-g150um/)

まずはDriverがすでに組み込まれているかチェックする。  
Driverは以下の参考①より、「rtl8192cu」です。  
[参考①](https://gist.github.com/nidate/48818c520f49f8f0d883)

raspberryパイへログインすると以下でした

``` sh
#カーネルに組み込まれているか確認したい場合
# modinfo rtl8192cu
filename:       /lib/modules/6.6.63/kernel/drivers/net/wireless/realtek/rtlwifi/rtl8192cu/rtl8192cu.ko.xz
firmware:       rtlwifi/rtl8192cufw_TMSC.bin
firmware:       rtlwifi/rtl8192cufw_B.bin
firmware:       rtlwifi/rtl8192cufw_A.bin
firmware:       rtlwifi/rtl8192cufw.bin
description:    Realtek 8192C/8188C 802.11n USB wireless
・
・略
・
name:           rtl8192cu
vermagic:       6.6.63 mod_unload modversions ARMv6 p2v8
parm:           swenc:Set to 1 for software crypto (default 0)
 (bool)
parm:           debug_level:Set debug level (0-5) (default 0) (int)
parm:           debug_mask:Set debug mask (default 0) (ullong)

#具体的な環境でのパスを調べる場合
# find /lib/modules/$(uname -r) -name "rtl8192cu*"
/lib/modules/6.6.63/kernel/drivers/net/wireless/realtek/rtlwifi/rtl8192cu
/lib/modules/6.6.63/kernel/drivers/net/wireless/realtek/rtlwifi/rtl8192cu/rtl8192cu.ko.xz

#実際にDriverが組み込まれているかどうかは以下でrtl8192cuがあるか確認
lsmod
#私の時はなかったの後述の設定します。

```

Driverの確認ができたので、あとはNICに認識されるように設定します。

この手順について詳しくは、[参考①](https://gist.github.com/nidate/48818c520f49f8f0d883)を参照してください。

``` sh
#udevへ設定
echo 'ACTION=="add",SUBSYSTEMS=="usb",ATTRS{idVendor}=="04bb",ATTRS{idProduct}=="094c",RUN+="/sbin/modprobe -qba rtl8192cu"' > /etc/udev/rules.d/network_drivers.rules
#modprobe.confへ設定
echo 'install rtl8192cu /sbin/modprobe --ignore-install rtl8192cu $CMDLINE_OPTS; /bin/echo "04bb 094c" > /sys/bus/usb/drivers/rtl8192cu/new_id' > /etc/modprobe.d/network_drivers.conf

#反映させるためにreboot

#wlan0の確認
# ip a
3: wlan0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue qlen 1000
    link/ether 00:a0:b0:f3:d7:c0 brd ff:ff:ff:ff:ff:ff

```

↑の手順をyocto環境に組み込みます。

``` sh
#環境変数の設定をする BUILDDIRはbuildする場所によって対象が異なると思うので環境によって変えてほしい。
export BUILDDIR=/home/reki/vol/x86_64/build_scarthgap_gen_rpi
source poky/oe-init-build-env $BUILDDIR

#変更前にbackupする backupはbootOKだった時のもの
cp conf/local.conf conf/local.conf_bk_3_bootOK

#自身のlayerを作成し、加えるます。既にある場合はSkip
cd /home/reki/vol/x86_64
bitbake-layers create-layer meta-myConfig -p 9
rm -rf ./meta-myConfig/recipes-example
cd build_scarthgap_gen_rpi
bitbake-layers add-layer ../meta-myConfig
cat conf/bblayers.conf
#確認:/home/reki/vol/x86_64/meta-myConfigが追加されていること
```

まずは「/etc/udev/rules.d」が属するパッケージの確認します。  
/etc/udev/rules.dはディレクトリです。  
追加するファイルは「network_drivers.rules」なので既存のパッケージ上には存在しないので、新しく作成しますが、参考までに近いパスを確認します。

``` sh
oe-pkgdata-util find-path /etc/udev/rules.d/*
eudev: /etc/udev/rules.d/80-net-name-slot.rules
eudev: /etc/udev/rules.d/local.rules
「eudev」がパッケージ名
左[パッケージ]:右[パス]
```

パッケージからどのbbファイル[yoctoにおける構成管理ファイル]なのか検索する。

``` sh
cd ../poky
find . -name eudev*.bb
./meta/recipes-core/udev/eudev_3.2.14.bb
#bbファイルでfile://local.rules \を検索し存在確認。
cat ./meta/recipes-core/udev/eudev_3.2.14.bb | grep file://local.rules
```

↑の「file://local.rules \」を参考に「network_drivers.rules」を追加していく。  
これをbackupしてRenameして書き換えようと思う。  
↑のlsコマンドでpathを検索すると以下

``` sh
cd ./meta/recipes-core/udev/
find . -name local.rules
./eudev/local.rules
```

つまり./meta/recipes-core/udev/eudev/local.rulesになる。  
以下はeudevをコピーとユーザ独自定義を追加する。  
先ほどのbackupした「local.rules」を利用する

``` sh

cd /home/reki/vol/x86_64
mkdir -p meta-myConfig/recipes-core/udev/
touch meta-myConfig/recipes-core/udev/eudev_%.bbappend
vi meta-myConfig/recipes-core/udev/eudev_%.bbappend
#ここから
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://network_drivers.rules"
do_install:append() {
    rm -f ${D}${sysconfdir}/udev/rules.d/network_drivers.rules
    install -m 0644 ${WORKDIR}/network_drivers.rules ${D}${sysconfdir}/udev/rules.d/network_drivers.rules
}
FILES_${PN} += "${sysconfdir}/udev/rules.d/network_drivers.rules"
PACKAGE_WRITE_DEPS += "base-files"
#ここまでを記載

mkdir -p meta-myConfig/recipes-core/udev/files
cp poky/meta/recipes-core/udev/eudev/local.rules meta-myConfig/recipes-core/udev/files/network_drivers.rules
#以下変更し記載。
vi meta-myConfig/recipes-core/udev/files/network_drivers.rules
全部削除し以下記載
ACTION=="add",SUBSYSTEMS=="usb",ATTRS{idVendor}=="04bb",ATTRS{idProduct}=="094c",RUN+="/sbin/modprobe -qba rtl8192cu"

#再度rootfsを作成しなおす。
bitbake -c cleansstate eudev && \
bitbake eudev
#以下があれば成功
ls build_scarthgap_gen_rpi/tmp/work/arm1176jzfshf-vfp-poky-linux-gnueabi/eudev/3.2.14/image/etc/udev/rules.d/network_drivers.rules

```

同じように今度はmodprobe.confを追加します。  
まずは「/etc/modprobe.d/*」が属するパッケージの確認します。  
/etc/modprobe.dはディレクトリです。  
追加するファイルは「network_drivers.conf」なので既存のパッケージ上には存在しないので、新しく作成しますが、参考までに近いパスを確認します。  

``` sh
oe-pkgdata-util find-path /etc/modprobe.d/*
ERROR: Unable to find any package producing path /etc/modprobe.d/*
#↑のような結果になり、/etc/modprobe.dへ追加するパッケージは存在しません。
#この場合はカスタムレシピを追加します。

```

以下は/etc/modprobe.d/network_drivers.confをカスタムレシピを追加する。

``` sh

#1. network_drivers.conf を含むレシピを作成
cd /home/reki/vol/x86_64
mkdir -p meta-myConfig/recipes-kernel/network-drivers/files
touch meta-myConfig/recipes-kernel/network-drivers/network-drivers.bb
#ライセンスチェックが必要なので、LICENSEを追加。
#今回は正式なライセンスを付与するものではなく、ライセンスの付与方法を記載
vi meta-myConfig/recipes-kernel/network-drivers/files/LICENSE
#ここから
Copyright (c) <Year> <Your Name>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software...
#ここまで

#LICENSEのmd5sumを確認する。
md5sum meta-myConfig/recipes-kernel/network-drivers/files/LICENSE
74dbfde691911c3aecb8ff7846e8d88e

vi meta-myConfig/recipes-kernel/network-drivers/network-drivers.bb
#ここから
SUMMARY = "USB WIFI Network driver configurations"
DESCRIPTION = "Modprobe configuration for USB WIFI network drivers"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://LICENSE;md5=74dbfde691911c3aecb8ff7846e8d88e"

SRC_URI = "file://network_drivers.conf \
           file://LICENSE"

do_populate_lic() {
    install -D ${WORKDIR}/LICENSE ${WORKDIR}/${BP}/LICENSE
}

do_install() {
    install -D ${WORKDIR}/network_drivers.conf ${D}${sysconfdir}/modprobe.d/network_drivers.conf
}

FILES_${PN} += "${sysconfdir}/modprobe.d/network_drivers.conf"
FILES_${PN} += "${WORKDIR}/${BP}LICENSE"
#ここまでを記載

#以下変更し記載。
vi meta-myConfig/recipes-kernel/network-drivers/files/network_drivers.conf
install rtl8192cu /sbin/modprobe --ignore-install rtl8192cu $CMDLINE_OPTS; /bin/echo "04bb 094c" > /sys/bus/usb/drivers/rtl8192cu/new_id

#build_scarthgap_gen_rpi/conf/local.confへ以下を追加
IMAGE_INSTALL:append = " \
  openssh \
  openssh-sftp-server \
  network-drivers " ★追加

bitbake network-drivers -c cleansstate
bitbake network-drivers -c cleanall
bitbake network-drivers
```

コマンドやツール類であるlsblk,lsusb,wpa_supplicant、iw、iwconfigをinstallする。  
iw,iwconfig,wpa-supplicantはwifiクライアント用のツールをインストールする必要があります。  
あと、diskの見えるかするlsblkやlsmod関連のツールもusb機器を判別するためにinstallします。

``` sh
#wpa_supplicantは下記へ入っているので、追加がない場合は追加
bitbake-layers show-layers | grep meta-networking
#出力なし、なので追加要
cd build_scarthgap_gen_rpi/
cat conf/bblayers.conf
---
BBLAYERS ?= " \
  /home/reki/vol/x86_64/poky/meta \
  /home/reki/vol/x86_64/poky/meta-poky \
  /home/reki/vol/x86_64/poky/meta-yocto-bsp \
  /home/reki/vol/x86_64/poky/meta-lts-mixins \
  /home/reki/vol/x86_64/poky/meta-raspberrypi \
  /home/reki/vol/x86_64/meta-myConfig \
---

cd /home/reki/vol/x86_64/poky
git ls-remote --heads https://git.openembedded.org/meta-openembedded | grep scarthgap
e92d0173a80ea7592c866618ef5293203c50544c        refs/heads/scarthgap
e92d0173a80ea7592c866618ef5293203c50544c        refs/heads/scarthgap-next

git clone -b scarthgap https://git.openembedded.org/meta-openembedded

#依存関係の都合上network→python→oeとなったので追加した。
bitbake-layers add-layer meta-openembedded/meta-oe
bitbake-layers add-layer meta-openembedded/meta-python
bitbake-layers add-layer meta-openembedded/meta-networking

cd ../build_scarthgap_gen_rpi
#追加結果確認は★
cat conf/bblayers.conf
---
BBLAYERS ?= " \
  /home/reki/vol/x86_64/poky/meta \
  /home/reki/vol/x86_64/poky/meta-poky \
  /home/reki/vol/x86_64/poky/meta-yocto-bsp \
  /home/reki/vol/x86_64/poky/meta-lts-mixins \
  /home/reki/vol/x86_64/poky/meta-raspberrypi \
  /home/reki/vol/x86_64/meta-myConfig \
★  /home/reki/vol/x86_64/poky/meta-openembedded/meta-oe \
★  /home/reki/vol/x86_64/poky/meta-openembedded/meta-python \
★  /home/reki/vol/x86_64/poky/meta-openembedded/meta-networking \

---
cd ../

#build_scarthgap_gen_rpi/conf/local.confへ以下の★を追加
IMAGE_INSTALL:append = " \
  openssh \
  openssh-sftp-server \
★  util-linux \
★  usbutils \
★  wpa-supplicant \
★  iw \
  network-drivers "

bitbake core-image-minimal

#確認
ls build_scarthgap_gen_rpi/tmp/work/raspberrypi-poky-linux-gnueabi/core-image-minimal/1.0/rootfs/etc/udev/rules.d/
ls build_scarthgap_gen_rpi/tmp/work/raspberrypi-poky-linux-gnueabi/core-image-minimal/1.0/rootfs/etc/modprobe.d/
cat build_scarthgap_gen_rpi/tmp/work/raspberrypi-poky-linux-gnueabi/core-image-minimal/1.0/rootfs/etc/network/interfaces

```

起動後書き込み設定

``` sh
#WIFIルータのSSIDとPASSPHRASEを書き込む
sh -c 'wpa_passphrase SSID PASSPHRASE >> /etc/wpa_supplicant.conf'
#あとこのままだとPASSPHRASEは見えてしまうので、消す
sed -i '/^#psk=/d' /etc/wpa_supplicant.conf
#これで、パスフレーズの平文が削除され、暗号化されたキー (psk=hex_encoded_key) のみが残ります。

#手動でやる
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant.conf
結果:Successfully initialized wpa_supplicant
#wpaプロセス確認
ps | grep wpa*
  418 root      9496 S    wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant.conf
#/etc/network/interfaceでwlan0を設定する
auto wlan0
iface wlan0 inet dhcp

#接続確認、いったんwlan0をdown→upする
ifdown wlan0
ifup wlan0
udhcpc: started, v1.36.1
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc: broadcasting select for 192.168.2.19, server 192.168.2.1
udhcpc: lease of 192.168.2.19 obtained from 192.168.2.1, lease time 14400
/etc/udhcpc.d/50default: Adding DNS 192.168.2.1

# wpa_cli status
Selected interface 'wlan0'
bssid=08:10:86:04:70:5d
freq=2462
ssid=HMAX03
id=1
mode=station
wifi_generation=4
pairwise_cipher=CCMP
group_cipher=CCMP
key_mgmt=WPA2-PSK
wpa_state=COMPLETED
ip_address=192.168.2.19
p2p_device_address=00:a0:b0:f3:d7:c0
address=00:a0:b0:f3:d7:c0
uuid=e506b618-9594-5c22-ba62-3676ecb92e68
#↑で接続OK

#今度は再起動しても自動的にアクセスできるようにする
#wpa_supplicantをSysVinitへ追加
---
cat >> /etc/init.d/wpa_supplicant << EOF
#!/bin/sh
### BEGIN INIT INFO
# Provides:          wpa_supplicant
# Required-Start:    \$network \$remote_fs
# Required-Stop:     \$network \$remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start/stop wpa_supplicant
### END INIT INFO

DAEMON=/usr/sbin/wpa_supplicant
CONF=/etc/wpa_supplicant.conf
INTERFACE=wlan0
OPTIONS="-B -i \$INTERFACE -c \$CONF"

case "\$1" in
    start)
        echo "Starting wpa_supplicant..."
        \$DAEMON \$OPTIONS
        ;;
    stop)
        echo "Stopping wpa_supplicant..."
        killall wpa_supplicant
        ;;
    restart)
        \$0 stop
        sleep 2
        \$0 start
        ;;
    status)
        pidof wpa_supplicant && echo "wpa_supplicant is running" || echo "wpa_supplicant is not running"
        ;;
    *)
        echo "Usage: \$0 {start|stop|restart|status}"
        exit 1
        ;;
esac

exit 0
EOF

---
#実行権限付与
chmod +x /etc/init.d/wpa_supplicant
#自動起動の設定
update-rc.d wpa_supplicant defaults
#サービスの起動と確認
/etc/init.d/wpa_supplicant start
/etc/init.d/wpa_supplicant status

#再起動しても自動的にアクセスできるようになる。
reboot

```

これでWIFIClientとして利用可能となる。

#### 1.1.3.4.USB-WIFIとLAN Network BrigeしてEthernet Converter化

LANであるeth0 → 無線WIFIのwlan0へインターネット接続ができるようにしたい。  
セキュリティの問題もあるが機能実現を優先

``` sh
#パッケージのInstallする
#build_scarthgap_gen_rpi/conf/local.confへ以下の★を追加
IMAGE_INSTALL:append = " \
  openssh \
  openssh-sftp-server \
  util-linux \
  usbutils \
  wpa-supplicant \
  iw \
　ebtables \
  dhcp-relay \
★ iptables \
★ tcpdump \
★ bridge-utils \
  network-drivers "

#リビルド
bitbake core-image-minimal

#From eth0 To wlan0 NAT Setting And ipv4 forward.
#File:/etc/network/interfacesを以下のようにする
---
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
      address 192.168.3.1
      netmask 255.255.255.0
      pre-up echo "1" > /proc/sys/net/ipv4/ip_forward
      pre-up iptables -t nat -A POSTROUTING -o wlan0 -s 192.168.3.0/24 -j MASQUERADE
      post-up udhcpd -S -I 192.168.3.1 /etc/udhcpd.conf
      post-down echo "0" > /proc/sys/net/ipv4/ip_forward
      post-down iptables -t nat -D POSTROUTING -o wlan0 -s 192.168.3.0/24 -j MASQUERADE
      post-down killall udhcpd

auto wlan0
iface wlan0 inet dhcp
---

#udhcpd Setting for eth0 And NAT Setting.
cat >> /etc/udhcpd.conf << EOF
start 192.168.3.100
end 192.168.3.200
interface eth0
lease 600
pidfile /var/run/udhcpd.pid
option subnet 255.255.255.0
option router 192.168.3.1
option dns 192.168.2.1 8.8.8.8
EOF

#netfilter有効化[iptableのNATのために必要]

#手動
modprobe br_netfilter
#自動
cat >> /etc/init.d/br_netfilter << EOF
#!/bin/sh
### BEGIN INIT INFO
# Provides:          br_netfilter
# Required-Start:    networking
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:
# Short-Description: Load br_netfilter module
### END INIT INFO

case "\$1" in
    start)
        modprobe br_netfilter
        echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
        echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
        echo 1 > /proc/sys/net/bridge/bridge-nf-call-arptables
        ;;
    stop)
        rmmod br_netfilter
        ;;
    *)
        echo "Usage: \$0 {start|stop}"
        exit 1
        ;;
esac

exit 0
EOF
#実行権限付与
chmod +x /etc/init.d/br_netfilter
#スタートアップへ追加
update-rc.d br_netfilter defaults

#設定後の確認
lsmod | grep br_netfilter
cat /proc/sys/net/bridge/bridge-nf-call-iptables
1
cat /proc/sys/net/bridge/bridge-nf-call-arptables
1
cat /proc/sys/net/bridge/bridge-nf-call-ip6tables
1

```

以下はchatGPTでQAした結果 セキュリティについて参考  
セキュリティに関連する相談です。  
以下のコマンドで192.168.3.0のサブネットからwlan0を通してインターネット接続できました。  
 echo "1" > /proc/sys/net/ipv4/ip_forward iptables -t nat -A POSTROUTING -o wlan0 -s 192.168.3.0/24 -j MASQUERADE
セキュリティについて改善点ありますか？  

改善点:  
-1. フィルタリングルールの追加  
現状の設定では、192.168.3.0/24 のクライアントすべてが wlan0 経由でインターネットにアクセスできます。  
不要な通信を制限するために、iptables にフィルタリングルールを追加しましょう。  
例えば、特定のポートだけ許可し、不要なトラフィックを遮断する:

``` sh
iptables -A FORWARD -s 192.168.3.0/24 -d 0.0.0.0/0 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -s 192.168.3.0/24 -d 0.0.0.0/0 -p tcp --dport 443 -j ACCEPT
iptables -A FORWARD -s 192.168.3.0/24 -j DROP
```

このルールにより、HTTP/HTTPS の通信のみ許可し、それ以外のトラフィックは遮断されます。  

-2. IP Spoofing 防止 (rp_filter の有効化)  
IPアドレスの偽装を防ぐために、rp_filter を有効にしましょう。  

``` sh
echo "1" > /proc/sys/net/ipv4/conf/all/rp_filter
echo "1" > /proc/sys/net/ipv4/conf/default/rp_filter
```

- これにより、不正なIPアドレスを使用してパケットを送信することを防止できます  

-3. ログ記録の追加  
どの通信が行われているかを監視するためにログを追加できます。  

``` sh
iptables -A FORWARD -s 192.168.3.0/24 -j LOG --log-prefix "FORWARD-DROP: "
```

- これにより、許可されていないパケットが syslog に記録されるようになります。  

-4. パケット数制限 (rate limiting) の導入

``` sh
iptables -A FORWARD -p tcp --syn -m limit --limit 10/sec -j ACCEPT
iptables -A FORWARD -p tcp --syn -j DROP
```

- これにより、TCPの新規接続試行を1秒間に10回までに制限できます。  

-5. MACアドレスによるフィルタリング  

許可したデバイスのみがインターネットに接続できるようにする:

``` sh
iptables -A FORWARD -m mac --mac-source AA:BB:CC:DD:EE:FF -s 192.168.3.100 -j ACCEPT
iptables -A FORWARD -s 192.168.3.0/24 -j DROP
```

- これで、特定のMACアドレスを持つ端末のみ許可できます。

##### 1.1.3.4.1.WIFIで日本用の設定可能にする

WIFI用のNICで規制データベースのinstallをしないと日本の無線設定ができないため設定可能にする。

``` sh
cd /home/reki/vol/x86_64

mkdir -p meta-myConfig/recipes-kernel/wireless-regdb/files

cat > meta-myConfig/recipes-kernel/wireless-regdb/files/LICENSE.spdx << EOF
SPDXVersion: SPDX-2.3
DataLicense: CC0-1.0
SPDXID: SPDXRef-DOCUMENT
DocumentName: ISC-License
DocumentNamespace: https://wireless.wiki.kernel.org/en/developers/regulatory/crda
Creator: Organization: wireless-regdb maintainers
Created: 2025-05-17T10:00:00Z
LicenseListVersion: 3.21
PackageName: wireless-regdb-static
PackageDownloadLocation: https://www.kernel.org/pub/software/network/wireless-regdb
FilesAnalyzed: false

SPDXID: SPDXRef-Package-wireless-regdb-static
PackageLicenseDeclared: ISC
LicenseID: LicenseRef-ISC

ExtractedText: |
  Copyright (c) 2008, Luis R. Rodriguez <mcgrof@gmail.com>
  Copyright (c) 2008, Johannes Berg <johannes@sipsolutions.net>
  Copyright (c) 2008, Michael Green <Michael.Green@Atheros.com>

  Permission to use, copy, modify, and/or distribute this software for any
  purpose with or without fee is hereby granted, provided that the above
  copyright notice and this permission notice appear in all copies.

  THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
  WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
  MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
  ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
  WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
  ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
  OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE
 
EOF

md5sum meta-myConfig/recipes-kernel/wireless-regdb/files/LICENSE.spdx
773199a544a81f68f929217d82bf5183  meta-myConfig/recipes-kernel/wireless-regdb/files/LICENSE.spdx

#↑の「773199a544a81f68f929217d82bf5183」部分を下記のmd5=...の...部分へ記載

cat > meta-myConfig/recipes-kernel/wireless-regdb/wireless-regdb_%.bbappend << EOF
RPROVIDES:${PN}-static= "wireless-regdb-static"
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://LICENSE.spdx"
LIC_FILES_CHKSUM += "file://../LICENSE.spdx;md5=773199a544a81f68f929217d82bf5183"
EOF

#★の「wireless-regdb-static \」をIMAGE_INSTALL:appendへ記載する。
vi build_scarthgap_gen_rpi/conf/local.conf
IMAGE_INSTALL:append = " \
  openssh \
  openssh-sftp-server \
  util-linux \
  usbutils \
  wpa-supplicant \
  iw \
  ebtables \
  dhcp-relay \
  iptables \
  tcpdump \
  bridge-utils \
  wireless-regdb-static \ ★
  bc \
  network-drivers "

#環境変数の設定をする BUILDDIRはbuildする場所によって対象が異なると思うので環境によって変えてほしい。
export BUILDDIR=/home/reki/vol/x86_64/build_scarthgap_gen_rpi
source poky/oe-init-build-env $BUILDDIR

#変更前にbackupする backupはbootOKだった時のもの
cp conf/local.conf conf/local.conf_bk_3_bootOK

#自身のlayerを作成し、加えるます。既にある場合はSkip
cd /home/reki/vol/x86_64
bitbake-layers create-layer meta-myConfig -p 9
rm -rf ./meta-myConfig/recipes-example
cd build_scarthgap_gen_rpi
bitbake-layers add-layer ../meta-myConfig
cat conf/bblayers.conf
#確認:/home/reki/vol/x86_64/meta-myConfigが追加されていること

#再度buildできるか確認
bitbake core-image-minimal

#確認
ls -la /home/reki/vol/x86_64/build_scarthgap_gen_rpi/tmp/work/raspberrypi-poky-linux-gnueabi/core-image-minimal/1.0/rootfs/lib/firmware
total 20
drwxr-xr-x 2 reki reki 4096 Mar  9  2018 .
drwxr-xr-x 7 reki reki 4096 Mar  9  2018 ..
-rw-r--r-- 1 reki reki 5980 Mar  9  2018 regulatory.db
-rw-r--r-- 1 reki reki 1085 Mar  9  2018 regulatory.db.p7s

#↑のregulatory.dbとregulatory.db.p7sがもし入っていない場合は単体でbuildできるか確認
#全部clean
bitbake wireless-regdb -c cleanall
#build
bitbake wireless-regdb

#結果
tree /home/reki/vol/x86_64/build_scarthgap_gen_rpi/tmp/work/all-poky-linux/wireless-regdb/2024.10.07/image
/home/reki/vol/x86_64/build_scarthgap_gen_rpi/tmp/work/all-poky-linux/wireless-regdb/2024.10.07/image
┌── etc
│   ├── wireless-regdb
│   │   ├── pubkeys
│   │   │   ├── wens.key.pub.pem
│   ├── usr
│       ├── lib
│       │   ├── crda
│       │   ├── regulatory.bin
│       │   ├── firmware
│       │       ├── regulatory.db
│       │       ├── regulatory.db.p7s

#再度build
bitbake core-image-minimal
#結果確認
ls -la /home/reki/vol/x86_64/build_scarthgap_gen_rpi/tmp/work/raspberrypi-poky-linux-gnueabi/core-image-minimal/1.0/rootfs/lib/firmware
total 20
drwxr-xr-x 2 reki reki 4096 Mar  9  2018 .
drwxr-xr-x 7 reki reki 4096 Mar  9  2018 ..
-rw-r--r-- 1 reki reki 5980 Mar  9  2018 regulatory.db
-rw-r--r-- 1 reki reki 1085 Mar  9  2018 regulatory.db.p7s

```
#### 1.1.3.5.不揮発性ディスク→ramdisk化して起動する

参考文献
https://mickey-happygolucky.hatenablog.com/entry/2025/03/13/084357
↑そのままだとLICENSEとかでエラー出たりするので少し手直しする  

ただし、initramfsで起動することは一般的なものとすこし異なる。  
一般的なinittamfsはkernel付帯、もしくはcpioイメージでgz圧縮して/dev/ramやtempfsへ展開し起動。  
今回はUbuntuのLiveイメージで使用されるCaspterのtoramオプションのような挙動をさせたい。  
↑をやりたい理由は設定したファイルを通常は不揮発性としたい。  
また、設定をどうしても固定化したい場合のみDiskの/をmountして書き込むようにさせたい。  
設定の保存するイメージはL2/L3SWのruuning-configやsave-configのような感じです。  

これができると何が良いか？  
 →誤った設定しても再起動すればまっさらになれる。  
 →不揮発性のDiskを誰かがmount操作するようにしない前提でウィルス入りされても再起動すればなくなる。  
  [書き込み可能性を減らしたいなら一度起動したらSDカード抜くという手もある]  
 →不揮発性のDiskへの書き込みが頻度が減るため、Disk摩耗が減る。  
 
以下で言うルートFSとは/の場所ではなくこれから起動されるDisk[HDDやSSDなど]の/のこと。  
本来initfamfsの/がルートFSということが正しいが、今回は起動させたいDiskの/をRAMへコピーしRAMを/にして起動させる。  

1.ルートFSをマウント  
2.ルートFSのサイズをduで計算  
3.bcでサイズを1.3倍  
4.tmpfs(RAM)を/dev/shmにマウント  
5.ルートFSのファイルをすべて/dev/shmにコピー  
6.ルートFSをアンマウント  
7.tmpfsをルートFSがマウントされていた場所にmoveマウント  
8.exitしてmountしたtempfsを起動。  

``` sh
cd /home/reki/vol/x86_64

build_scarthgap_gen_rpi/conf/local.confは以下を追加[initramfsのサイズやBOOTのSpaceは環境によって変更。]
# enable uart
ENABLE_UART = "1"

# Bundle initramfs
INITRAMFS_IMAGE = "core-image-minimal-initramfs"
INITRAMFS_IMAGE_BUNDLE = "1"
BOOT_SPACE = "1073741"
INITRAMFS_MAXSIZE = "315400"
IMAGE_FSTYPES_pn-${INITRAMFS_IMAGE} = "${INITRAMFS_FSTYPES}"

# fallback to shell in the initramfs
INITRAMFS_SCRIPTS = "\
                      initramfs-framework-base \
                      initramfs-module-debug \
                    bc \
 "
CMDLINE_DEBUG = "debug shell=after:rootfs"

IMAGE_BOOT_FILES = "${BOOTFILES_DIR_NAME}/* \
                 ${@make_dtb_boot_files(d)} \
               ${KERNEL_IMAGETYPE}-${INITRAMFS_LINK_NAME}.bin;${SDIMG_KERNELIMAGE} \
"
#↑をDiskへ焼いて起動させるとdebug shell=after:rootfsによるrootfs起動は後にできる。ただしexitすると起動してくる。
#以下のコマンドで/rootfs[diskは勝手にmountされた]の/データ→/dev/shmへコピーしmount moveで移動,
#raspberryパイのメモリカードの/bootだけmountされないようにした後にexitしてtempfsを起動させる。
$ size=$(du -sk "/rootfs" | awk '{print $1}')
$ size=$(echo "($size * 1.3)/1" | bc)
$ mkdir -p /dev/shm
$ mount -t tmpfs -o size="$size"k tmpfs /dev/shm
$ cp -a /rootfs/* /dev/shm
$ umount /rootfs
$ mount -o move /dev/shm /rootfs
$ sed -i 's|/dev/mmcblk0p1|#/dev/mmcblk0p1|' /rootfs/etc/fstab
$ exit

起動後、mountするとmmc関連のDiskがなくなる。
dhするとtempfs以外の不揮発性のDiskがなくなることを確認

meta-toramというものを追加
bitbake-layers create-layer -p10 -a ../meta-toram
mkdir -p ../meta-toram/recipes-core/initrdscripts/files
cat > ../meta-toram/recipes-core/initrdscripts/files/91-toram << EOF
#!/bin/sh -e

toram_enabled() {
    return 0
}

toram_run() {
    size=\$(du -sk "\$ROOTFS_DIR" | awk '{print \$1}')
    size=\$(echo "(\$size * 1.3)/1" | bc)

    mkdir -p /dev/shm
    mount -t tmpfs -o size="\$size"k tmpfs /dev/shm
    cp -a \$ROOTFS_DIR/* /dev/shm
    umount \$ROOTFS_DIR
    mount -o move /dev/shm \$ROOTFS_DIR
}
EOF

cat > ../meta-toram/recipes-core/initrdscripts/files/LICENSE << EOF
Copyright (c) <Year> <Your Name>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software...
EOF
md5sum ../meta-toram/recipes-core/initrdscripts/files/LICENSE
27a85940604d5a5877d8df2a57d15dd7  ../meta-toram/recipes-core/initrdscripts/files/LICENSE

cat > ../meta-toram/recipes-core/initrdscripts/initramfs-module-toram_0.1.bb << EOF
SUMMARY = "initramfs-framework module for toram"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://LICENSE;md5=27a85940604d5a5877d8df2a57d15dd7"
RDEPENDS:\${PN} = "initramfs-framework-base bc"

SRC_URI = "file://91-toram"
SRC_URI += "file://LICENSE"

S = "\${WORKDIR}"

do_install() {
    install -d \${D}/init.d
    install -m 0755 \${WORKDIR}/91-toram \${D}/init.d/91-toram
}

FILES:\${PN} = "/init.d/91-toram"

EOF

wksファイルの編集
[sed -i 's|/dev/mmcblk0p1|#/dev/mmcblk0p1|' /rootfs/etc/fstab相当]
mkdir -p ../meta-toram/wic
下記の「part /boot --source bootimg-partition --ondisk mmcblk0 --fstype=vfat --label boot --active --align 4096 --size 100」の/bootのmountpointを削除
 cat ../poky/meta-raspberrypi/wic/sdimage-raspberrypi.wks
# short-description: Create Raspberry Pi SD card image
# long-description: Creates a partitioned SD card image for use with
# Raspberry Pi. Boot files are located in the first vfat partition.

part /boot --source bootimg-partition --ondisk mmcblk0 --fstype=vfat --label boot --active --align 4096 --size 100
part / --source rootfs --ondisk mmcblk0 --fstype=ext4 --label root --align 4096

cat >../meta-toram/wic/sdimage-raspberrypi-toram.wks << EOF
part --source bootimg-partition --ondisk mmcblk0 --fstype=vfat --label boot --active --align 4096 --size 100
part / --source rootfs --ondisk mmcblk0 --fstype=ext4 --label root --align 4096
EOF

$ tree ../meta-toram/
┌── meta-toram
│   ├── conf
│   │   ├── layer.conf
│   ├── COPYING.MIT
│   ├── README
│   ├── recipes-core
│   │   ├── initrdscripts
│   │   │   ├── files
│   │   │   │   ├── 91-toram
│   │   │   │   ├── LICENSE
│   │   │   ├── initramfs-module-toram_0.1.bb
│   ├── recipes-example
│   │   ├── example
│   │   │   ├── example_0.1.bb
│   ├── wic
│       ├── sdimage-raspberrypi-toram.wks

build_scarthgap_gen_rpi/conf/local.confへ以下を記載
INITRAMFS_SCRIPTS = "\
        initramfs-module-toram \
 "
WKS_FILE = "sdimage-raspberrypi-toram.wks"

#リビルド
bitbake core-image-minimal

bootして起動すればOK

#ここからは余談だが、以下のようなコマンドをすれば、/etcと/mnt/rootfs/etcの設定ファイルどおしのdiffを取得できる。
# diff -r /mnt/rootfs/etc /etc
Only in /etc: resolv.conf
Only in /etc: ssh/ssh_host_ecdsa_key
Only in /etc: ssh/ssh_host_ecdsa_key.pub
Only in /etc: ssh/ssh_host_ed25519_key
Only in /etc: ssh/ssh_host_ed25519_key.pub
Only in /etc: ssh/ssh_host_rsa_key
Only in /etc: ssh/ssh_host_rsa_key.pub
Only in /etc: volatile.cache

#しかし、この内容はresolv.confは毎回変わる可能性あり[DHCPなので...subnetが変われば変わる]
#sshは基本一度設定すれば更新かけない限りそのままの鍵になるが現状存在しないので毎回作成となる。
#煩わしいのであればコピー
#voltile.cacheは多分tmpのvoltile関連のcacheっぽいので不要

#もし↑のファイルを除外して変更点をチェックするなら以下。
#diff -r /mnt/rootfs/etc /etc | grep -vE "resolv.conf|ssh_host_.*|volatile.cache"

#この結果をpatchファイルすれば/etcのファイル更新できる。
#またmountして直接変更もできる。

```
