+++
title = "为raspberry pi 4 构建交叉编译工具链"
date = 2021-06-13

[taxonomies]
tags = ["raspberry pi", "crosstool-ng"]
+++


1. 安装各种开发工具依赖包
```bash
sudo apt-get install autoconf automake bison bzip2 cmake flex g++ gawk gcc gettext git gperf help2man libncurses5-dev libstdc++6 libtool libtool-bin make patch python3-dev rsync texinfo unzip wget xz-utils
```
2. 克隆，配置，编译，安装crosstool-ng
```bash
git clone https://github.com/crosstool-ng/crosstool-ng.git
cd crosstool-ng
git checkout crosstool-ng-1.24.0
./bootstrap
./configure  --prefix=`pwd`
make 
make install
```
3. 以rpi3 作为基线配置
crosstool-ng  自带不少配置模板，其中包括rpi 3的配置，可以在此基础上更改为rpi4的配置,  crosstool-ng 的list-samples 命令用于查看自带模板列表:
```bash
./ct-ng  list-samples
```
output:
```bash
Status  Sample name
[L...]   aarch64-rpi3-linux-gnu
[L..X]   aarch64-unknown-linux-android
[L...]   aarch64-unknown-linux-gnu
[L...]   aarch64-unknown-linux-uclibc
[L...]   alphaev56-unknown-linux-gnu
[L...]   alphaev67-unknown-linux-gnu
[L...]   arc-arc700-linux-uclibc
[L...]   arc-multilib-elf32
[L...]   arc-multilib-linux-uclibc
[L...]   arm-bare_newlib_cortex_m3_nommu-eabi
[L...]   arm-cortex_a15-linux-gnueabihf
[L..X]   arm-cortexa5-linux-uclibcgnueabihf
[L...]   arm-cortex_a8-linux-gnueabi
[L..X]   arm-cortexa9_neon-linux-gnueabihf
[L..X]   x86_64-w64-mingw32,arm-cortexa9_neon-linux-gnueabihf
.........
.........
```
使用show-${item-name} 命令可以查看某个配置模板详情:
```bash
./ct-ng  show-aarch64-rpi3-linux-gnu
```
output:
```bash
[L...]   aarch64-rpi3-linux-gnu
    Languages       : C,C++
    OS              : linux-4.20.8
    Binutils        : binutils-2.32
    Compiler        : gcc-8.3.0
    C library       : glibc-2.29
    Debug tools     : gdb-8.2.1
    Companion libs  : expat-2.2.6 gettext-0.19.8.1 gmp-6.1.2 isl-0.20 libiconv-1.15 mpc-1.1.0 mpfr-4.0.2 ncurses-6.1 zlib-1.2.11
    Companion tools :
```
使用rpi3作为配置基础:
```bash
./ct-ng   aarch64-rpi3-linux-gnu
```
然后使用menuconfig 命令定制一些配置项：
```bash
./ct-ng   menuconfig
```
{{ image(src="https://oscimg.oschina.net/oscnet/up-03990a61e361529502f436885c4586f3b4b.png",  position="left") }}

{{ image(src="https://oscimg.oschina.net/oscnet/up-15be68da5618c064fba843db5a0c9392519.png", position="left") }}
{{ image(src="https://oscimg.oschina.net/oscnet/up-423967ee1085df1c9be61a77642e27c0453.png", position="left") }}

图1：把工具链目录的只读属性关闭，允许后续安装其他开发库；

图2:   rpi4 使用的cpu是 Cortex-A72， 所以替换成cortex-a72,允许编译器生成更好的汇编代码；

图3：工具链的vendor string 改成rpi4；

3. 构建工具链
```bash
./ct-ng   build
```
编译后，生成的工具链在~/x-tools/aarch64-rpi4-linux-gnu  目录下:
```bash
ls  ~/x-tools/aarch64-rpi4-linux-gnu/bin/
```
output:
```bash
aarch64-rpi4-linux-gnu-addr2line     aarch64-rpi4-linux-gnu-gcc-8.3.0      aarch64-rpi4-linux-gnu-ldd
aarch64-rpi4-linux-gnu-ar            aarch64-rpi4-linux-gnu-gcc-ar         aarch64-rpi4-linux-gnu-ld.gold
aarch64-rpi4-linux-gnu-as            aarch64-rpi4-linux-gnu-gcc-nm         aarch64-rpi4-linux-gnu-nm
aarch64-rpi4-linux-gnu-c++           aarch64-rpi4-linux-gnu-gcc-ranlib     aarch64-rpi4-linux-gnu-objcopy
aarch64-rpi4-linux-gnu-cc            aarch64-rpi4-linux-gnu-gcov           aarch64-rpi4-linux-gnu-objdump
aarch64-rpi4-linux-gnu-c++filt       aarch64-rpi4-linux-gnu-gcov-dump      aarch64-rpi4-linux-gnu-populate
aarch64-rpi4-linux-gnu-cpp           aarch64-rpi4-linux-gnu-gcov-tool      aarch64-rpi4-linux-gnu-ranlib
aarch64-rpi4-linux-gnu-ct-ng.config  aarch64-rpi4-linux-gnu-gdb            aarch64-rpi4-linux-gnu-readelf
aarch64-rpi4-linux-gnu-dwp           aarch64-rpi4-linux-gnu-gdb-add-index  aarch64-rpi4-linux-gnu-size
aarch64-rpi4-linux-gnu-elfedit       aarch64-rpi4-linux-gnu-gprof          aarch64-rpi4-linux-gnu-strings
aarch64-rpi4-linux-gnu-g++           aarch64-rpi4-linux-gnu-ld             aarch64-rpi4-linux-gnu-strip
aarch64-rpi4-linux-gnu-gcc           aarch64-rpi4-linux-gnu-ld.bfd
```
将此目录加入PATH环境变量:
```bash
export PATH=$PATH:~/x-tools/aarch64-rpi4-linux-gnu/bin
```
编译测试hello world代码hello.c:
```c
#include <stdio.h>
#include <stdlib.h>
int main (int argc, char *argv[])
{
    printf ("Hello, world!\n");
    return 0;
}
```
交叉编译:
```bash
aarch64-rpi4-linux-gnu-gcc  hello.c  -o hello
```
```bash
file  ./hello
```
output:
```bash
./hello: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, for GNU/Linux 4.20.8, with debug_info, not stripped
```
copy  测试二进制hello 到rpi4 开发板, 测试执行:
```bash
scp ./hello   rpi4w:~/
./hello
```
4.  交叉编译第三方库
默认工具链中只包含了基本的c, c++ 的标准库，如果我的程序使用一些额外的第三方库，怎么办？
交叉编译第三方库，并安装到工具链的sysroot 目录，比如我的程序使用了sqlite 库

下载sqlite 源码，配置，编译，安装到工具链的sysroot目录下:
```bash
wget  https://www.sqlite.org/2021/sqlite-autoconf-3350500.tar.gz
tar -zxvf sqlite-autoconf-3350500.tar.gz
cd sqlite-autoconf-3350500/
CC=aarch64-rpi4-linux-gnu-gcc    ./configure --host=aarch64-rpi4-linux-gnu   --prefix=/usr
make
make  DESTDIR=$(aarch64-rpi4-linux-gnu-gcc    -print-sysroot)   install
```
安装后在toolchain 的sysroot 下，可以看到sqlite相关的文件，比如:
```bash
ls    -l     `aarch64-rpi4-linux-gnu-gcc  -print-sysroot`/usr/lib  
```
output:
```bash
-rw-r--r-- 1 jianyu jianyu 10985808 Jun 13 10:37 libsqlite3.a
-rwxr-xr-x 1 jianyu jianyu      965 Jun 13 10:37 libsqlite3.la
lrwxrwxrwx 1 jianyu jianyu       19 Jun 13 10:37 libsqlite3.so -> libsqlite3.so.0.8.6
lrwxrwxrwx 1 jianyu jianyu       19 Jun 13 10:37 libsqlite3.so.0 -> libsqlite3.so.0.8.6
-rwxr-xr-x 1 jianyu jianyu  7312032 Jun 13 10:37 libsqlite3.so.0.8.6
```
在PKG_CONFIG_LIBDIR环境变量中，加入sqlite 包的sqlite.pc 文件所在的路径, 方便使用pkg-config  搜索开发库的头文件和库的路径:
```bash
export  PKG_CONFIG_LIBDIR=$(aarch64-rpi4-linux-gnu-gcc -print-sysroot)/usr/lib/pkgconfig
```

简单的sqlite 测试代码(sqlite-test.c)，使用sql 语句查询sqlite 的版本信息:
```bash
#include <sqlite3.h>
#include <stdio.h>

int main(void) {
    sqlite3 *db;
    sqlite3_stmt *res;
    
    int rc = sqlite3_open(":memory:", &db);
    
    if (rc != SQLITE_OK) {
        fprintf(stderr, "Cannot open database: %s\n", sqlite3_errmsg(db));
        sqlite3_close(db);
        return 1;
    }
    
    rc = sqlite3_prepare_v2(db, "SELECT SQLITE_VERSION()", -1, &res, 0);    
    
    if (rc != SQLITE_OK) {
         fprintf(stderr, "Failed to fetch data: %s\n", sqlite3_errmsg(db));
        sqlite3_close(db);
        return 1;
    }    
    
    rc = sqlite3_step(res);
    
    if (rc == SQLITE_ROW) {
        printf("%s\n", sqlite3_column_text(res, 0));
    }
    sqlite3_finalize(res);
    sqlite3_close(db);
    return 0;
}
```
编译sqlite-test.c:
```bash
aarch64-rpi4-linux-gnu-gcc  $(pkg-config sqlite3 --libs --cflags)  sqlite-test.c  -o sqlite-test
```
观察sqlite-test 二进制的动态库依赖:
```bash
aarch64-rpi4-linux-gnu-ldd  --root=$(aarch64-rpi4-linux-gnu-gcc -print-sysroot) ./sqlite-test
```
output:
```bash
libsqlite3.so.0 => /usr/lib/libsqlite3.so.0 (0x00000000deadbeef)
        libm.so.6 => /lib/libm.so.6 (0x00000000deadbeef)
        libc.so.6 => /lib/libc.so.6 (0x00000000deadbeef)
        ld-linux-aarch64.so.1 => /lib/ld-linux-aarch64.so.1 (0x00000000deadbeef)
        libdl.so.2 => /lib/libdl.so.2 (0x00000000deadbeef)
        libpthread.so.0 => /lib/libpthread.so.0 (0x00000000deadbeef)
```
也可以使用readelf 工具观察:
```bash
aarch64-rpi4-linux-gnu-readelf -a  ./sqlite-test|egrep  'interpreter|library'
```
output:
```bash
 [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libsqlite3.so.0]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
```
copy arm64 二进制sqlite-test到目标机器rpi4上运行:
```bash
scp ./sqlite-test rpi4w:~/
./sqlite-test 
```
output:
```bash
3.34.1
```
需要目标机器上安装sqlite 的动态库文件， 我的rpi4 使用了ubuntu server,  已经安装sqlite库:
```bash
dpkg -L libsqlite3-0
```
output:
```bash
/.
/usr
/usr/lib
/usr/lib/aarch64-linux-gnu
/usr/lib/aarch64-linux-gnu/libsqlite3.so.0.8.6
/usr/share
/usr/share/doc
/usr/share/doc/libsqlite3-0
/usr/share/doc/libsqlite3-0/README.Debian
/usr/share/doc/libsqlite3-0/changelog.Debian.gz
/usr/share/doc/libsqlite3-0/copyright
/usr/lib/aarch64-linux-gnu/libsqlite3.so.0
```
刚才在开发机器上编译安装的sqlite 版本是 3.35.5， 从测试代码的输出可以看到，目标机器上的sqlite版本是3.34.1，这是靠库的soname   libsqlite3.so.0， 以及最终库文件libsqlite3.so.0.8.6 达成API 兼容。
