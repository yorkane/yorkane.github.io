# [Centos7 Openresty Installation](https://github.com/yorkane/yorkane.github.io/issues/10)

# Openresty Installation and Optimization

## Installation 
```sh
# Install dependency

 yum -y install wget lrzsz mlocate libtool gcc gcc-c++ pcre-devel openssl-devel curl git make zlib-devel

# curl -Lk https://openresty.org/download/openresty-1.17.8.1rc1.tar.gz | tar xvz 
curl -Lk https://openresty.org/download/openresty-1.17.8.1rc1.tar.gz | tar xvz
 
 # Compile and install
cd openresty-*
git clone https://github.com/vozlt/nginx-module-vts.git
#git clone https://github.com/openresty/echo-nginx-module.git
git clone https://github.com/yorkane/nginx-http-concat.git
#git clone https://github.com/nginxinc/ngx-stream-nginmesh-dest.git

make clean
rm -f /usr/local/openresty/bin/nginx
./configure --with-pcre-jit --with-http_stub_status_module \
--with-http_ssl_module \
--with-http_realip_module \
--with-http_v2_module \
--with-stream_ssl_preread_module --with-stream --with-stream_ssl_module \
--with-http_gzip_static_module \
--add-dynamic-module=./nginx-module-vts \
--add-dynamic-module=./nginx-http-concat
#--with-http_sub_module \
#--add-dynamic-module=./ngx-stream-nginmesh-dest/module \
#--add-dynamic-module=./echo-nginx-module \

gmake & gmake install
# make resty global executable
cd /usr/local/openresty/nginx/
ln -s /usr/local/openresty/bin/resty /usr/local/sbin/resty
ln -s /usr/local/openresty/luajit/include/luajit-2.1/ /usr/include/lua5.1
ln -s /usr/local/openresty/luajit/lib/libluajit-5.1.so /lib/liblua-5.1.so


# Clean redundent files
rm -f /usr/local/openresty/nginx/conf/*.default
rm -f /usr/local/openresty/nginx/conf/koi-*
rm -f /usr/local/openresty/nginx/conf/win-*
rm -f /usr/local/openresty/nginx/conf/fastcgi*
rm -f /usr/local/openresty/nginx/conf/uwsgi*
rm -f /usr/local/openresty/nginx/conf/scgi*
rm -rf /usr/local/openresty/nginx/scgi*
rm -rf /usr/local/openresty/nginx/uwsgi*
rm -rf /usr/local/openresty/nginx/fastcgi*
rm -rf /usr/local/openresty/nginx/client_body*
rm -rf /usr/local/openresty/nginx/temp*
#mv -f /usr/local/openresty/nginx/modules/*.so /usr/local/openresty/modules/
#rm -rf /usr/local/openresty/nginx/modules
rm -f /usr/local/openresty/modules/*.old


mv /usr/local/openresty/nginx/sbin/nginx /usr/local/openresty/bin/ && rm -rf /usr/local/openresty/nginx/sbin/
sed -i "s@/usr/local/openresty/nginx/sbin/nginx@/usr/local/openresty/bin/nginx@" /usr/local/openresty/bin/resty  # not work in macos
sed -i '/^$main_include_directives/d' /usr/local/openresty/bin/resty # resty conf
sed -i '/^$main_conf_lines/d' /usr/local/openresty/bin/resty # resty conf
sed -i 's/^$env_list/$main_conf_lines\n$main_include_directives\n\n$env_list/' /usr/local/openresty/bin/resty #move the main conf section a head

ln -sf /usr/local/openresty/bin/resty /usr/local/bin/resty
ln -sf /usr/local/openresty/bin/nginx /usr/local/bin/nginx
ln -sf /usr/local/openresty/bin/openresty /usr/local/bin/openresty


# add web group
groupadd web -g 6000
useradd ngx -u 5000 -g web
#useradd ngx -u 5000 -g 6000
chown -R ngx:web /usr/local/openresty/nginx/
chown -R ngx:web /data/nginx_cache/

# chmod 755 ./*.sh
# chmod -R 766 ./

```

# Luarocks installation
```sh
yum install -y unzip
wget https://luarocks.org/releases/luarocks-3.3.0.tar.gz
tar zxpf luarocks-3.3.0.tar.gz
cd luarocks-3.3.0
./configure && make && sudo make install
cd ..
rm -rf ./luarocks*

```


## C10K System optimization
### Please remember modify the open file limit in `nginx.conf`
* worker_rlimit_nofile 65535; # main
* worker_connections   65535; # events
```sh
ulimit -SHn 65535
sed -i '/^ulimit -SHn.*/d' /etc/profile  #delete
echo "ulimit -SHn 65535" >> /etc/profile #append

# Delete the max openfile configs init script
sed -i '/^fs.file-max.*/d' /etc/sysctl.conf
# Append file-max config
echo "fs.file-max = 6553560" >> /etc/sysctl.conf
sysctl -p # take effect

sed -i '/^\*.*soft.*nofile.*/d' /etc/security/limits.conf
sed -i '/^\*.*soft.*nproc.*/d' /etc/security/limits.conf
sed -i '/^\*.*hard.*nofile.*/d' /etc/security/limits.conf
sed -i '/^\*.*hard.*nproc.*/d' /etc/security/limits.conf
cat >> /etc/security/limits.conf <<eof
*     soft   nproc           65535
*     hard   nproc           65535
*     soft   nofile          65535
*     hard   nofile          65535
eof
```

## Register openresty service 
```sh
#Create service file
cat > /etc/systemd/system/openresty.service <<eof
[Unit]
Description=The nginx HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/openresty/nginx/logs/nginx.pid
ExecStartPre=/usr/local/openresty/bin/nginx -t
ExecStart=/usr/local/openresty/bin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/usr/local/openresty/bin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
eof

systemctl daemon-reload
systemctl enable openresty
systemctl start openresty
```

## Compile and Install dynamic moudles
```sh
cd ~/openresty-1.13.6.2
git clone https://github.com/vozlt/nginx-module-vts.git
git clone https://github.com/nginxinc/ngx-stream-nginmesh-dest.git
git clone https://github.com/slact/ngx_lua_ipc.git
git clone https://github.com/openresty/echo-nginx-module.git
git clone https://github.com/yorkane/nginx-http-concat.git

cd build/nginx-1.13.6/
./configure --add-dynamic-module=../../nginx-module-vts \
--add-dynamic-module=../../echo-nginx-module \
--add-dynamic-module=../../ngx-stream-nginmesh-dest/module \

make modules
```

## Create Ramdisk for Cache folder
```sh
# Delete the loaded ramfs
mkdir /data
mkdir /data/nginx_cache
mkdir /data/nginx_cache/proxy_temp
mkdir /data/nginx_cache/ramcache
mkdir /data/nginx_cache/stale_cache
umount /data/nginx_cache/ramcache
# Delete the ramfs init script
sed -i '/^mount -t ramfs -o size=2500M.*/d' /etc/fstab

# write script into init script
echo "mount -t ramfs -o size=2500M ramfs /data/nginx_cache/ramcache" >> /etc/fstab
# mount ramdisk 2.5G
mount -t ramfs -o size=2500M ramfs /data/nginx_cache/ramcache
chown -R ngx:web /data/nginx_cache/
# For some reason to use tmpfs instead of ramfs 
# mount -t tmpfs -o size=2500M tmpfs /data/nginx_cache/ramcache

# 
```

## Create Ramdisk for Cache folder
```sh
# Delete the loaded ramfs
umount /usr/local/openresty/nginx/nginx_cache/ramcache
# Delete the ramfs init script
sed -i '/^mount -t ramfs -o size=4500M.*/d' /etc/fstab

# write script into init script
echo "mount -t ramfs -o size=4500M ramfs /usr/local/openresty/nginx/nginx_cache/ramcache" >> /etc/fstab
# mount ramdisk 4.5G
mount -t ramfs -o size=4500M ramfs /usr/local/openresty/nginx/nginx_cache/ramcache

# For some reason to use tmpfs instead of ramfs 
# mount -t tmpfs -o size=4500M tmpfs /usr/local/openresty/nginx/nginx_cache/ramcache

# 
```

## Get stale server code, and init
```sh
git clone -b release/v.0.1 http://dita@stash.wtvdev.com/scm/luap/stales.git

cd stales
mv .git /usr/local/openresty/nginx/
cd /usr/local/openresty/nginx/
sh local_init.cmd
```

## Configuration Schema
```sh
#根目录
├── approot  #程序目录
│   ├── app_cache.lua
│   ├── app.lua
│   ├── tool.lua
│   └── utils.lua
├── conf # 配置主目录
│   ├── cert # ssl及 密钥钥公保存
│   │   └── ssl.conf
│   ├── inc # 包含或者需要定制 配置
│   │   ├── http.conf
│   │   ├── location_empty_favicon.conf
│   │   ├── location_error_page.conf
│   │   ├── location_letsencrypt.conf
│   │   ├── log.set  # 修改日志格式及保存路径 不会被覆盖
│   │   ├── lua.set 
│   │   ├── mime.types
│   │   ├── resolver.set  # 修改DNS解析服务器及缓存时间
│   │   ├── stale_cache.set # stale缓存策略，不可更改
│   │   ├── stale_ramcache.set # 内存缓存策略，不可更改
│   │   ├── worker.set # worker 数量配置
│   │   └── worker.set.default
│   ├── nginx.conf # 主nginx 配置，不可修改
│   └── server
│       ├── 80.conf # 主端口配置，可以为空
│       ├── upstreams.conf # 后端upstream 配置，不能为空
│       ├── mock.conf # 测试模拟服务器，不可修改
│       ├── ramcache.conf # 内存缓存服务器 可以复制为80.conf
│       ├── servers.conf #额外配置的服务器 可以为空
│       └── stale.conf # 磁盘缓存服务器 不可修改

```

## Macos installation
```sh
curl -Lk https://openresty.org/download/openresty-1.15.8.1rc0.tar.gz | tar xvz
cd openresty-1.15.8.1rc0

git clone https://github.com/vozlt/nginx-module-vts.git
git clone https://github.com/nginxinc/ngx-stream-nginmesh-dest.git
git clone https://github.com/slact/ngx_lua_ipc.git
git clone https://github.com/openresty/echo-nginx-module.git


./configure --with-cc-opt="-I/usr/local/opt/openssl/include/ -I/usr/local/opt/pcre/include/" --with-ld-opt="-L/usr/local/opt/openssl/lib/ -L/usr/local/opt/pcre/lib/" -j8 --with-pcre-jit --with-http_stub_status_module \
--with-http_ssl_module \
--with-http_realip_module \
--with-http_sub_module \
--with-stream_ssl_preread_module --with-stream --with-stream_ssl_module \
--with-http_gzip_static_module \
--add-dynamic-module=./nginx-module-vts \
--add-dynamic-module=./echo-nginx-module \
--add-dynamic-module=./ngx-stream-nginmesh-dest/module

rm /usr/local/bin/resty
ln -s /usr/local/openresty/bin/resty /usr/local/bin/

# add group read and write permission 
# MacOS user manual add groud at system preference
dscl . -create /Groups/web # create user
dscl . -create /Users/web RealName "ForWebServer" #create group
dscl . -append /Groups/web GroupMembership web #add user to group

```
> 出现以上问题的原因是，在安装nginx的时候没有指定openssl的解压路径。正确的做法如下：
./configure --prefix=/usr/local/nginx  --with-openssl=/usr/local/openssl-1.0.1j --with-http_ssl_module 

> 如果pcre和zlib出现类似的问题，指定路径就可。
--with-pcre=/usr/local/pcre-7.7 --with-zlib=/usr/local/zlib-1.2.3 --with-http_stub_status_module


## Extract Openresty Binary pack
```sh
mv /usr/local/openresty/nginx/sbin/nginx /usr/local/openresty/bin/ && rm -rf /usr/local/openresty/nginx/sbin/
sed -i "s@/usr/local/openresty/nginx/sbin/nginx@/usr/local/openresty/bin/nginx@" /usr/local/openresty/bin/resty  # not work in macos
sed -i '/^$main_include_directives/d' /usr/local/openresty/bin/resty # resty conf
sed -i '/^$main_conf_lines/d' /usr/local/openresty/bin/resty # resty conf
sed -i 's/^$env_list/$main_conf_lines\n$main_include_directives\n\n$env_list/' /usr/local/openresty/bin/resty #move the main conf section a head

ln -sf /usr/local/openresty/bin/resty /usr/local/bin/resty
ln -sf /usr/local/openresty/bin/nginx /usr/local/bin/nginx
rm -rf pod site/pod site/manifest COPYRIGHT /usr/local/openresty/bin/openresty
tar -cf - bin/ luajit/ lualib/ modules/ site/ resty.index | xz -9v > or.tar.xz
mv or.tar.xz nginx/html/ -f
```
## install full pack
```sh
curl -L http://stash.wtvdev.com/projects/LUAP/repos/openresty_bin/browse/stale_el7.tar.xz | tar xvJ

```

# Extra libs in public server
```
yum -y install wget lrzsz xxhash-libs xxhash-devel

sudo yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm

# Install the yum-utils package (for the yum-config-manager command)
#yum -y install yum-utils
# Command to enable the repository
#yum-config-manager --enable remi##
sudo yum install -y vips

yum install -y https://rpms.remirepo.net/enterprise/7/remi/x86_64/libwebp7-1.0.3-1.el7.remi.x86_64.rpm

yum install -y https://rpms.remirepo.net/enterprise/7/remi/x86_64/ImageMagick-libs-6.9.11.7-1.el7.remi.x86_64.rpm

yum install -y https://rpms.remirepo.net/enterprise/7/remi/x86_64/vips-8.9.2-1.el7.remi.x86_64.rpm



```

# Extra libs in demostic server
> Probaly need VPN to speedup download
```
yum -y install wget lrzsz xxhash-libs
yum -y install https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/l/luarocks-2.3.0-1.el7.x86_64.rpm

```


```
export http_proxy=http://172.18.0.1:1008
export https_proxy=http://172.18.0.1:1008

yum install -y wget lrzsz libwebp

yum install -y https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/l/libraqm-0.7.0-4.el7.x86_64.rpm
yum install -y https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/m/matio-1.5.3-1.el7.x86_64.rpm


yum install -y https://rpms.remirepo.net/enterprise/7/remi/x86_64/libwebp7-1.0.3-1.el7.remi.x86_64.rpm
yum install -y https://rpms.remirepo.net/enterprise/7/remi/x86_64/ImageMagick-libs-6.9.11.7-1.el7.remi.x86_64.rpm
yum install https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/l/libraqm-0.7.0-4.el7.x86_64.rpm
yum install -y https://rpms.remirepo.net/enterprise/7/remi/x86_64/vips-8.9.2-1.el7.remi.x86_64.rpm


http://mirror.centos.org/centos/7/os/x86_64/Packages/libgfortran-4.8.5-39.el7.x86_64.rpm

yum install libgfortran


wget https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/l/libraqm-0.7.0-4.el7.x86_64.rpm
wget https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/m/matio-1.5.3-1.el7.x86_64.rpm
wget https://rpms.remirepo.net/enterprise/7/remi/x86_64/libwebp7-1.0.3-1.el7.remi.x86_64.rpm
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/libgfortran-4.8.5-39.el7.x86_64.rpm
wget https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/l/libaec-1.0.4-1.el7.x86_64.rpm
wget https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/h/hdf5-1.8.12-11.el7.x86_64.rpm
wget https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/o/openslide-3.4.1-1.el7.x86_64.rpm
wget https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/c/cfitsio-3.370-10.el7.x86_64.rpm
wget https://rpms.remirepo.net/enterprise/7/remi/x86_64/ImageMagick-libs-6.9.11.7-1.el7.remi.x86_64.rpm
wget https://rpms.remirepo.net/enterprise/7/remi/x86_64/vips-8.9.2-1.el7.remi.x86_64.rpm

```

# Luarocks
```
yum install -y unzip lua lua-devel git gcc gcc-c++ make
curl -Lk https://luarocks.org/releases/luarocks-3.3.1.tar.gz|tar xzv
cd luarocks*
./configure && make && make install
cd ..
rm -rf luarocks* 

```