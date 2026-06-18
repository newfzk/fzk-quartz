---
title: log_format nginx日志格式定义
type: basic-note
date: 2025-06-25
tags:
  - topic/Nginx
  - topic/计算机网络
status: to-review
---

# log_format nginx日志格式定义

语法：`log_format name [escape=default|json] string;`

其中各部分具体说明如下：

- `name`: 日志格式名称 以此作为依据供`access_log`指令引用日志格式；
- `escape`: 变量中的字符编码方式，可以省略，默认是default
- `string`: 要定义的日志格式内容，该参数可以有多个，有多行（nginx自动拼接），可以使用nginx变量；

示例如下：

- 日志格式配置示例

```conf
# nginx 内置变量 官方文档
# https://nginx.org/en/docs/http/ngx_http_core_module.html#variables
# $http_请求头名 任意请求头字段 变量名最后部分是转换为小写的字段名 其中 破折号转换成下划线
log_format json_tarce_log escape=json '{'
            # rs10 traceId 对应 X_TRACE_ID http头字段
            '"X_TRACE_ID":"$http_x_trace_id",'
            '"remote_addr":"$remote_addr",'
            '"remote_user":"$remote_user",'
            '"time_local":"$time_local",'
            '"request":"$request",'
            '"status":"$status",'
            '"body_bytes_sent":"$body_bytes_sent",'
            '"http_referer":"$http_referer",'
            '"http_user_agent":"$http_user_agent",'
            '"http_x_forwarded_for":"$http_x_forwarded_for"'
            '}';
```

> **注意**: log_format 只能配置在http{}代码块中 **不能写在server/location或upstream等块中
>
> 如果希望在`/etc/nginx/conf.d/`中的配置文件里修改`http{}`块的内容，不能定义新的`http`块，而是应该向全局已有的`http`块中添加内容
>
> 因为nginx主配置文件`/etc/nfinx/nginx.conf`通常会包含一行`include /etc/nginx/conf.d/*.conf`
>
> 即**所有`/etc/nginx/conf.d`下的`.conf`文件都会被作为`http`块的一部分解析
>
> 即直接在`/etc/nginx/conf.d/`下的配置文件中直接定义即可，不要写在文件中的`server`（或其他）代码块中就行