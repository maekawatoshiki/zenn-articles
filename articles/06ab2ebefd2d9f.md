---
title: "RV64G環境向けにCPythonをビルドする"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "riscv"]
published: true
---

RISC-V（RV64G）環境でCPython（3.12.4）を利用する機会があったため，ビルド手順をまとめました．

## GCCの準備

まず，RISC-Vのツールチェインをインストールします．
ここでは，インストール先を `$HOME/riscv/rv64` とします．


```bash
#!/bin/sh -eux

export RISCV=${RISCV:-$HOME/riscv}

git clone https://github.com/riscv-collab/riscv-gnu-toolchain
cd riscv-gnu-toolchain

mkdir -p build
cd build

../configure \
    --prefix=$RISCV/rv64 \
    --with-arch=rv64imafd_zifencei \
    --with-abi=lp64d \
    --with-cmodel=medany \
    --disable-multilib # NOTE: Not to use C extension

make linux -j$(nproc)
make install
```

## CPython（3.12.4）のビルド

続いて，CPython をビルドします．いくつか必要なライブラリも前もってビルドしておきます．
また，ビルド時にはホスト側で動作しているPythonが必要です．

*tips:* ビルド時にいくつかのPythonモジュールを無効化することで，依存を減らすことができます．
必要なものまで無効にする必要はありませんが，ビルドの上手くいかない依存ライブラリが多い印象でした．
無効化するモジュールは，`config.site` にて指定します．

*tips:* はじめは全体を静的リンクする予定だったのですが，[Building Python Statically](https://wiki.python.org/moin/BuildStatically) 等を参考にしても至る所でリンクエラーが発生したため諦めました．


```bash
#!/bin/sh -eux

export CC="$HOME/riscv/rv64/bin/riscv64-unknown-linux-gnu-gcc"
export CXX="$HOME/riscv/rv64/bin/riscv64-unknown-linux-gnu-g++"
export LD="$HOME/riscv/rv64/bin/riscv64-unknown-linux-gnu-ld"
export AR="$HOME/riscv/rv64/bin/riscv64-unknown-linux-gnu-ar"
export STRIP="$HOME/riscv/rv64/bin/riscv64-unknown-linux-gnu-strip"
export RANLIB="$HOME/riscv/rv64/bin/riscv64-unknown-linux-gnu-ranlib"
export OBJDUMP="$HOME/riscv/rv64/bin/riscv64-unknown-linux-gnu-objdump"

#
# Build xz
#
(
git clone https://github.com/tukaani-project/xz --depth 1
cd xz
./autogen.sh
./configure --host=riscv64-unknown-linux-gnu --prefix=$PWD/target
make -j
make install
)

#
# Build libffi
#
(
git clone https://github.com/libffi/libffi --depth 1 --branch v3.4.6
cd libffi
./autogen.sh
./configure --host=riscv64-unknown-linux-gnu --prefix=$PWD/target
make -j
make install
)

#
# Build zlib
#
(
git clone https://github.com/madler/zlib --depth 1 --branch v1.3.1
cd zlib
./configure --prefix=$PWD/target
make -j
make install
)

#
# Build readline
#
(
wget http://ftp.gnu.org/pub/gnu/readline/readline-8.2.tar.gz
tar xvf readline-8.2.tar.gz
cd readline-8.2
./configure --host=riscv64-unknown-linux-gnu --prefix=$PWD/target
make -j
make install
)

#
# Build cpython
#
(
export CONFIG_SITE=./config.site
export PATH="$HOME/riscv/rv64/bin:${PATH}"
export CFLAGS="-mcmodel=medany -I$PWD/zlib/target/include -I$PWD/libffi/target/include -I$PWD/xz/target/include -I$PWD/readline-8.2/target/include"
export CPPFLAGS="${CFLAGS}"
export LDFLAGS="-mcmodel=medany -L$PWD/zlib/target/lib -L$PWD/libffi/target/lib -L$PWD/xz/target/lib -L$PWD/readline-8.2/target/lib"

git clone https://github.com/python/cpython --depth 1 --branch v3.12.4
cd cpython

cat <<EOF > $CONFIG_SITE
ac_cv_file__dev_ptmx=no
ac_cv_file__dev_ptc=no
py_cv_module__uuid=n/a
py_cv_module_nis=n/a
py_cv_module__bz2=n/a
py_cv_module__curses=n/a
py_cv_module__curses_panel=n/a
py_cv_module__dbm=n/a
py_cv_module__gdbm=n/a
py_cv_module__hashlib=n/a
py_cv_module__ssl=n/a
py_cv_module__tkinter=n/a
EOF

./configure --host=riscv64-unknown-linux-gnu --build=riscv64 --with-build-python=$(which python3.12) --disable-ipv6
make -j$(nproc)
)
```

## 動作確認

QEMUを利用して，ビルドしたCPythonの動作を確認します．
無効化したモジュールはあるものの，基本的な部分は動作するようです．

```bash
$ PYTHONPATH=$PWD/cpython/Lib qemu-riscv64 -L ~/riscv/rv64/sysroot/ ./cpython/python
Python 3.12.4 (tags/v3.12.4:8e8a4ba, Jul 22 2024, 13:14:02) [GCC 12.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import math
>>> math.sin(3.1415/2)
0.999999998926914
>>> import platform
>>> platform.machine()
'riscv64'
>>>
```
