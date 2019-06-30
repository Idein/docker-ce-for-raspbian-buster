# Raspbian(buster)でarmv6,v7共に動作するdocker-ceのパッケージング

## 概要

2019/6/30現在，raspbian buster armv6(pi0など)環境上で動作する `docker-ce` のパッケージが提供されていない．
stretchではかろうじて最新ひとつ手前のバージョンがraspi共通で動作させることのできるものであったが，
それもcontainerdの古さによりbuster上では動作しなくなってしまったことが確認された．
そのため，本ドキュメントではraspi共通で動作させることのできる `docker-ce` パッケージを作成する手順を示し，
その成果物を同梱しておく．

本手順はpi0環境をメインに記載しているが，平行してpi3でも実施して同じようにパッケージングを行っている．
結論，pi3上で生成したパッケージでもpi0,pi3環境共に動作に問題は無いようだった．
コンパイル時間に大差があるため，問題無いならpi3でビルドしたい．
ただ，途中 `uname -m` でARCHを拾っている箇所があり，armv6lになるかarmv7lになるか変わるのでとりあえずpi0でやっておく．

クロスコンパイルもできるかもしれないのだが，qemuユーザーモードエミュレーションには面倒そうな箇所がある上，
途中までx86_64上で実施してみても`GOARCH`等を付けるとdockerdのビルド時にSEGVしたりしていた．
ただの"Hello,World!"をビルドするgo buildでも確率的にSEGVしていたので，
これはアカンとクロスコンパイルは見切った．

## 前準備

パッケージングスクリプトはdockerイメージを多用するため，パッケージング環境にはdockerが入ていなければならない．
[raspbian stretch](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-04-09/2019-04-08-raspbian-stretch-lite.zip) (でなら armv6,v7 両方で動く `docker-ce` packageがあるので)を焼いてpi0起動する．
パッケージングにはdockerイメージを複数使う．また，goプログラムのリンカが利用するメモリもpi3までの容量では不足する．
swap領域も必要になるためパッケージングには予め容量の大きなSDカードを使う．8GBだと不足だった．

headlessでsshまでできるようにしておき，以下ssh後

### メモリが不足するので，swapを拡張しておく

```bash
sudo nano -w /etc/dphys-swapfile
-CONF_SWAPSIZE=100
+CONF_SWAPSIZE=2048
```

### upgrade

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo reboot
```

## Docker環境構築

### raspbian docker リポジトリの設定

```bash
sudo nano -w /etc/apt/sources.list.d/docker.list
```

```
deb [arch=armhf] https://download.docker.com/linux/raspbian stretch stable
```

### `docker-ce` のインストール

`18.06.1~ce~3-0~raspbian` は raspbian stretch armv6,v7 で動くのでとりあえずビルド環境用にpinningして使う．

```bash
sudo nano -w /etc/apt/preferences.d/docker.pref
```

```
Package: docker-ce
Pin: version 18.06.1~ce~3-0~raspbian
Pin-Priority: 990

Package: aufs-tools
Pin: version *
Pin-Priority: -1
```

```bash
wget -O- https://download.docker.com/linux/raspbian/gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install -y docker-ce
sudo usermod -aG docker pi
```

この後，再ログインしてsudoナシでdocker叩けるか確認

```bash
docker images
```

## パッケージング

### golangイメージのベースイメージ作成

raspbian向けgolangイメージを作るためのベースイメージをraspbianで作る

```bash
git clone git@github.com:docker-library/buildpack-deps.git
cd buildpack-deps
git rev-parse HEAD
fa587a0d10fd627c1890345db640d1a55cfab3fc # 念のため，作業時のHEADを記録
cd buster/curl
nano -w Dockerfile
-FROM debian:buster
+FROM idein/actcast-rpi-app-base:buster # 大本のベースイメージをraspbian buster のdebootstrap最小構成に
docker build . -t idein/buildpack-deps:buster-curl
cd ../scm
nano -w Dockerfile
-FROM buildpack-deps:buster-curl
+FROM idein/buildpack-deps:buster-curl
docker build . -t idein/buildpack-deps:buster-scm
cd
```

### raspbian向けのgolangイメージ作成

```bash
git clone git@github.com:docker-library/golang.git
cd golang
git rev-parse HEAD
fc23a7764ddaa10f6b36356018ff11d684c58487 # 念のため，作業時のHEADを記録
cd 1.12/stretch
nano -w Dockerfile
-FROM buildpack-deps:stretch-scm
+FROM idein/buildpack-deps:buster-scm
docker build . -t idein/golang:1.12-buster
cd
```

### `docker-ce` パッケージング

```bash
git clone git@github.com:docker/docker-ce-packaging.git
cd docker-ce-packaging
git checkout v18.09.6
git clone git@github.com:docker/cli.git
git -C cli checkout 18.09
git clone git@github.com:docker/engine.git
git -C engine checkout 18.09
nano -w image/Dockerfile.engine

     btrfs-tools \
+    libbtrfs-dev \ # busterからは必要物がこのパッケージにも分割された
     ca-certificates \

-    git checkout -q "$RUNC_COMMIT" && \
+    git checkout -q 2b18fe1d885ee5083ef9f0838fee39b62d653e30 && \

cp -r deb/raspbian-{stretch,buster}
nano -w deb/raspbian-buster/Dockerfile

-ARG BUILD_IMAGE=resin/rpi-raspbian:stretch
+ARG BUILD_IMAGE=idein/actcast-rpi-app-base:buster

 ENV DISTRO raspbian
-ENV SUITE stretch
+ENV SUITE buster

sed -i 's/alpine/arm32v6\/alpine/' deb/Makefile            # alpineイメージはnativeビルドだと動作しない
sed -i 's/raspbian-stretch/raspbian-buster/' deb/Makefile
nohup make VERSION=18.09.6 DOCKER_BUILD_PKGS=raspbian-buster ENGINE_DIR=$(pwd)/engine CLI_DIR=$(pwd)/cli GO_BASE_IMAGE=idein/golang GO_VERSION=1.12-buster deb &
```

pi0なら6〜7時間後完了する．祈って待つ

```
find deb/debbuild/ -type f
deb/debbuild/raspbian-buster/docker-ce_18.09.6~3-0~raspbian-buster.dsc
deb/debbuild/raspbian-buster/docker-ce_18.09.6~3-0~raspbian-buster.tar.gz
deb/debbuild/raspbian-buster/docker-ce_18.09.6~3-0~raspbian-buster_armhf.changes
deb/debbuild/raspbian-buster/docker-ce-cli_18.09.6~3-0~raspbian-buster_armhf.deb # これと
deb/debbuild/raspbian-buster/docker-ce_18.09.6~3-0~raspbian-buster_armhf.deb     # これが欲しかったもの
deb/debbuild/raspbian-buster/docker-ce-build-deps_1.0_all.deb
deb/debbuild/raspbian-buster/docker-ce_18.09.6~3-0~raspbian-buster_armhf.buildinfo
```

成果物と `nohup.out` (パッケージングのログ)を取り出しておく．

```bash
scp -r pi@pi0.local:docker-ce-packaging/deb/debbuild/raspbian-buster raspbian-buster-pi0
scp    pi@pi0.local:docker-ce-packaging/nohup.out raspbian-buster-pi0/
```

## 動作確認

[raspbian buster](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-06-24/2019-06-20-raspbian-buster-lite.zip)を焼いて起動．以下，パッケージング時と同様にsshできたとこから

おやくそく

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo reboot
```

作ったパッケージをscp等で `~/raspbian-buster-pi0` 以下に送り込んでおく．

### Hypriotリポジトリの登録

必要なのは [`containerd.io`](https://packagecloud.io/Hypriot/rpi/packages/raspbian/stretch/containerd.io_1.2.6-1_armhf.deb)．
Hypriotリポジトリは設定スクリプトがあるが，まだbusterリポジトリが無いのでbuster環境ではupdateが失敗する．stretchのものでいい．

```bash
sudo nano -w /etc/apt/sources.list.d/Hypriot_rpi.list
```

```
deb https://packagecloud.io/Hypriot/rpi/raspbian/ stretch main
#deb-src https://packagecloud.io/Hypriot/rpi/raspbian/ stretch main
```

```bash
wget -O- https://packagecloud.io/Hypriot/rpi/gpgkey | sudo apt-key add -
sudo apt-get update
```

### pinning

作った `docker-ce` パッケージとHypriotの `containerd.io` パッケージを対象に

```bash
sudo nano -w /etc/apt/preferences.d/docker.pref
```

```
Package: docker-ce
Pin: version 18.09.6~3-0~raspbian-buster
Pin-Priority: 990

Package: containerd.io
Pin: version 1.2.6-1
Pin-Priority: 990
```

### `containerd.io` のインストール

`docker-ce` の依存関係

```bash
sudo apt-get install -y containerd.io
```

### `docker-ce` のインストール

作成したパッケージを `dpkg` で

```bash
cd raspbian-buster-pi0
sudo dpkg -i docker-ce-cli_18.09.6~3-0~raspbian-buster_armhf.deb
sudo dpkg -i docker-ce_18.09.6~3-0~raspbian-buster_armhf.deb
```

### 実行

```bash
sudo docker images
sudo docker run --rm idein/actcast-rpi-app-base:buster echo hello
hello
```

pi3環境でも同様に動作確認
