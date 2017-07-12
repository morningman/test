# Palo 编译及部署

## Palo 编译
本文档基于以下 Linux 发行版本进行说明：

    Ubuntu 16.04.2 (64bit)
    CentOS 7.1 x86_64 (64bit)

其他系统环境请参照本文档进行操作。如有问题，欢迎通过 issue 或者邮件反馈。

### 脚本说明

**build.sh**

Palo 源码编译的入口脚本。

第三方依赖库的编译脚本放在 thirdparty 下。

**thirdparty/vars.sh**

定义了编译所需的环境变量，以及所有第三方库的下载路径。

`PARALLEL:` 编译 C/C++ 程序时的并行度，即 make -j 的参数。

`JAVA_HOME：` 设置JDK所在路径。

**thirdparty/download-thirdparty.sh**

所有第三方库的下载、解压和 patch 操作。

**thirdparty/build-thirdparty.sh**

编译第三方库的入口脚本，其中会调用 download-thirdparty.sh。

不会重复下载、解压和 patch 第三方库。

默认情况下，用户直接执行 `sh thirdparty/build-thirdparty.sh` 即可开始下载及编译第三方库。

### 环境准备

系统版本：

    Ubuntu 16.04.2 (64bit)
    或
    CentOS 7.1 x86_64 (64bit)

GCC：

    4.8.2 或更高

JDK：

    jdk-7u80-linux-x64 或更高。如需要，请前往 oracle 网站下载并部署。

    如需要，请在 thirdparty/vars.sh 中设置 JAVA_HOME 环境变量。

### 编译步骤

1. 安装以下依赖：

    Ubuntu:

    `sudo apt-get install g++ ant cmake zip byacc flex automake libtool binutils-dev libiberty-dev bison`

    CentOS:

    `sudo yum install ant cmake byacc flex automake libtool binutils-devel bison`

2. 安装完成第一步后，需执行：

    `sudo updatedb`

    用以更新 locate 使用的数据库，以方便使用 locate 命令检查依赖库是否存在。

3. 执行编译第三方库：

   `sh thirdparty/build-thirdparty.sh`

    所有第三方库源码将下载并解压到: thirdparty/src 中

    第三方库安装路径：thirdparty/installed

4. 执行 Palo 编译脚本。

    `sh build.sh`

    最终的部署文件生成在 output/ 路径下，分为 Frontend（FE） 和 Backend（BE）。

### 常见编译问题

1. 执行 `sh build-thirdparty.sh` 报错: source: not found

    请使用 /bin/bash 代替 sh 执行脚本。

2. 编译 gperftools
    - 找不到 libunwind

        创建到 libunwind.so.x 的软链：

        `cd thirdparty/installed/lib && ln -s libunwind.so.8 libunwind.so`

        之后重新执行：

        `build-thirdparty.sh`

3. 编译 thrift
    - 找不到 libssl 或 libcrypto

        创建到系统 libssl.so.x 的软链：

        `cd thirdparty/install/lib`

        `rm libssl.so libcrypto.so`

        `ln -s /usr/lib64/libssl.so.10 libssl.so`

        `ln -s /lib64/libcrypto.so.10 libcrypto.so`

        (系统库路径可能不相同，请对应修改)

        在 thirdparty/build-thirdparty.sh 中，注释掉 build_openssl

        之后重新执行：

        `build-thirdparty.sh`

    - thrifty.hh 不存在

        thirdparty/patches/thrift-0.8.0.patch 应该已经解决了这个问题。如还遇到此问题，请执行： 

        `mv thirdparty/src/thrift-0.8.0/compiler/cpp/thrifty.hh thirdparty/src/thrift-0.8.0/compiler/cpp/thrifty.h`

        之后重新执行：

        `sh build-thirdparty.sh`

    - No rule to make target \`gen-cpp/Service.cpp', needed by \`Service.lo'.  Stop.

        重新执行：
        
        `sh build-thirdparty.sh`

    - Boost.Context fails to build -> Call of overloaded 'callcc(...) is ambiguous' 

        如果你使用 gcc 4.8 或 4.9 版本，则可能出现这个问题。
        
        执行以下命令：
        
        `cd thirdparty/src/boost_1_64_0`
        
        `patch -p0 < ../../patches/boost-1.64.0-gcc4.8.patch`
        
        之后重新执行：
        
        `sh build-thirdparty.sh`
        
        > 参考：https://github.com/boostorg/fiber/issues/121
        
    - syntax error near unexpected token `GLIB,'
        
        检查是否安装了 pkg-config，并且版本高于 0.22。
        
        如果已经安装，但依然出现此问题，请删除 `thirdparty/src/thrift-0.8.0` 后，重新执行：
        
        `sh build-thirdparty.sh`

4. 编译 curl
    - libcurl.so: undefined reference to `SSL_get0_alpn_selected'

        在 thirdparty/build-thirdparty.sh 中确认没有注释掉 build_openssl。

        同时注释掉 build_thrift。

        之后重新执行：

        `sh build-thirdparty.sh`

5. 编译 Palo
    - [bits/c++config.h] 或 [cstdef] 找不到

        在 be/CMakeLists.txt 中修改对应的 CLANG_BASE_FLAGS 中设置的 include 路径。

        可以通过命令：

        `locate c++config.h`

        确认头文件的地址。

    - cstddef: no member named 'max_align_t' in the global namespace

        在 Ubuntu 16.04 环境下可能会遇到此问题。

        首先通过 `locate cstddef` 定位到系统的 cstddef 文件位置。

        打开 cstddef 文件，修改如下片段：

        `namespace std 
        { 
        // We handle size_t, ptrdiff_t, and nullptr_t in c++config.h. 
        +#ifndef __clang__ 
        using ::max_align_t; 
        +#endif 
        } 
        #endif `
        

        > 参考：

    - /bin/bash^M: bad interpreter: No such file or directory

        对应的脚本程序可能采用了 window 的换行格式。

        使用 vim 打开对应的脚本文件，执行：

        `:set ff=unix`

        保存并退出。

## Broker 编译

Broker 用于从其他数据源（如 HDFS、BAIDU BOS）导入数据。

1. 进入 fs_brokers/ 下对应 Broker 的目录。

2. 执行 `sh build.sh` 开始编译。

3. 最终的部署文件生成在对应 Broker 的 output/ 路径下。

## 部署

Palo 主进程包括 Frontend（FE）和 Backend（BE）。

FE 主要负责以下工作：

- 元数据管理
- 集群管理
- 接收用户请求
- 查询计划生成

BE 主要负责以下工作：

- 数据存储
- 查询计划执行

### 单机部署

通过在一台机器上搭建仅包含一个 FE 和一个 BE 的 Palo，快速熟悉和使用 Palo。

1. 准备部署环境

   将编译生成在 output/ 路径下的 fe/ 和 be/ 文件夹，拷贝到自定义的部署路径下。

2. FE 配置及启动

   1. 配置 fe/conf/fe.conf

        `JAVA_HOME`：绝对路径，指向 jdk 所在位置（请使用与编译时相同版本的 JDK ）。

        `meta_dir`：元数据存放位置。默认在 fe/Palo-meta/ 下。需**手动创建**该目录。

   2. 启动 FE

        `sh bin/start_fe.sh`

        FE 进程将启动并进入后台执行。日志默认存放在 fe/log/ 目录下。

        如启动失败，可以通过查看 fe/log/fe.log 或者 fe/log/fe.out 查看错误信息。

3. BE 配置及启动

   1. 配置 be/conf/be.conf

        `storage_root_path`：数据存放目录。使用 `;` 分隔（最后一个目录后不要加 `;`）

   2. 添加 BE

        BE 节点需要先在 FE 中添加，才可加入集群。
        
        请使用任意 mysql-client 连接到 FE：
        
        `./mysql-client -h host -P port -uroot`

        其中 host 为 FE 所在节点 ip；port 为 fe/conf/fe.conf 中的 query_port；默认使用root账户，无密码登录。

        登录后，执行以下命令添加 BE：

        `ALTER SYSTEM ADD BACKEND "host:port";`

        其中 host 为 BE 所在节点 ip；port 为 be/conf/be.conf 中的 heartbeat_service_port。

   3. 启动 BE

        `sh bin/start_be.sh`

        BE 进程将启动并进入后台执行。日志默认存放在 be/log/ 目录下。

        如启动失败，可以通过查看 be/log/be.INFO 或者 be/log/be.out 查看错误信息。

   4. 查看 BE 状态

        再次使用 mysql-client 连接到 FE，并执行：

        `SHOW PROC '/backends';`

        查看 BE 运行情况。如一切正常，`isAlive` 列应为 `true`。

### 分布式部署

Palo 支持 FE 节点的高可用，以及 BE 节点的横向扩展。在单机部署之上，可以通过在其他节点添加更多的 FE 或 BE，以提高 Palo 的可用性和性能。

1. FE 高可用及扩展。

    在分布式环境下，FE 分为 Leader，Follower 和 Observer 三种角色。 

    一个 Leader 和多个 Follower 组成一个选举组，在 Leader 宕机情况下，剩余的 Follower 会选出新的 Leader，保证 FE 的高可用。

    Observer 角色不参与 Leader 的选举，但可以接收所有用户请求，以保证 FE 的扩展性。

    第一个启动的 FE 自动成为 Leader。在此基础上，可以添加若干 Follower 和 Observer。

    1. 添加 Follower 或 Observer

        使用 mysql-client 连接到已启动的 FE，并执行：

        `ALTER SYSTEM ADD FOLLOWER "host:port";`

        或

        `ALTER SYSTEM ADD OBSERVER "host:port";`

        其中 host 为 Follower 或 Observer 所在节点 ip；port 为其配置文件 fe.conf 中的 edit_log_port。

    2. 配置及启动 Follower 或 Observer

        Follower 和 Observer 的配置同 Leader 的配置。

        第一次启动时，需执行以下命令：

        `sh bin/start_fe.sh -helper host:port`

        其中 host 为 Leader 所在节点 ip；port 为 Leader 的配置文件 fe.conf 中的 edit_log_port。

        `-helper` 参数仅在 Follower 和 Observer 第一次启动时才需要。再次启动时，直接执行：

        `sh bin/start_fe.sh`

        即可。

    3. 查看 Follower 或 Observer 运行状态

        使用 mysql-client 连接到**任一**已启动的 FE，并执行：

        `SHOW PROC '/frontend';`

        可以查看当前已加入集群的 FE 及其对应角色。

2. BE 横向扩展

    同**单机部署**章节中一样。我们通过以下步骤添加更多的 BE：

    1. 使用 mysql-client 连接任一已启动的 FE，执行 `ALTER SYSTEM ADD BACKEND` 语句添加 BE。
    2. 部署、配置并使用 `sh bin/start_be.sh` 命令启动 BE。
    3. 使用 mysql-client 连接任一已启动的 FE，执行 `SHOW PROC '/backends';` 查看 BE 运行状况。

### Broker 部署

Broker 以插件的形式，独立于 Palo 部署。

1. 准备部署环境

    进入对应的 Broker 的 output 目录，将编译生成的目录拷贝到自定义部署目录下。

2. 配置 Broker

    在 conf/ 目录下对应的配置文件中，配置 Broker 启动参数（如端口）。

3. 添加 Broker

    使用 mysql-client 连接任一已启动的 FE，执行以下命令：

    `ALTER SYSTEM ADD BROKER broker_name "host:port";`

    其中 host 为 Broker 所在节点 ip；port 为 Broker 配置文件中的 broker_ipc_port。

4. 查看 Broker 状态

    使用 mysql-client 连接任一已启动的 FE，执行以下命令查看 Broker 状态：

    `SHOW PROC "/brokers";`

5. 多 Broker

    通过部署多个 Broker 可以提高 Palo 通过 Broker 访问数据的并发度和性能。

    首先我们可以在不同节点启动多个 Broker。

    然后使用 mysql-client 连接任一已启动的 FE，执行以下命令：

    `ALTER SYSTEM ADD BROKER broker_name "host1:port1","host2:port2",...;`

    其中 host* 为各个 Broker 所在节点ip；port* 为对应 Broker 的配置文件中的 broker_ipc_port。
