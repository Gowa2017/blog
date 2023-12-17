---
title: 利用github API对markdown进行渲染
toc: true
date: 2016-12-22 09:15:18
tags: [Github, Markdown, Nginx, Php]
categories:  Misc
---
很喜欢github形式的markdown渲染，所以百度了一下，怎么样渲染成那样。结果还真有，github全占支持markdown，还提供了api接口。
<!--more-->
# 环境
**OS: CentOS 5.10 x86_64**
个人比较喜欢nginx，所以就用了nginx + php来建立环境，不过在安装php-fpm的时候可能要花点力气，主要是因为用的是 CentOS 5.10的系统，源有点老。
这里就不赘述安装过程了，主要说一下配置的时候所遇到的蛋疼的问题。
# 配置
## nginx配置
路径:/etc/nginx/conf.d/default.conf

```
server {
    listen       80 default_server;
    #listen       [::]:80 default_server;
    server_name  _;
    root         /usr/share/nginx/html;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / { 
        index index.html index.md README.md;
    }   
        location ~ \.php$ {
            root           /usr/share/nginx/html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }   
    location ~ \.md$ {
        rewrite ^/([^?]*)(?:\?(.*))? /md.php?f=$1&$2 last; 
    }   

    error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }

}
```
## php-fpm这里
这个的配置就是要把user/group都改成nginx，用户跟组一致才行。
## 建立md.php文件

```php
<?php
// 参数检查代码省略，然而这是必须的，否则你的 VPS 将会有后顾之忧

function curl_raw($url, $content) {
    $curl = curl_init($url);
    curl_setopt($curl, CURLOPT_HEADER, false);
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($curl, CURLOPT_HTTPHEADER,
        array("Content-type: application/json",
              "User-Agent: " . $_SERVER['HTTP_USER_AGENT']));
    curl_setopt($curl, CURLOPT_POST, true);
    curl_setopt($curl, CURLOPT_POSTFIELDS, $content);
    curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, false);

    $json_response = curl_exec($curl);

    $status = curl_getinfo($curl, CURLINFO_HTTP_CODE);

    curl_close($curl);

    return $json_response;
}

$markdown_filename = $_GET['f'];

$markdown_text = file_get_contents($markdown_filename);

$render_url = 'https://api.github.com/markdown';

$request_array['text'] = $markdown_text;
$request_array['mode'] = 'markdown';

$html_article_body = curl_raw($render_url, json_encode($request_array));

echo '<!DOCTYPE html><html><head><meta charset="utf-8"><title>' . $markdown_filename . '</title><link rel="stylesheet" href="/md_github.css" type="text/css" /></head>';
echo '<article class="markdown-body">';
echo $html_article_body;
echo '</article></body></html>';
?>
```
## 最后是css
[点此下载](https://github.com/sindresorhus/github-markdown-css/blob/gh-pages/github-markdown.css)
