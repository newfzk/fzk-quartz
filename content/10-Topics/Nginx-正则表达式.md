---
title: Nginx 正则表达式 — 语法与常用模式
type: basic-note
date: 2026-07-10
tags:
  - topic/Nginx
  - topic/计算机网络
status: to-review
---

# Nginx 正则表达式 — 语法与常用模式

Nginx 多个指令支持正则匹配，其正则语法基于 **PCRE（Perl Compatible Regular Expressions）** 库，与大多数编程语言的正则语法一致。

## 核心语法速查

| 元字符 | 含义 | 示例 |
|:---:|:---|:---|
| `^` | 匹配字符串开头 | `^/api` 匹配以 `/api` 开头的字符串 |
| `$` | 匹配字符串结尾 | `\.php$` 匹配以 `.php` 结尾的字符串 |
| `.` | 匹配任意单个字符（除换行符） | `.+` 匹配任意字符 |
| `?` | 前一个字符出现 0 次或 1 次 | `https?` 匹配 `http` 或 `https` |
| `*` | 前一个字符出现 0 次或多次 | `[^/]+` 匹配非 `/` 字符一次或多次 |
| `+` | 前一个字符出现 1 次或多次 | `.+` 匹配至少一个字符 |
| `\` | 转义字符 | `\.` 匹配字面量 `.`，`\$` 匹配字面量 `$` |
| `( )` | 捕获分组，用于提取匹配内容 | `(/.*)` 捕获路径部分 |
| `$1`–`$9` | 引用分组捕获的内容 | `$1` 引用第一个 `()` 捕获的内容 |
| `[^ ]` | 否定字符集 | `[^/]` 匹配任何不是 `/` 的字符 |
| `\|` | 逻辑或 | `http|https` 匹配 `http` 或 `https` |

> `~`开头 表示后面的是正则匹配
## 实战拆解：proxy_redirect 中的正则

这是 [[Nginx-proxy-redirect]] 中的通用替换规则：

```nginx
proxy_redirect ~^https?://[^/]+(/.*)$ /rs$1;
```

将这条规则**逐段拆解**：

```
~               标记：这是一个正则匹配（而非前缀匹配）
^               字符串开始
http            匹配字面量 "http"
s?              匹配 s 零次或一次 → 同时匹配 http 和 https
:               匹配字面量 ":"
//              匹配字面量 "//"
[^/]+           匹配一个或多个非 "/" 字符 → 匹配域名+端口部分（如 "192.168.1.1:30331"）
(               捕获分组开始
  /             匹配字面量 "/"
  .*            匹配任意字符零次或多次 → 匹配剩余路径
)               捕获分组结束
$               字符串结束
/rs$1           替换为 "/rs" + 第一个捕获分组的内容
```

> **图解匹配过程**：
> ```
> 后端 Location: http://192.168.1.1/apps/sys/sys0020
>                ^^^^^^ ^^ ^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^
>                http   s? ://  [^/]+     (/.*)  → 捕获 $1 = "/apps/sys/sys0020"
> 
> 替换结果：/rs/apps/sys/sys0020
> ```

### 示例对照

| 后端返回的 Location | 是否匹配 | 替换为 |
|:---|:---:|:---|
| `http://192.168.1.1/apps/sys` | ✅ | `/rs/apps/sys` |
| `https://192.168.1.1:30331/abc` | ✅ | `/rs/abc` |
| `http://nginx-svc/apps` | ✅ | `/rs/apps` |
| `https://example.com/api/v1/users` | ✅ | `/rs/api/v1/users` |
| `http://`（无域名） | ❌ | — |
| `/relative/path`（相对路径） | ❌ | — |

## Nginx 中支持正则的指令

Nginx 中 `~` 前缀表示**启用正则匹配**，`~*` 表示**启用正则匹配并忽略大小写**。

### 1. `proxy_redirect` — 重写后端 Location 头

```nginx
# ~ 后跟正则，替换地址中可使用 $1-$9 引用捕获分组
proxy_redirect ~^https?://[^/]+(/.*)$ /rs$1;
```

详见 [[Nginx-proxy-redirect]]。

### 2. `location` — 按正则路由请求

```nginx
# 正则匹配路径（区分大小写）
location ~ \.php$ {
    fastcgi_pass unix:/var/run/php-fpm.sock;
}

# 正则匹配路径（忽略大小写）
location ~* \.(jpg|png|gif)$ {
    expires 30d;
}
```

详见 [[Nginx-location块]]。

### 3. `server_name` — 按正则匹配域名

```nginx
# 匹配所有以 .example.com 结尾的域名
server_name ~^(?<subdomain>.+)\.example\.com$;
```

### 4. `rewrite` — URL 重写（最常使用正则）

```nginx
# 将 /old-path/xxx 重写为 /new-path/xxx
rewrite ^/old-path/(.*)$ /new-path/$1 permanent;

# 将 http 重定向到 https
rewrite ^https?://(.*)$ https://$1 permanent;
```

### 5. `map` 指令的值匹配

```nginx
map $http_host $backend {
    ~^api\.      api-backend;
    ~^www\.      web-backend;
    default      fallback-backend;
}
```

### 6. `if` 条件判断

```nginx
if ($request_uri ~* "^/admin") {
    return 403;
}
```

## 注意事项

1. **`.` 需要转义**：匹配字面量 `.` 必须写成 `\.`（如 `\.php$`），否则 `.` 匹配任意字符
2. **分组引用范围**：Nginx 支持 `$1` 到 `$9` 共 9 个捕获分组
3. **`~` 与 `~*` 的区别**：`~` 区分大小写，`~*` 忽略大小写
4. **`~` 前的空格不能省略**：`proxy_redirect ~^...`（空格），而不是 `proxy_redirect~^...`
5. **贪婪匹配**：`.*` 默认贪婪（尽可能多匹配），可使用 `.*?` 转为非贪婪（但 Nginx 中 `/.*` 通常已经足够）
6. **不能嵌套分组命名**：部分 Nginx 模块支持 `(?<name>...)` 命名分组（如 `server_name`），但 `proxy_redirect` 只支持数字引用 `$1-$9`

## 相关笔记

- [[Nginx-proxy-redirect]]
- [[Nginx-location块]]
- [[Nginx-proxy-pass-路径处理]]
