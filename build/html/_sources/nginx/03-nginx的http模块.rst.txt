nginx的http模块
==========================================

配置块的嵌套情况
------------------------------------
.. code-block:: bash 

    http {
          upstream {}
          map {}
          geo {}
          server {}
          server{
            if () {}
            location {
            }
            location {
            }
          }

    }

指令的上下文
------------------------------------
指令的上下文是控制指令在哪个地方生效。这里按照官方网站的介绍一个。

.. image:: ../images/nginx11.png

这个例子说明log_format这个指令只能在http的片段内进行配置。

指令的合并
------------------------------------

- 值指令： 可以合并，子类可以从父类获取值的。可以向上覆盖的。
- 动作类指令： 不可以合并。

.. image:: ../images/nginx12.png

listen 指令
------------------------------------

参考： https://nginx.org/en/docs/http/ngx_http_core_module.html

这里提供几种listen方式

.. code-block:: text 

    listen unix:/var/run/nginx.sock;
    listen 127.0.0.1:8000;
    listen 127.0.0.1;
    listen 8000;
    listen \*:8000;
    listen [::]:8000 ipv6only=on;
    listen [::1];

接收请求事件模块
------------------------------------

操作系统会完成三次握手的， nginx通过epoll_wait读取到握手完毕包， nginx开始分配连接池 connection_poll_size:512 , nginx的http模块会给这个请求读取设置一个超时时间
client_header_time:60s, 然后分配一个缓冲区来读取header信息， client_header_buffer_size=1k， 接下来nginx继续通过epoll_wait获取header信息，存储在这个buffer中。

.. image:: ../images/nginx13.png

接收请求事件模块详细
------------------------------------

接收请求是2个大的部分， 一个是请求头部分，一个是请求body部分。 这里先说下第一个部分。

http模块处理请求，会先分配内存池的request_poll_size:4k，会解析行的， 如果第一行数据比较大， 原来的client_header_buffer_size 是放不下的，需要分配大内存。
large_clent_header_buffers: 4 8k 的大小。 继续解析行。 解析好后，uri变量就有值了， 接下来继续获取header信息 。 这个和uri使用共同的buffer。 

.. image:: ../images/nginx14.png 

上面提到的几个参数重点这里说明下

- client_header_buffer_size: 默认是1k的， 也就是默认1k,如果uri和header信息这个1k不够存储的话才会使用large_client_header_buffers指定的大小。
- large_client_header_buffers: 默认4 8k， 如果8k能够存储下，那就只是用1个8k， 不够的话，2个，再不够就3个，但是最多使用4个8k的大小。如果超过4*8k大小那就返回414 url too large ,or 400.
- client_header_timeout: 默认60s，如果60s内没有读取完毕，就响应408.

正则表达式
------------------------------------ 
nginx的正则是标准的posix正则， 这里简单说下。

.. image:: ../images/nginx15.png

nginx的正则也是支持捕获组的，也支持捕获组命名的。 样例如下

.. code-block:: bash 

  rewrite^/admin/website/solution/(\d+)/change/uploads/(.*)\.(?<ext>png|jpg|gif|jpeg|bmp)$ /static/uploads/$2/$3.$ext last;


server_name 指令
------------------------------------
server_name 后面可以指定多个域名的， 第一个是主域名。
相关参数 server_name_in_redirect on表示在重定向的时候使用主域名而非用户访问过来的域名进行重定向。

补充说明: 

server_name 的指定的特殊情况。 "" 表示匹配没有传递host头部的请求， _表示匹配所有。 

.. literalinclude:: ../files/server_name.conf
   :encoding: utf-8
   :language: text 

测试下如下

.. code-block:: bash 

  # server_name_in_redirect on 的情况下  可以发现重定向的结果都是变成了主域名的host了。

  curl http://n-n1.linuxpanda.tech:8084/ -IL
  HTTP/1.1 302 Moved Temporarily
  Server: openresty/1.19.9.1
  Date: Mon, 06 Dec 2021 12:28:56 GMT
  Content-Type: text/html
  Content-Length: 151
  Location: http://n-n1.linuxpanda.tech:8084/redirect
  Connection: keep-alive

  curl http://n-n2.linuxpanda.tech:8084/ -IL
  HTTP/1.1 302 Moved Temporarily
  Server: openresty/1.19.9.1
  Date: Mon, 06 Dec 2021 12:29:44 GMT
  Content-Type: text/html
  Content-Length: 151
  Location: http://n-n1.linuxpanda.tech:8084/redirect
  Connection: keep-alive

使用正则创建变量
------------------------------------

.. literalinclude:: ../files/server_name_regex_var.conf
   :encoding: utf-8
   :language: text 

验证结果如下

.. code-block:: text 

  [root@zhaojiedi-elk-2 conf]# curl http://n-n2.linuxpanda.tech:8084/
  n-n2
  [root@zhaojiedi-elk-2 conf]# curl http://n-n1.linuxpanda.tech:8084/
  n-n1

server匹配顺序
------------------------------------

#. 精确匹配的
#. \*在前的泛域名 
#. \*在后的泛域名
#. 正则匹配的
#. default server 

其中default server 是在listen指定default的，那就是default server ， 如果没有。 那就是nginx加载的第一个server为default server 。 

nginx处理的11个阶段
------------------------------------

.. image:: ../images/nginx16.png 

.. image:: ../images/nginx17.png 

#. post_read 
#. server_rewrite 
#. find_config 
#. rewrite 
#. post_rewrite 
#. preaccess
#. access
#. post_access
#. precontent 
#. content 
#. log 

如何获取真实的用户ip地址。
------------------------------------

用户访问到我们的服务，可能经过多层代理后到达的， 一般情况下通过这几种方式获取用户真实ip地址。 

- http头部的X-Real-IP用于传递用户的真实ip地址。  
- http头部的X-Forwarded-For用于传递代理等中间ip地址。 一般最后就是用户ip地址。

帮助文档： https://nginx.org/en/docs/http/ngx_http_realip_module.html

real_ip_header 这个默认值是X-Real-IP的 

.. note::  这个模块默认是不会编译到nginx的， 需要启用才可以的。 --with-http_realip_module	方式启用。

.. literalinclude:: ../files/real_ip.conf
   :encoding: utf-8
   :language: text 

测试结果如下

.. code-block:: bash 

  # 可以看到，real_ip_recursive on 开启后， set_real_ip_from指定的地址不会识别为real ip 的， 找到最后一个才算
  [root@zhaojiedi-elk-2 nginx]# curl http://n-realip.linuxpanda.tech -H "X-Forwarded-For: 1.1.1.1 2.2.2.2 10.157.1.2 10.157.1.3"
  Client real ip: 2.2.2.2
  [root@zhaojiedi-elk-2 nginx]# curl http://n-realip.linuxpanda.tech -H "X-Forwarded-For: 1.1.1.1 2.2.2.2 3.3.3.3 10.157.1.2 10.157.1.3"
  Client real ip: 3.3.3.3
  [root@zhaojiedi-elk-2 nginx]# curl http://n-realip.linuxpanda.tech -H "X-Forwarded-For: 10.157.1.2 10.157.1.3"
  Client real ip: 10.157.1.2

  # 可以看到，real_ip_recursive off 关闭后，只是简单的取最后一个的。
  [root@zhaojiedi-elk-2 nginx]# curl http://n-realip.linuxpanda.tech -H "X-Forwarded-For: 1.1.1.1 2.2.2.2 3.3.3.3 10.157.1.2 10.157.1.3"
  Client real ip: 10.157.1.3



如果拿到用户的ip如何使用
------------------------------------
nginx中可以通过变量访问到用户的真实ip地址。 remote_addr就是用户的ip地址。 有了这个用户ip可以做些限流等分流操作。


return 指令
------------------------------------
这个指令结束后续处理，直接给用户响应一个code和一些内容。

官方参考： https://nginx.org/en/docs/http/ngx_http_rewrite_module.html

几种样例

.. code-block:: bash 
  
  return 404 "find nothing!" 
  return http://www.baidu.com 
  return 502 

验证return errorpage优先级
------------------------------------

.. literalinclude:: ../files/return.conf
   :encoding: utf-8
   :language: text 
   :emphasize-lines: 9


效果验证

.. code-block:: text 
  
  # return 405; 关闭
  [root@zhaojiedi-elk-2 sites]# curl http://n-return.linuxpanda.tech:8084/ -IL
  HTTP/1.1 404 Not Found
  Server: openresty/1.19.9.1
  Date: Tue, 07 Dec 2021 02:47:12 GMT
  Content-Type: application/octet-stream
  Content-Length: 14
  Connection: keep-alive
  
  # return 405; 启用
  [root@zhaojiedi-elk-2 sites]# curl http://n-return.linuxpanda.tech:8084/ -IL
  HTTP/1.1 405 Not Allowed
  Server: openresty/1.19.9.1
  Date: Tue, 07 Dec 2021 02:48:12 GMT
  Content-Type: text/html
  Content-Length: 163
  Connection: keep-alive


通过验证我们知道， 

- server种的return指令是高于location的， 和配置前后无关系的。
- return就是直接返回了， 不会在经过errpage这些处理的。 

error_page 
------------------------------------
error_page 用于给特定code的展示一个特定的错误页面。 url可以包含变量的。 适用于给用户友好提示。

官方参考： https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page

几种样例配置

.. code-block:: text 

  # 这种方式，会引发内部的重定向，请求对应的页面， 方法为get， 而不是请求的原始方法。
  error_page 404 /404.html; 
  error_page 500 502 503 504 /5xx.html ;

  # 使用 = 方式，可以改变响应码的。
  error_page 404 =200 /empty.gif;

  # 这个没有指定200 ，那就是跟进/404.php的返回码来定。
  error_page 404 = /404.php; 

  # 下面的这个部分就是将404请求，转发给后端backend来响应
  location / { 
      error_page 404 =@fallback; 
  }
  location @fallback {
    proxy_pass http://backend ; 
  }

  # 使用url的方式， 默认是响应码是302的， 当然可以指定其他的 301， 302 303 307 308  只能这几个。 第二个404=301 就是指定方式。
  error_page 403 http://www.linuxpanda.tech/forbidden.html;
  error_page 404=301 http://www.linuxpanda.tech/notfound.html; 


rewrite指令
------------------------------------
这个指令的功能是将指定的url替换为新的url。  支持正则表达式提取的。

当replacement以http:// 或者https://或者$schema开头， 则直接返回302重定向。

替换后的url跟进flag指定的方式进行处理。

- last 这个url接下来使用新的location进行匹配。 
- break 停止执行。
- redirect 返回302 。 
- permanent 返回301 。

先准备下环境
.. code-block:: bash 

  [root@zhaojiedi-elk-2 sites]# mkdir ../../html/{first,second,third}
  [root@zhaojiedi-elk-2 sites]# echo "1" >> ../../html/first/1.txt
  [root@zhaojiedi-elk-2 sites]# echo "2" >> ../../html/second/2.txt
  [root@zhaojiedi-elk-2 sites]# echo "3" >> ../../html/third/3.txt
  [root@zhaojiedi-elk-2 sites]# tree ../../html/
  ../../html/
  ├── 50x.html
  ├── first
  │   └── 1.txt
  ├── index.html
  ├── second
  │   └── 2.txt
  └── third
      └── 3.txt

对应配置如下

.. literalinclude:: ../files/rewrite.conf
   :encoding: utf-8
   :language: text 


验证结果

.. code-block:: bash 

  # 可以看到这个url直接命中了location片段， 就return了。
  [root@zhaojiedi-elk-2 sites]# curl http://n-rewrite.linuxpanda.tech:8084/third/3.txt
  third!

  # 访问这个url命中了/second的location， 然后被break了， 后面return没有机会执行了， 然后直接content了， html/3.txt进行相应了。 
  [root@zhaojiedi-elk-2 sites]# curl http://n-rewrite.linuxpanda.tech:8084/second/3.txt
  3
  # 使用last会继续进行location的， 然后走第二个，然后和上面的案例一样了。 
  [root@zhaojiedi-elk-2 sites]# curl http://n-rewrite.linuxpanda.tech:8084/first/3.txt
  3

  # 然后进行调整rewrite和return的顺序， 在进行测试

  [root@zhaojiedi-elk-2 sites]# curl http://n-rewrite.linuxpanda.tech:8084/first/3.txt
  first!

  # 测试第一个重定向。
  [root@zhaojiedi-elk-2 sites]# curl http://n-rewrite.linuxpanda.tech:8084/redirect1/1.txt -IL
  HTTP/1.1 301 Moved Permanently
  Server: openresty/1.19.9.1
  Date: Tue, 07 Dec 2021 04:45:17 GMT
  Content-Type: text/html
  Content-Length: 175
  Location: http://n-rewrite.linuxpanda.tech:8084/1.txt
  Connection: keep-alive

  HTTP/1.1 404 Not Found
  Server: openresty/1.19.9.1
  Date: Tue, 07 Dec 2021 04:45:17 GMT
  Content-Type: text/html
  Content-Length: 159
  Connection: keep-alive

  [root@zhaojiedi-elk-2 sites]# curl http://n-rewrite.linuxpanda.tech:8084/redirect2/2.txt -IL
  HTTP/1.1 302 Moved Temporarily
  Server: openresty/1.19.9.1
  Date: Tue, 07 Dec 2021 04:45:59 GMT
  Content-Type: text/html
  Content-Length: 151
  Location: http://n-rewrite.linuxpanda.tech:8084/2.txt
  Connection: keep-alive

  HTTP/1.1 404 Not Found
  Server: openresty/1.19.9.1
  Date: Tue, 07 Dec 2021 04:45:59 GMT
  Content-Type: text/html
  Content-Length: 159
  Connection: keep-alive

  # 测试重定向一个url的。默认是302临时重定向的。
  [root@zhaojiedi-elk-2 sites]# curl http://n-rewrite.linuxpanda.tech:8084/redirect3/3.txt -IL
  HTTP/1.1 302 Moved Temporarily
  Server: openresty/1.19.9.1
  Date: Tue, 07 Dec 2021 04:46:35 GMT
  Content-Type: text/html
  Content-Length: 151
  Connection: keep-alive
  Location: http://n-rewrite.linuxpanda.tech/3.txt

  curl: (7) Failed connect to n-rewrite.linuxpanda.tech:80; Connection refused

  # 重定向4 
  [root@zhaojiedi-elk-2 sites]# curl http://n-rewrite.linuxpanda.tech:8084/redirect4/4.txt -IL
  HTTP/1.1 301 Moved Permanently
  Server: openresty/1.19.9.1
  Date: Tue, 07 Dec 2021 04:47:40 GMT
  Content-Type: text/html
  Content-Length: 175
  Connection: keep-alive
  Location: http://n-rewrite.linuxpanda.tech/4.txt

  curl: (7) Failed connect to n-rewrite.linuxpanda.tech:80; Connection refused


可以看出： 

- rewrite 和return的指令优先级基本一致， 谁在前谁先生效。
- permanent 是301， last这是302的，默认也是302的重定向。
- break这个会跳过执行，然后执行content阶段的。
- last这个会继续回头进行匹配location的。  内部重定向的，

rewrite 行为记录error日志
------------------------------------
通过rewrite_log on即可启用的， 


if指令
------------------------------------
if指令使用在server或者location中的。 

.. code-block:: bash 

  # 匹配ie浏览器
  if ($http_user_agent ~ MSIE) {
      rewrite ^(.*)$ /msie/$1 break;
  }

  # 提取cooke部分作为变量
  if ($http_cookie ~* "id=([^;]+)(?:;|$)") {
      set $id $1;
  }

  # 判定请求方法
  if ($request_method = POST) {
      return 405;
  }

  # 判定变量，进行速度限制
  if ($slow) {
      limit_rate 10k;
  }

  # 无效refer 给返回403 
  if ($invalid_referer) {
      return 403;
  }


location指令
------------------------------------

官方参考： https://nginx.org/en/docs/http/ngx_http_core_module.html#location

.. image:: ../images/nginx18.png

- 精确匹配优先
- ^~禁用正则次之
- 最长正则
- 最长匹配前缀
  



