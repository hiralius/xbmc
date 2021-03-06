![Kodi Logo](resources/banner_slim.png)

# rootfs(のコピー)を使ったRaspberry Pi用クロスビルドの手順

公式の手順は、Kodiのすべての依存ライブラリをビルドする上に、
ビルドに必要なnativeのツールもすべてビルドするようになっているため
時間・ディスクスペースを非常に多く消費する。
またクロスビルドの設定では非常にエラーが起きやすくなっている。
そこで、依存ライブラリはできるだけrpi(rpi1, raspbian Stretch)にインストールしたパッケージを使い、
nativeツールはビルドホスト上のものを使用する場合の手順を以下に述べる。

## ルートファイルシステムのコピーの用意

1. 依存ライブラリをrpiに事前にインストール
- 面倒な場合は、ディストリビューションのkodiパッケージの情報を調べて、
  ビルドに必要なdevパッケージをすべてインストールすればよい。
- 依存ライブラリはdocs/README.{Linux, Ubuntsu}.mdに挙げられているが、
  オプション扱いのものが含まれている上、ビルドホストで必要なパッケージも含まれている。
  必須のライブラリは、./CMakeLists.txtの required_deps 及びcmake/platform/rbpi.cmake
  に挙げられている。具体的なライブラリ名はcmake/modules/FindXXXX.cmakeを参照。
- 必須ライブラリ自体の依存ライブラリも必要となること、-devパッケージが必要であることに注意。
- 必須ライブラリの内、ffmpeg, crossguid, libfmt, fstrcmp, RapidJSON, flatbufferは
  インストール不要。(後のビルド処理内でスタティックライブラリとして自動的にビルド・リンクされる)
- その他の依存ライブラリも、後ほどスタティックライブラリとしてビルドすることもできるので、
  Kodiだけのためにここで必ずしもインストールしておかなくてもよい。
- ただし、libinputはうまくクロスビルドできないのでこの段階でインストールしておくと良い。
- xkbcommonも実行時にエラーが出るので、この段階でインストールしておく
- ./CMakeList.txtでoptional_depsとして挙げられているライブラリについては、
  自分が使用する機能に関係なければインストールしなくても良い。
- KodiのVideo再生の機能として、ISDBのMULTI2暗号化されたままのファイル等も再生したい場合は、
  libdemulti2 と {libpcscliteかlibyakisoba}のインストールも必要。\
  NOTE: tvheadendからストリーミングして視聴する分には不要。(tvheadend側で復号するため)

2. rpiルートファイルシステムのコピー
- ファイルシステム全体をコピーする必要はなく、/opt/vc, /lib, /usr/{include,lib}があればOK。
以下では、そのパスを/opt/rpi/rootfsとする。

## クロスビルド環境の用意

1. コピーしたファイルシステムで、絶対パス指定のシンボリックリンクを修正 \
(のちのconfigure/ビルド時にエラーになるため)
```
$ find /opt/rpi/rootfs -type l -lname /\* -execdir bash -c \
      'ln -sf /opt/rpi/rootfs$(readlink {}) {}' \;
```
2. クロスビルドのtool chainの用意
- 簡略化のため、ここではプリビルトされたツールを使用
```
$ cd /opt/rpi; git clone --depth 1 https://github.com/raspberrypi/tools
```
この中のarm-bcm2708/arm-rpi-4.9.3-linux-gnueabihfを使う。

3. リンカースクリプト内の絶対パスの修正
- opt/rpi/tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/arm-linux-gnueabihf/sysroot/usr/lib/にあるlibc.soを以下のように変換する。 libpthread.soも同様。
```
GROUP ( /lib/libc.so.6 /usr/lib/libc_nonshared.a  AS_NEEDED ( /lib/ld-linux-armhf.so.3 ) )
↓
GROUP ( libc.so.6 libc_nonshared.a  AS_NEEDED ( ld-linux-armhf.so.3 ) )
```

## クロスビルドする依存ライブラリとnativeツールの格納先の準備

1. 依存ライブラリとネイティブツールのための"ビルドディレクトリ"を作成
```
$ mkdir -p /opt/rpi/kodi-rpi
```
2. Kodiのソースの用意
```
$ git clone https://github.com/0p1pp1/xbmc kodi
```
3. 不足している依存ライブラリのクロスビルドのためのconfigure
```
$ cd kodi/tools/depends
$ ./bootstrap
$ ./configure --host=arm-linux-gnueabihf --prefix=/opt/rpi/kodi-rpi \
      --with-toolchain=/opt/rpi/tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf \
      --with-firmware=/opt/rpi/rootfs --with-platform=raspberry-pi \
      [--with-ffmpeg_options="--cpu=arm1176jzf-s --enable-libdemulti2" \]
      --disable-debug --with-sdk-path=/opt/rpi/rootfs
```
([]内はDEMULTI2復号しながらの再生機能が必要な場合のみ)
4. ネイティブツールはビルドホストにインストール済みのものを利用するための細工
```
$ cd /opt/rpi/kodi-rpi
$ mkdir native-work; mv x86_64-linux-gnu-native native-upper
$ mkdir x86_64-linux-gnu-native
$ sudo mount -t overlay -o lowerdir=/usr,upperdir=native-upper,workdir=native-work \
    overlay x86_64-linux-gnu-native
```

## ビルド

1. 必要なネイティブツールのビルド
```
$ cd /opt/rpi/kodi/tools/depends;
$ make -C native/TexturePacker
$ make -C native/JsonSchemaBuilder
```
2. 不足しているライブラリをクロスビルド(スタティックライブラリとして)
```
$ export PKG_CONFIG_PATH=/opt/rpi/rootfs/usr/lib/arm-linux-gnueabihf/pkgconfig
$ cd /opt/rpi/kodi/tools/depends
$ make -C target/XXX
....
```
XXXに必要な依存ライブラリが不足していると、上記ステップは(configure中に)エラーで停止する。
不足ライブラリは、エラー出力(の末尾付近)にCould NOT find YYY (...)のようなメッセージとして示されるので、
tools/depends/target/{libyyy, yyy}等を探し、同様に`make -C target/libyyy`でビルドする。
それが成功したら、`make -C target/XXX distclean`してから`make -C target/XXX`でやり直す。

//$ make -C libfstrcmp # fstrcmpは(内部ビルドとかで)cmakeでクロスビルドするとエラーになる

3. Kodi本体のクロスビルド
```
$ make -C target/cmakebuildsys # 本体のconfigure
```
必須ライブラリが見つからないとエラーで停止するので、上記2.の手順でビルド、インストールした後、
再度`make -C target/cmakebuildsys`を実行する。これをエラーが出なくなるまで繰り返す。
```
$ cd /opt/rpi/kodi/build # 本体のビルドディレクトリ
$ cmake --build .
```
4. 追加のプラグインのビルド
```
$ cd /opt/rpi/kodi/tools/depend/target/binary-addons
$ make ADDONS="pvr.hts" PREFIX=/opt/rpi/kodi/build/addons CMAKE_EXTRA=-DPACKAGE_ZIP=ON
```

5. インストール・実行例
```
$ rsync -avzh /opt/rpi/kodi/build/{kodi-rbpi, addons, media, system, userdata} \
  pi@raspberry-pi:/home/pi/kodi

$ # raspberry-pi上で
$ ./kodi/kodi-rbpi
```

# Raspberry Pi build guide
This guide has been tested with Ubuntu 16.04 (Xenial) x86_64 and 18.04 (Bionic). It is meant to cross-compile Kodi for the Raspberry Pi using **[Kodi's unified depends build system](../tools/depends/README.md)**. Please read it in full before you proceed to familiarize yourself with the build procedure.

If you're looking to build Kodi natively using **[Raspbian](https://www.raspberrypi.org/downloads/raspbian/)**, you should follow the **[Ubuntu guide](README.Ubuntu.md)** instead. Several other distributions have **[specific guides](README.md)** and a general **[Linux guide](README.Linux.md)** is also available.

## Table of Contents
1. **[Document conventions](#1-document-conventions)**
2. **[Install the required packages](#2-install-the-required-packages)**
3. **[Get the source code](#3-get-the-source-code)**  
  3.1. **[Get Raspberry Pi tools and firmware](#31-get-raspberry-pi-tools-and-firmware)**
4. **[Build tools and dependencies](#4-build-tools-and-dependencies)**
5. **[Build Kodi](#5-build-kodi)**

## 1. Document conventions
This guide assumes you are using `terminal`, also known as `console`, `command-line` or simply `cli`. Commands need to be run at the terminal, one at a time and in the provided order.

This is a comment that provides context:
```
this is a command
this is another command
and yet another one
```

**Example:** Clone Kodi's current master branch:
```
git clone https://github.com/xbmc/xbmc kodi
```

Commands that contain strings enclosed in angle brackets denote something you need to change to suit your needs.
```
git clone -b <branch-name> https://github.com/xbmc/xbmc kodi
```

**Example:** Clone Kodi's current Krypton branch:
```
git clone -b Krypton https://github.com/xbmc/xbmc kodi
```

Several different strategies are used to draw your attention to certain pieces of information. In order of how critical the information is, these items are marked as a note, tip, or warning. For example:
 
**NOTE:** Linux is user friendly... It's just very particular about who its friends are.  
**TIP:** Algorithm is what developers call code they do not want to explain.  
**WARNING:** Developers don't change light bulbs. It's a hardware problem.

**[back to top](#table-of-contents)** | **[back to section top](#1-document-conventions)**

## 2. Install the required packages
Install build dependencies needed to cross-compile Kodi for the Raspberry Pi:
```
sudo apt install autoconf bison build-essential curl default-jdk gawk git gperf libcurl4-openssl-dev zlib1g-dev
```

**[back to top](#table-of-contents)**

## 3. Get the source code
Change to your `home` directory:
```
cd $HOME
```

Clone Kodi's current master branch:
```
git clone https://github.com/xbmc/xbmc kodi
```

### 3.1. Get Raspberry Pi tools and firmware
Clone Raspberry Pi tools:
```
git clone https://github.com/raspberrypi/tools --depth=1
```

Clone Raspberry Pi firmware:
```
git clone https://github.com/raspberrypi/firmware --depth=1
```

**[back to top](#table-of-contents)**

## 4. Build tools and dependencies
Create target directory:
```
mkdir $HOME/kodi-rpi
```

Prepare to configure build:
```
cd $HOME/kodi/tools/depends
./bootstrap
```

**TIP:** Look for comments starting with `Or ...` and only execute the command(s) you need.

Configure build for Raspberry Pi 1:
```
./configure --host=arm-linux-gnueabihf --prefix=$HOME/kodi-rpi --with-toolchain=$HOME/tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf --with-firmware=$HOME/firmware --with-platform=raspberry-pi --disable-debug
```

Or configure build for Raspberry Pi 2 and 3:
```
./configure --host=arm-linux-gnueabihf --prefix=$HOME/kodi-rpi --with-toolchain=$HOME/tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf --with-firmware=$HOME/firmware --with-platform=raspberry-pi2 --disable-debug
```

Build tools and dependencies:
```
make -j$(getconf _NPROCESSORS_ONLN)
```

**TIP:** By adding `-j<number>` to the make command, you can choose how many concurrent jobs will be used and expedite the build process. It is recommended to use `-j$(getconf _NPROCESSORS_ONLN)` to compile on all available processor cores. The build machine can also be configured to do this automatically by adding `export MAKEFLAGS="-j$(getconf _NPROCESSORS_ONLN)"` to your shell config (e.g. `~/.bashrc`).

**[back to top](#table-of-contents)** | **[back to section top](#4-build-tools-and-dependencies)**

## 5. Build Kodi
Configure CMake build:
```
cd $HOME/kodi
make -C tools/depends/target/cmakebuildsys
```

Build Kodi:
```
cd $HOME/kodi/build
make -j$(getconf _NPROCESSORS_ONLN)
```

Install to target directory:
```
make install
```

After the build process is finished, you can find the files ready to be installed inside `$HOME/kodi-rpi`. Look for a directory called `raspberry-pi-release` or `raspberry-pi2-release`.

**[back to top](#table-of-contents)**


