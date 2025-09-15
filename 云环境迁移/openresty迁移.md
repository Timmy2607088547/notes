# openresty迁移

重命名

> hostnamectl set-hostname

安装openresty  

>  wget https://openresty.org/download/openresty-1.21.4.1.tar.gz

安装相关依赖：

> yum install -y pcre-devel openssl-devel zlib-devel gcc make perl libxml2-devel libxslt-devel gd-devel ibpng-devel libjpeg-devel freetype-devel GeoIP-devel
>
> yum install -y libxml2-devel libxslt-devel
>
> yum install -y gcc make pcre-devel openssl-devel perl zlib-devel git
>
> yum install -y pcre-devel openssl-devel zlib-devel gcc make perl
>
> yum install -y gd-devel
>
> yum install -y GeoIP-devel

编译

> ./configure --prefix=/usr/local/nginx  --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module  --with-http_addition_module --with-cc-opt=-O2 --with-mail_ssl_module  --with-pcre-jit --with-http_geoip_module --with-http_gzip_static_module  --with-http_image_filter_module --with-http_sub_module --with-http_xslt_module  --with-stream --with-stream_ssl_module --with-http_ssl_module --with-luajit
>
> make -j$(nproc)
>
> make install

配置免密

ssh-keygen -t rsa

新建目录

> mkdir -p /data/deploy/prod
>
> mkdir -p /data/logs/nginx
>
> sudo chmod -R 755 /data/deploy/

同步资源

> yum install rsync -y
>
> rsync -avzP -e "ssh -T -c aes128-ctr -o Compression=no"  /data/ root@172.20.28.31:/data/
>
> rsync -avzP --exclude='logs/' -e "ssh -T -c aes128-ctr -o Compression=no" /data/ root@172.20.28.42:/data/
>
> scp -r conf root@172.20.28.41:/usr/local/nginx/nginx/

磁盘挂载

> sudo mkdir /data
>
> ```
> sudo mkfs.ext4 /dev/vdb  
> sudo blkid /dev/vdb
> sudo vi /etc/fstab
> UUID=cde6042f-a742-4986-b7ba-450e10da8fc0   /data   ext4  defaults        0       0
> sudo mount -a  # 自动挂载fstab中所有未挂载的设备，用于测试配置是否正确
> systemctl daemon-reload
> df -h /data    # 检查是否成功挂载
> 
> sudo mkdir /var/lib/container/
> sudo mkfs.ext4 /dev/vdc
> sudo blkid /dev/vdc
> UUID=034a5977-4d76-4e79-9996-d89a0b32c908   /var/lib/container/   ext4  defaults        0       0
> sudo mount -a  # 自动挂载fstab中所有未挂载的设备，用于测试配置是否正确
> systemctl daemon-reload
> df -h   /var/lib/container/
> ```

172.16.1.26 online-openresty01

172.20.28.20

172.20.28.32





prod-openresty-mid-01  172.18.4.200

172.20.28.44