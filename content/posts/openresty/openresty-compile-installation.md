---
title: "自己动手编译安装OpenResty"
date: 2019-06-16T12:11:09+08:00
authors:
    - windvalley
categories:
    - OpenResty
tags:
    - install
    - openresty
draft: true
---

自己动手编译安装有时候是必须的, 因为根据具体项目的需要, 比如CDN项目, 官方的默认编译选项就缺少一些必要的模块, 比如`ngx_cache_purge`模块, 如果需要对`ipv6`做支持, 还需要nginx的`--with-ipv6`编译选项, 等等.

下面我们通过参考OpenResty官方的打包文件, 来进行编译安装.

## 编译安装pcre

> 参考: https://github.com/openresty/openresty-packaging/blob/master/rpm/SPECS/openresty-pcre.spec

```bash
cd /usr/local/src

wget -c http://ftp.pcre.org/pub/pcre/pcre-8.42.tar.bz2

tar xjf pcre-8.42.tar.bz2
cd pcre-8.42

./configure --prefix=/usr/local/openresty/pcre \
    --disable-cpp --enable-jit \
    --enable-utf --enable-unicode-properties

make -j24 V=1 > /dev/stderr

make install

# 删除用不到的文件和目录
rm -rf /usr/local/openresty/pcre/bin
rm -rf /usr/local/openresty/pcre/share
rm -f  /usr/local/openresty/pcre/lib/*.la
rm -f  /usr/local/openresty/pcre/lib/*pcrecpp*
rm -f  /usr/local/openresty/pcre/lib/*pcreposix*
rm -rf /usr/local/openresty/pcre/lib/pkgconfig
```

最后的安装文件如下:

```bash
tree /usr/local/openresty/pcre
├── include
│   ├── pcre.h
│   └── pcreposix.h
└── lib
    ├── libpcre.a
    ├── libpcre.so -> libpcre.so.1.2.10
    ├── libpcre.so.1 -> libpcre.so.1.2.10
    └── libpcre.so.1.2.10
```

## 编译安装zlib

> 参考: https://github.com/openresty/openresty-packaging/blob/master/rpm/SPECS/openresty-zlib.spec

```bash
cd /usr/local/src
wget -c http://www.zlib.net/zlib-1.2.11.tar.xz
tar xf zlib-1.2.11.tar.xz
cd zlib-1.2.11

./configure --prefix=/usr/local/openresty/zlib

make -j24 \
CFLAGS='-O3 -D_LARGEFILE64_SOURCE=1 -DHAVE_HIDDEN -g' \
SFLAGS='-O3 -fPIC -D_LARGEFILE64_SOURCE=1 -DHAVE_HIDDEN -g' > /dev/stderr

make install

# 删除用不到的文件和目录
rm -rf /usr/local/openresty/zlib/share/
rm -f /usr/local/openresty/zlib/lib/*.la
rm -rf /usr/local/openresty/zlib/lib/pkgconfig/
```

最后的安装文件如下:

```bash
tree /usr/local/openresty/zlib
├── include
│   ├── zconf.h
│   └── zlib.h
└── lib
    ├── libz.a
    ├── libz.so -> libz.so.1.2.11
    ├── libz.so.1 -> libz.so.1.2.11
    └── libz.so.1.2.11
```

## 编译安装openssl

> 参考: https://github.com/openresty/openresty-packaging/blob/master/rpm/SPECS/openresty-openssl.spec

```bash
cd /usr/local/src

wget -c https://www.openssl.org/source/openssl-1.1.0j.tar.gz
wget -c https://raw.githubusercontent.com/openresty/openresty/master/patches/openssl-1.1.0d-sess_set_get_cb_yield.patch --no-check-certificate
wget -c https://raw.githubusercontent.com/openresty/openresty/master/patches/openssl-1.1.0j-parallel_build_fix.patch --no-check-certificate

tar zxf openssl-1.1.0j.tar.gz
cd openssl-1.1.0j

# 打补丁
patch -p1 < ../openssl-1.1.0d-sess_set_get_cb_yield.patch
patch -p1 < ../openssl-1.1.0j-parallel_build_fix.patch

# 编译
./config \
    no-threads shared zlib -g \
    enable-ssl3 enable-ssl3-method \
    --prefix=/usr/local/openresty/openssl \
    --libdir=lib \
    -I/usr/local/openresty/zlib/include \
    -L/usr/local/openresty/zlib/lib \
    -Wl,-rpath,/usr/local/openresty/zlib/lib:/usr/local/openresty/openssl/lib

make -j24

make install_sw

# 删除用不到的文件和目录
rm -f /usr/local/openresty/openssl/bin/c_rehash
rm -rf /usr/local/openresty/openssl/lib/pkgconfig
```

最后的安装文件如下:

```bash
tree /usr/local/openresty/openssl
├── bin
│   └── openssl
├── include
│   └── openssl
│       ├── aes.h
│       ├── *
└── lib
    ├── engines-1.1
    │   ├── capi.so
    │   └── padlock.so
    ├── libcrypto.a
    ├── libcrypto.so -> libcrypto.so.1.1
    ├── libcrypto.so.1.1
    ├── libssl.a
    ├── libssl.so -> libssl.so.1.1
    └── libssl.so.1.1
```

## 编译安装openresty

> 参考: https://github.com/openresty/openresty-packaging/blob/master/rpm/SPECS/openresty.spec

```bash
cd /usr/local/src
# 如果下载出现ssl报错, 先yum update wget, 再下载
wget -c https://openresty.org/download/openresty-1.15.8.1.tar.gz
tar zxf openresty-1.15.8.1.tar.gz
cd openresty-1.15.8.1

# 编译
./configure \
    --prefix=/usr/local/openresty \
    --with-cc-opt="-DNGX_LUA_ABORT_AT_PANIC \
                -I/usrl/local/openresty/zlib/include \
                -I/usr/local/openresty/pcre/include \
                -I/usr/local/openresty/openssl/include" \
    --with-ld-opt="-L/usr/local/openresty/zlib/lib \
                -L/usr/local/openresty/pcre/lib \
                -L/usr/local/openresty/openssl/lib \
                -Wl,-rpath,/usr/local/openresty/zlib/lib:/usr/local/openresty/pcre/lib:/usr/local/openresty/openssl/lib" \
    --with-pcre-jit \ # 开启PCRE JIT（just-in-time）编译技术, 显著提升正则表达式的处理效率
    --without-http_rds_json_module \
    --without-http_rds_csv_module \
    --without-lua_rds_parser \
    --with-stream \
    --with-stream_ssl_module \
    --with-stream_ssl_preread_module \
    --with-http_v2_module \
    --without-mail_pop3_module \
    --without-mail_imap_module \
    --without-mail_smtp_module \
    --with-http_stub_status_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_auth_request_module \
    --with-http_secure_link_module \
    --with-http_random_index_module \
    --with-http_gzip_static_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-threads \
    --with-luajit-xcflags='-DLUAJIT_NUMMODE=2 -DLUAJIT_ENABLE_LUA52COMPAT' \
    -j24

make -j24

make install

# 删除用不到的文件和目录
rm -rf /usr/local/openresty/luajit/share/man
rm -rf /usr/local/openresty/luajit/lib/libluajit-5.1.a
```

最后安装的文件如下:

```bash
tree  -L 2 /usr/local/openresty
├── bin
│   ├── md2pod.pl
│   ├── nginx-xml2pod
│   ├── openresty -> /usr/local/openresty/nginx/sbin/nginx
│   ├── opm
│   ├── resty
│   ├── restydoc
│   └── restydoc-index
├── COPYRIGHT
├── luajit
│   ├── bin
│   ├── include
│   ├── lib
│   └── share
├── lualib
│   ├── cjson.so
│   ├── librestysignal.so
│   ├── ngx
│   ├── redis
│   ├── resty
│   └── tablepool.lua
├── nginx
│   ├── conf
│   ├── html
│   ├── logs
│   └── sbin
├── openssl  # 注意这个目录非本次安装的, 之前就存在的
│   ├── bin
│   ├── include
│   └── lib
├── pcre # 注意这个目录非本次安装的, 之前就存在的
│   ├── include
│   └── lib
├── pod
│   ├── *
├── resty.index
├── site
│   ├── lualib
│   ├── manifest
│   └── pod
└── zlib # 注意这个目录非本次安装的, 之前就存在的
    ├── include
    └── lib
```
