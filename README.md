# Raspbian(buster)でarmv6,v7共に動作するdocker-ceのパッケージング

## 概要

raspbian buster armv6(pi0など)環境上で動作する `docker-ce` のパッケージング

2019/6/30時点では，raspbian buster armv6(pi0など)環境上で動作する `docker-ce` のパッケージが提供されていなかったため，
元々，本リポジトリはでは動作パッケージを作成する方法を示していたが，現在は提供されている．
そのため本リポジトリは自家パッケージングの手順になっている．

クロスコンパイルもできるかもしれないのだが，qemuユーザーモードエミュレーションには面倒そうな箇所がある上，
途中までx86\_64上で実施してみても`GOARCH`等を付けるとdockerdのビルド時にSEGVしたりしていた．
ただの"Hello,World!"をビルドするgo buildでも確率的にSEGVしていたので，
これはアカンとクロスコンパイルは見切っている．

## 前準備

パッケージングスクリプトはdockerイメージを利用するため，パッケージング環境にはdockerが入っていなければならない．
[raspbian buster](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-09-30/2019-09-26-raspbian-buster-lite.zip)を焼いてpi0起動する．
パッケージングにはdockerイメージを複数使う．また，goプログラムのリンカが利用するメモリもpi3までの容量では不足する．
swap領域も必要になるためパッケージングには予め容量の大きなSDカードを使う．8GBだと不足だった．

headlessでsshまでできるようにしておき，以下ssh後

### メモリが不足するので，swapを拡張しておく

```bash
sudo nano -w /etc/dphys-swapfile
```

```
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
deb [arch=armhf] https://download.docker.com/linux/raspbian buster stable
```

### `docker-ce` のインストール

```bash
sudo nano -w /etc/apt/preferences.d/docker.pref
```

```
Package: aufs-tools
Pin: version *
Pin-Priority: -1
```

```bash
wget -O- https://download.docker.com/linux/raspbian/gpg | sudo apt-key add -
sudo apt update
sudo apt install -y docker-ce
sudo usermod -aG docker pi
```

この後，再ログインしてsudoナシでdocker叩けるか確認

```bash
docker images
```

## パッケージング

```console
git clone --depth=1 --branch=v19.03.5 https://github.com/docker/docker-ce
cd docker-ce
sed -i 's/alpine/arm32v6\/alpine/' components/packaging/deb/Makefile
make DOCKER_BUILD_PKGS=raspbian-buster deb
```

pi0なら4時間後くらいに完了する．祈って待つ

```console
find components/packaging/deb/debbuild/raspbian-buster/ -type f
components/packaging/deb/debbuild/raspbian-buster/docker-ce_19.03.5~3-0~raspbian-buster_armhf.buildinfo
components/packaging/deb/debbuild/raspbian-buster/docker-ce_19.03.5~3-0~raspbian-buster.dsc
components/packaging/deb/debbuild/raspbian-buster/docker-ce_19.03.5~3-0~raspbian-buster_armhf.deb # これが欲しかったもの
components/packaging/deb/debbuild/raspbian-buster/docker-ce_19.03.5~3-0~raspbian-buster.tar.gz
components/packaging/deb/debbuild/raspbian-buster/docker-ce-cli_19.03.5~3-0~raspbian-buster_armhf.deb
components/packaging/deb/debbuild/raspbian-buster/docker-ce_19.03.5~3-0~raspbian-buster_armhf.changes
```

成果物を取り出しておく．

```console
scp -r pi@pi0.local:docker-ce/components/packaging/deb/debbuild/raspbian-buster deb
tar cvjf docker-ce-for-raspbian-buster-19.03.5.tar.bz2 deb
```

## 動作確認

[raspbian buster](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-09-30/2019-09-26-raspbian-buster-lite.zip)を焼いて起動．以下，パッケージング時と同様にsshできたとこから

おやくそく

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo reboot
```

作ったパッケージをscp等で `~/deb` 以下に送り込んでおく．

### `containerd.io` のインストール

別途containerd.ioをパッケージングして入れておく

### `docker-ce` のインストール

作成したパッケージを `apt` で

```bash
sudo apt install -y ./deb/docker-ce-cli_19.03.5~3-0~raspbian-buster_armhf.deb --no-install-recommends
sudo apt install -y ./deb/docker-ce_19.03.5~3-0~raspbian-buster_armhf.deb --no-install-recommends
```

### 実行

```bash
sudo docker images
sudo docker run --rm idein/actcast-rpi-app-base:buster echo hello
hello
```

pi3環境でも同様に動作確認
