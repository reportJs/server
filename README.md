# server

## Record (nginx settings)
    当大家都在使用，高效的nginx来做各种web代理和服务时，大家认为它是很高效的，所以我们本节借助nginx，快速搭建一个高效的
日志纪录服务，当然我们也可以使用Java，Golang等编译语言，不建议使用PHP等。另一个原因是我们依然有一系列nginx常规日志需要
分析。

### reportJs.js 引用
    http://logs.i-vectors.com:8099/reportJs.min.js

### 1.Configure the log format
    首先我们在 nginx.conf 的http配置下增加自定义日志格式。
```json   
http {
log_format web_log_post escape=json '{"@ts":"$time_local",'
                         '"@addr":"$remote_addr",'
                         '"@method":"$request_method",'
                         '"@ua":"$http_user_agent",'
                         '"@data":"$request_body"}';

log_format web_log_get  escape=json  '{"@ts":"$time_local",'
                         '"@addr":"$remote_addr",'
                         '"@method":"$request_method",'
                         '"@ua":"$http_user_agent",'
                         '"@data":"$arg_data"}';
```

###  2.Report interface conf.d/you_report_web.conf
 我启动一个http服务用来收集上报的日志信息，具体模板如下
```json
server {
    listen 80;

    # 域名
    server_name www.service.com;

    add_header Access-Control-Allow-Headers X-Requested-With;
    add_header Access-Control-Allow-Methods GET,POST,OPTIONS;

    location /reportJs.js {
        alias /usr/share/nginx/html/reportJs.min.js;
    }
    location /reportJs.min.js {
        alias /usr/share/nginx/html/reportJs.min.js;
    }

    location / {
        default_type application/json;
        # 信任的域名
        if ($http_origin ~* (http?://.*\.mywebsiet\.com$)) {
                add_header Access-Control-Allow-Origin $http_origin;
        }
        # get 参数读取
        if ($request_method != POST) {
                #default_type application/json;
                access_log /var/log/nginx/children_web_error_get.log web_log_get;
                return 200 '{"status":"success","result":"nginx json"}';
        }
        # post 参数处理
        access_log /var/log/nginx/children_web_error_post.log web_log_post;
        proxy_pass http://www.service.com/null/null;
    }
    # 空返回 写给post
    location /null/null {
        default_type application/json;
        return 200 '{"status":"success","result":"nginx json"}';
    }
}
```
### 3.日志格式
    我们得到的日志格式如下
#### get 
 ```json
 {
   "@ts":"12/Jan/2018:17:09:29 +0800",
  "@addr":"210.12.48.132",
  "@method":"GET",
  "@ua":"Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.132 Safari/537.36","@data":"encodeURIComponent(JSON.stringify(data))"
 }
 ```
#### post 
 ```json
 {
  "@ts":"12/Jan/2018:17:09:29 +0800",
  "@addr":"210.12.48.132",
  "@method":"POST",
  "@ua":"Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.132 Safari/537.36","@data":"JSON.stringify(data)"
 }
 ```

 4.Filebeat 上报
    Filebeat是一个日志文件托运工具，在你的服务器上安装客户端后，filebeat会监控日志目录或者指定的日志文件，追踪读取这些文件（追踪文件的变化，不停的读），并且转发这些信息到elasticsearch或者logstarsh中存放。

cd /etc/pki/tls
openssl req -x509 -batch -nodes -subj "/CN=*.i-vectors.com/" -days 3650 -newkey rsa:2048 -keyout private/logstash-beats.key -out certs/logstash-beats.crt