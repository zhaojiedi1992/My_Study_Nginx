openresty及nginx源码
==========================================

第三方模块源码的快速阅读方法
------------------------------------

1. 分析模块提供的config文件
2. 分析nginx_module_t 模块
3. 分析nginx_commmand_t数组
4. 对于http模块，分析ngx_http_module_t
5. 分析模块生效方法。

config结构
------------------------------------

- ngx_addon_name: 定义模块名称
- HTTP_MODULES: 定义模块
- NGX_ADDON_SRCS: 哪些模块的源文件需要编译

.. literalinclude:: ../files/header_more.config
   :encoding: utf-8
   :language: bash


nginx启动和退户回调方法
------------------------------------

.. code-block:: c

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

nginx启动过程
------------------------------------

.. image:: ../images/nginx34.png


master进程循环
------------------------------------

.. image:: ../images/nginx35.png

work进程循环
------------------------------------

.. image:: ../images/nginx36.png

http模块初始化
------------------------------------

.. image:: ../images/nginx37.png

http模块的11个阶段
------------------------------------

.. code-block:: c

    typedef enum {
        NGX_HTTP_POST_READ_PHASE = 0,

        NGX_HTTP_SERVER_REWRITE_PHASE,

        NGX_HTTP_FIND_CONFIG_PHASE,
        NGX_HTTP_REWRITE_PHASE,
        NGX_HTTP_POST_REWRITE_PHASE,

        NGX_HTTP_PREACCESS_PHASE,

        NGX_HTTP_ACCESS_PHASE,
        NGX_HTTP_POST_ACCESS_PHASE,

        NGX_HTTP_PRECONTENT_PHASE,

        NGX_HTTP_CONTENT_PHASE,

        NGX_HTTP_LOG_PHASE
    } ngx_http_phases;

添加模块到11个阶段的常用方法
------------------------------------

1. 直接覆盖式，通过 clcf->handler = nginx_http_proxy_handler;
2. 通过链表。top->filter1(还有next指向后续filter2)


if指令连续出现
------------------------------------
演示案例

具体原因分析

if指令是rewrite阶段的执行的，会在if条件为真的时候，替换掉当前请求的配置，if是向上继承的，在rewrite阶段顺序执行时，
每次if为真都会替换当前请求的配置。

正确姿势

1. 熟练掌握rewrite的5个指令。
2. if是可以正确执行，不依赖外部的指令。
3. break会阻断后续rewrite的阶段指令的执行。


coredump 核心转储文件
------------------------------------
worker_rlimit_core限制core文件大小，working_directory 控制coredump放置的目录。

官方参考： http://nginx.org/en/docs/ngx_core_module.html#worker_rlimit_core

如何生成coredump文件

配置nginx配置如下

.. code-block:: bash

    worker_rlimit_core 100M;
    working_directory /tmp/nginx;

发送信号

.. code-block:: bash

    # 确认一个work进程id
    [root@zhaojiedi-elk-2 nginx]# ps -ef |grep nginx
    root     22358     1  0 20:35 ?        00:00:00 nginx: master process ./sbin/nginx
    root     22622 22358  0 20:37 ?        00:00:00 nginx: worker process
    root     22696 22358  0 20:37 ?        00:00:00 nginx: worker process
    root     22894 20221  0 20:39 pts/0    00:00:00 grep --color=auto nginx

    #确认下段异常错误为11
    [root@zhaojiedi-elk-2 nginx]# kill -l
    1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
    6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
    11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
    16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
    21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
    26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
    31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
    38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
    43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
    48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
    53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
    58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
    63) SIGRTMAX-1	64) SIGRTMAX
    # send signal    
    [root@zhaojiedi-elk-2 nginx]# kill -11 22622 22696
    [root@zhaojiedi-elk-2 nginx]# ll /tmp/nginx/
    total 0
    [root@zhaojiedi-elk-2 nginx]# ll /home/coresave/
    total 9016
    -rw------- 1 root root 5124096 Jan 17 20:39 core.nginx.22622.1642423167
    -rw------- 1 root root 5124096 Jan 17 20:39 core.nginx.22696.1642423167

这里机器单独配置了core位置了，你是nginx配置的为啥不能覆盖系统的呢。

.. code-block:: bash 

    [root@zhaojiedi-elk-2 nginx]# sysctl  -a |grep patt
    kernel.core_pattern = /home/coresave/core.%e.%p.%t



gdb的基本使用
------------------------------------

- bt: 显示函数堆栈调用情况
- f: 显示某1堆栈详细信息
- p: 打印变量值
- l: 显示附近代码
- x: 显示具体内存值
- i: 显示信息

gdb分析上面产生的core文件

.. literalinclude:: ../files/gdb1.txt
   :encoding: utf-8
   :language: text
   :emphasize-lines: 1,2,21,38,57


debug定位问题
------------------------------------
我们引入了第三方模块可能存在bug的，打开debug_points后则遇到问题后停止服务，方便定位问题。

- abort: 生成coredump后结束进程
- stop: 结束进程


控制debug级别的error.log日志输出
------------------------------------
debug_connection针对特定客户端打印debug级别的日志，其他的日志正常打印。 

.. note:: 需要nginx编译的时候--with-debug编译选项。

debug日志分析
------------------------------------

.. code-block:: text 

    建立连接
        ssl握手
    接受请求的头部
        解析行
        解析头部
    11阶段处理
        查找location
        反向代理构造上游的请求
        接收客户端请求包体
    构造响应头
    发送响应
        过滤模块

.. literalinclude:: ../files/error.log
   :encoding: utf-8
   :language: text

openresty简介
------------------------------------

官方地址： https://openresty.org/cn/

详细的需要看对应组件的帮助文档。 github有对应的功能描述。


主要组成

.. image:: ../images/nginx38.png

运行机制

.. image:: ../images/nginx39.png


openrestysdk分类
------------------------------------

- cosocket通信
- 共享内存的字典
- 定时器
- 基于携程的并发编程
- 修改请求
- 修改响应
- 自请求

openresty的使用要点
------------------------------------

- 不破坏事件驱动实现，不要阻塞nginx进度调度。
- 不破坏nginx低内存的消耗优点
- 保持lua代码高效

openrestry的nginx核心模块
------------------------------------

- ndk_http_module: 开发工具包
- ngx_http_lua_module:openrestry提供http服务lua编程能力的核心模块
- ngx_http_lua_upstream_mudule: http_lua的补充，提供upstream api。
- ngx_stream_lua_module: 提供四层服务lua编程能力的核心模块。

openrestry的nginx工具模块
------------------------------------

- ngx_http_headers_more_filter_module: rewrite阶段处理请求，修改请求响应的header头的。
- ngx_http_rds_json——filter_module: 过滤模块，将rds格式的转换为json格式。
  
openrestry的官方lua模块
------------------------------------

- lua_redis_parser: 将redis相应解析为lua数据结构。
- lua_rds_parser: 将mysql postgress数据库响应解析为lua数据结构
- lua_restry_dns: 基于cosocket实现dns协议的通信。
- lua_resty_redis: 基于ngx.socket.tcp实现的redis客户端。
- lua_resty_string: 字符串转换函数
  
详细模块参考： https://openresty.org/cn/components.html


如何在nginx中嵌入lua
------------------------------------

.. image:: ../images/nginx40.png


在nginx过程嵌入lua代码
------------------------------------

- init_by_lua , init_by_lua_block , init_by_lua_file : master启动。
- init_worker_by_lua: work启动的时候调用
- set_by_lua: 
- rewrite_by_lua: 
- access_by_lua: 
- content_by_lua: 
- log_by_lua: 


lua ffi
------------------------------------
提供一种lua语音使用c语言函数的功能。

系统级配置指令
------------------------------------

- lua_malloc_trim: 每N个请求使用mallock_trim方法，将缓存的空闲内存归还操作系统。
- lua_code_cache: lua vm的所有请求共享
- lua_package_path: 设置lua模块的路径
- lua_package_cpath: 设置lua调用c模块的路径地址。


nginx的变量
------------------------------------

ngx.var.VAR_NAME 可以访问和修改变量

ngx.req
------------------------------------

- ngx.req.get_headers 
- ngx.req.get_method
- ngx.req.http_version
- ngx.req.get_uri_args
- ngx.arg[index]
- ngx.req.get_post_args 
- ngx.req.read_body 
- nginx.req.get_body_data
- nginx.req.get_body_file
  
发送响应sdk
------------------------------------
- ngx.print 
- ngx.say 
- ngx.flush 
- ngx.exit 
- ngx.eof

日志
------------------------------------

详细参考： https://github.com/openresty/lua-nginx-module#ngxlog

cosocket
------------------------------------

- lua_socket_connect_time: 连接超时时间
- lua_socket_send_timeout: 2次写超时时间
- lua_socket_read_timeout: 2次读超时时间
- lua_socket_bufer_size：  设置读缓冲区大小
- lua_socket_pool_size: 设置连接池最大连接数
- lua_socket_keepalive_timeout: 连接空闲时间
- lua_socket_log_errors: 是否记录错误日志到nginx的error.log中。

- ngx.socket.tcp : tcp相关方法函数
- nginx.socket.socket: 获取socket对象
- ngx.socket.udp： udp client 


多协程方法
------------------------------------

- ngx.thread.spawn ： 生成轻量级线程。
- coroutine create : 创建lua协程。

la_resty_lock锁
------------------------------------

- lock: 锁定
- unlock: 解锁
- expire: timeout 最大等待时间。
  

定时器
------------------------------------

- ngx.timer.at 定时器触发后执行callback
- ngx.timer.every: 每多久执行一次
- ngx.timer.runing_count: 运行的定时器数量
- ngx.timer.pending_count 等待执行的定时器数量

共享内存
------------------------------------
提供跨work的共享内存共享内容机制。是原子的，线程安全的。ngx.shared.DICT.

nginx主请求和子请求
------------------------------------

1. 子请求的生命周期是依赖父请求
2. 父请求可以通过postpone_filter处理子请求的响应。
3. ngx.location.capture 可以生成子请求。


基于openrestry的waf防火墙
------------------------------------

github search waf 


