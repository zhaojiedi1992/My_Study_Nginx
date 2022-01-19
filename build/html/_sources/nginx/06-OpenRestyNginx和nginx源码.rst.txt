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

.. literalinclude:: ../files/if_double.conf
   :encoding: utf-8
   :language: bash

验证下

.. code-block:: bash 
    #todo 

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

todo

debug定位问题
------------------------------------
我们引入了第三方模块可能存在bug的，打开debug_points后则遇到问题后停止服务，方便定位问题。

- abort: 生成coredump后结束进程
- stop: 结束进程

todo


控制debug级别的error.log日志输出
------------------------------------
debug_connection针对特定客户端打印debug级别的日志，其他的日志正常打印。 

.. note:: 需要nginx编译的时候--with-debug编译选项。

debug日志分析
------------------------------------

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







