# CentOS安装Nginx并配置HTTPS

服务器环境: CentOS7

nginx版本: 1.16.0

## 安装Nginx

### 安装依赖
    
1. 安装`nginx`需要先将官网下载的源码进行编译,编译依赖`gcc`环境

```shell script
yum install gcc-c++
```
    
2.PCRE(Perl Compatible Regular Expressions) 是一个`Perl`库,包括`perl`兼容的正则表达式库.
`nginx`的`http`模块使用`pcre`来解析正则表达式,所以需要安装`pcre`库,`pcre-devel`是使用`pcre`开发的一个二次开发库,`nginx`也需要此库

```shell script
yum install -y pcre pcre-devel` 
```

3.`zlib`库提供了很多种压缩和解压缩的方式,nginx使用`zlib`对`http`包的内容进行`gzip`

```shell script
yum install -y zlib zlib-devel
```
    
4.`OpenSSL`是一个强大的安全套接字层密码库,囊括主要的密码算法、常用的密钥和证书封装管理功能及`SSL`协议,并提供丰富的应用程序供测试或其它目的使用.

`nginx`不仅支持`http`协议,还支持`https`(即在ssl协议上传输http)

```shell script
yum install -y openssl openssl-devel
```  
### 下载
    
[Nginx 官网](https://nginx.org/en/download.html)

```shell script
wget -c https://nginx.org/download/nginx-1.16.0.tar.gz
```  

<details>
    <summary>参数含义</summary>
    
- `-c` 断点续传

</details>

### 解压

```shell script
tar -zxvf nginx-1.16.0.tar.gz
```

<details>
    <summary>参数含义</summary>

- `-z` :是否同时具有gzip的属性，即是否需要用gzip压缩
        
- `-x` :从归档文件中解析文件
    
- `-v` :压缩过程中显示文件
    
- `-f` :使用文件名

</details>
    
### 配置
    
进入文件

```shell script
cd nginx-1.16.0
```

+ 配置文件信息
    
    - 普通配置不需要配置HTTPS
    
    ```shell script
    ./configure
    ```
  
    - 如果需要配置HTTPS则执行
  
    ```shell script
    ./configure --with-http_ssl_module
    ```

### 编译安装        
```shell script
make

make install
```

### 设置开机自动

```shell script
cd /lib/systemd/system

vim nginx.service
```

将以下文本复进文件中保存即刻

```text
[Unit]
Description=nginx
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
   
### 命令详情

|描述|命令|
|----|----|
|设置开机自启动|`systemctl enable nginx.service`|
|停止开机自启动|`systemctl disable nginx.service`|
|启动Nginx服务|`systemctl start nginx.service`|
|停止Nginx服务|`systemctl stop nginx.service`|
|重新启动服务|`systemctl restart nginx.service`|
|查看服务当前状态|`systemctl status nginx.service`|

### 参考资料

[centos安装部署Nginx,并配置https](https://www.jianshu.com/p/5a57d1bea859)

[Nginx+Center OS 7.2 开机启动设置](https://www.cnblogs.com/piscesLoveCc/p/5867900.html)

## 配置HTTPS

HTTPS呢使用[Let's Encrypt](https://letsencrypt.org/)的免费证书

借助[`acme.sh`](https://github.com/Neilpang/acme.sh)这个项目来帮助我们生产证书

### 生产HTTPS证书

安装 acme.sh

```shell script
curl https://get.acme.sh | sh
```
      
创建一个alias方便使用(可不用)
   
```shell script
alias acme.sh=~/.acme.sh/acme.sh`
```
  
配置DNS,这里我使用的是阿里云的,因为我的域名本身就是在阿里云

```shell script
export Ali_Key="xxxxxxxxxxx"

export Ali_Secret="xxxxxxxxxxxxxxxxxxx"
``` 

生成通配符证书

```shell script
acme.sh --issue --dns dns_ali -d 你的主域名 -d *.你的主域名
```

<details>
    <summary>具体例子</summary>
    
```shell script
acme.sh --issue --dns dns_ali -d haohaoli.cn -d *.haohaoli.cn
```
</details>

### 将证书配置到Nginx

移动证书

```shell script
# 创建证书存放的位置
mkdir /usr/local/nginx/ssl

# 移动证书
acme.sh --install-cert -d 你的主域名 \
--key-file       /usr/local/nginx/ssl/你的主域名.key  \
--fullchain-file /usr/local/nginx/ssl/fullchain.cer \
--reloadcmd     "systemctl reload nginx.service"
```

<details>
    <summary>具体例子</summary>
    
```shell script
acme.sh --install-cert -d haohaoli.cn \
--key-file       /usr/local/nginx/ssl/haohaoli.key  \
--fullchain-file /usr/local/nginx/ssl/fullchain.cer \
--reloadcmd     "systemctl reload nginx.service"
```
</details>

修改nginx配置文件

```shell script
vim /usr/local/nginx/conf/nginx.conf
```

找到server为443端口位置

```text
server {
    listen       443 ssl;
    server_name  www域名 你的主域名;

    ssl_certificate      /usr/local/nginx/ssl/fullchain.cer;
    ssl_certificate_key  /usr/local/nginx/ssl/你的主域名.key;

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    location / {
        root   html;
        index  index.html index.htm;
    }
}
```

<details>
    <summary>例如</summary>
    
```text
server {
    listen       443 ssl;
    server_name  www.haohaoli.cn haohaoli.cn;

    ssl_certificate      /usr/local/nginx/ssl/fullchain.cer;
    ssl_certificate_key  /usr/local/nginx/ssl/haohaoli.cn.key;

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    location / {
        root   html;
        index  index.html index.htm;
    }
}
```
</details>


配置http请求转发为https请求,在`server`为80的`server_name`下添加:

```text
return       301 https://$server_name$request_uri;
```

最后重启加载nginx配置文件

```shell script
systemctl reload nginx.service
``` 

### 参考资料

[快速签发 Let's Encrypt 证书指南](https://www.cnblogs.com/esofar/p/9291685.html)
   
[利用 acme.sh 获取网站证书并配置https访问](https://my.oschina.net/u/3042999/blog/1858891)
   
[acme.sh](https://github.com/Neilpang/acme.sh)
   
[acme.sh-DNS API](https://github.com/Neilpang/acme.sh/wiki/dnsapi)