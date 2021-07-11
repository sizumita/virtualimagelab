# M1 Macにx86のubuntuを入れてsolo5を使ってMirageOSを動かす

MirageOSの研究をするために、M1 Macで動かしたかったが、virtioを入れる方法がよくわからなかったので、x86_64のubuntuを入れてその中で動かしたメモ。

## 環境

- MacBook Pro (13-inch, M1, 2020)
- Apple M1
- メモリ 16 GB
- macOS Big Sur 11.3.1

MirageOSのインストールは [こちら](https://debslink.hatenadiary.jp/entry/20200516/1589616362)を参考にした。

## Dockerでx86_64のUbuntu 20.04をビルドする

```dockerfile
FROM ubuntu:20.04
```

と書いたDockerfileをdocker buildしても、arm64/v8向けのビルドしかできない。そこで、`--platform`引数を使ってプラットフォームを指定してやる必要がある。

その前に、自分の環境でx86_64向けにビルドできるのか、`docker buildx ls`コマンドで調べる。

```zsh
❯ docker buildx ls
NAME/NODE DRIVER/ENDPOINT STATUS  PLATFORMS
default * docker
  default default         running linux/arm64, linux/amd64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

`linux/amd64`とあるので、ビルドできそう。

```zsh
❯ docker build -t mirage . --platform linux/amd64
```

でビルドできた。

## x86_64のDocker imageを実行する

ここで少しハマったのだが、ホストと同じアーキテクチャではないimageを実行するには特権が必要?らしく、`--privileged`引数が必要だった。

```zsh
docker run -it -d --name mirage-1 --privileged mirage
```

## OCaml入りのimageを用意する。

ここまでくればあとは簡単なので、Dockerfileを書いていけば良い。

```Dockerfile
FROM ubuntu:20.04

RUN apt-get update && apt-get update && apt-get install util-linux software-properties-common -y
RUN add-apt-repository ppa:avsm/ppa && apt-get update
RUN apt-get install opam ocaml gcc make bubblewrap m4 pkg-config qemu-kvm -y
RUN opam init -y
RUN eval `opam config env`
RUN opam switch create 4.12.0
RUN eval $(opam env)
RUN opam install mirage -y
RUN eval $(opam env)
```

このDockerfileではMirageOSのインストールまでやっている。

## mirage-skeletonのtutorialを動かしてみる

動作確認のため、mirage-skeltonのtutorialを動かしてみる。

```bash
# git clone https://github.com/mirage/mirage-skeleton.git
# cd mirage-skelton/tutorial/hello
```

ビルドのための設定をしていく。ここでmirageコマンドが実行できなかったら、`eval $(opam env)`と実行すると治る(はず)。

```bash
# mirage configure -t virtio
```

次に、make dependを実行して、ビルドに必要なツールやパッケージをインストールする。

```bash
# make depend
opam pin add -k path --no-action --yes mirage-unikernel-hello-virtio . && opam depext --yes --update mirage-unikernel-hello-virtio ; opam pin remove --no-action mirage-unikernel-hello-virtio
[WARNING] Running as root is not recommended
Package mirage-unikernel-hello-virtio does not exist, create as a NEW package? [Y/n] y
[mirage-unikernel-hello-virtio.~dev] synchronised (file:///mirage-skeleton/tutorial/hello)
[WARNING] Failed checks on mirage-unikernel-hello-virtio package definition from source at file:///mirage-skeleton/tutorial/hello:
  warning 49: The following URLs don't use version control but look like version control URLs: "https://github.com/mirage/mirage-skeleton.git#master"
mirage-unikernel-hello-virtio is now pinned to file:///mirage-skeleton/tutorial/hello (version ~dev)
...
Done.
```

最後に、makeをしてhello.virtioを生成する。

```bash
# make
mirage build
[WARNING] Running as root is not recommended
[WARNING] var was deprecated in version 2.1 of the opam CLI. Use opam var instead or set OPAMCLI environment variable to 2.0.
config.exe: [WARNING] pkg-config solo5-bindings-virtio --variable=ld returned nothing, using ld
```

lsコマンドでhello.virtioが生成されていることを確認できる。

```bash
# ls
Makefile  _build  config.ml  dune  dune-project  dune.build  dune.config  hello.virtio  hello_libvirt.xml  key_gen.ml  main.ml  mirage-unikernel-hello-virtio.opam  myocamlbuild.ml  unikernel.ml
```

solo5-virtio-runコマンドで実行すると、5回helloを送信してからexitしているのがわかる。

```bash
# solo5-virtio-run hello.virtio
+ exec qemu-system-x86_64 -cpu Westmere -m 128 -nodefaults -no-acpi -display none -serial stdio -device isa-debug-exit -kernel /mirage-skeleton/tutorial/hello/hello.virtio
            |      ___|
  __|  _ \  |  _ \ __ \
\__ \ (   | | (   |  ) |
____/\___/ _|\___/____/
Solo5: Bindings version v0.6.8
Solo5: Memory map: 128 MB addressable:
Solo5:   reserved @ (0x0 - 0xfffff)
Solo5:       text @ (0x100000 - 0x1d9fff)
Solo5:     rodata @ (0x1da000 - 0x20cfff)
Solo5:       data @ (0x20d000 - 0x2b8fff)
Solo5:       heap >= 0x2b9000 < stack < 0x8000000
Solo5: Clock source: TSC, frequency estimate is 1002585410 Hz
2021-07-11 11:05:19 -00:00: INF [application] hello
2021-07-11 11:05:20 -00:00: INF [application] hello
2021-07-11 11:05:21 -00:00: INF [application] hello
2021-07-11 11:05:22 -00:00: INF [application] hello
Solo5: solo5_exit(0) called
```

## 最後に

ここまでやっていて気が付いたのだが、solo5-virtio-runはARMに対応しているので、virtioもARMに対応しているのでは...

ただ、`mirage configure -t virtio`コマンドでコケたので実行できなさそう。わかる方いたら教えて欲しいです。

次は、http serverを立ててみたい。
