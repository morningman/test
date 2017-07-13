# Palo安装文档

本文档会介绍Palo的系统依赖，编译，部署和常见问题解决。

## 1. 系统依赖

Palo当前只能运行在Linux系统上，无论是编译还是部署，都建议确保系统安装了如下软件或者库：
* GCC 4.8.2+，Oracle JDK 1.7+，Python 2.7+，确认gcc, java, python命令指向正确版本, 设置JAVA_HOME环境变量

* Ubuntu需要安装：`sudo apt-get install g++ ant cmake zip byacc flex automake libtool binutils-dev libiberty-dev bison`;  安转完成后，需要执行`sudo updatedb`

* CentOS需要安装: `sudo yum install ant cmake byacc flex automake libtool binutils-devel bison`; 安装完成后，需要执行`sudo updatedb`

## 2. 编译

默认提供了 Ubuntu 16.04, Centos 7.1 环境的预编译版本，可以直接下载使用。如果预编译版本有问题，或者是其它系统，建议按照下面步骤进行源码编译。

### 2.1 编译第三方依赖库
``
sh thirdparty/build-thirdparty.sh
``

_注意:build-thirdparty.sh 依赖thirdparty目录下的其它两个脚本，其中vars.sh定义了一些编译第三方库时依赖的环境变量，download-thirdparty.sh负责完成对依赖源码包的下载。_

### 2.2 编译 Palo FE 和 BE
``
sh build.sh
``
### 2.3 (可选) 编译 FS_Broker

FS_Broker用于从其他数据源（如Hadoop HDFS、百度云BOS）导入数据时使用，如果不需要从这两个数据源导入数据可以先不编译和部署。需要的时候，可以后面再编译和部署。
需要哪个broker，就进入fs_brokers/下对应的 broker 的目录，执行`sh build.sh`即可，执行完毕后，产生的部署文件生成在对应 broker的output目录下。

## 3. 部署

Palo主要包括Frontend（FE）和 Backend（BE）两个进程。其中FE主要负责元数据管理、集群管理、接收用户请求和查询计划生成；BE主要负责数据存储和查询计划执行。

一般100台集群，可能根据性能需求部署1到5台FE，而剩下的全部机器部署 BE。其中建议FE 的机器采用带有RAID卡的 SAS 硬盘，或者SSD硬盘，不建议使用普通 SATA 硬盘；而BE 对硬盘没有太多要求。

### 3.1 单FE部署

* 拷贝FE部署文件到指定节点
将源码编译生成的 output 下的 fe 文件夹拷贝到要部署 FE 的节点的一个账户根目录下的fe目录下

* 配置FE
配置文件为 conf/fe.conf。其中注意：`meta_dir`：元数据存放位置。默认在 fe/palo-meta/ 下。需**手动创建**该目录。

* 启动FE
`sh bin/start_fe.sh`
FE进程启动进入后台执行。日志默认存放在 fe/log/ 目录下。如启动失败，可以通过查看 fe/log/fe.log 或者 fe/log/fe.out 查看错误信息。

* 如需部署多FE，请参见常见问题中的“FE高可用”


### 3.2 多BE部署

* 拷贝BE部署文件到所有要部署BE的节点
将源码编译生成的 output 下的 be 文件夹拷贝到要部署 BE 的节点的一个账户根目录下的be目录下

* 修改所有 BE 的配置
修改 be/conf/be.conf。主要是配置`storage_root_path`：数据存放目录，使用 `;` 分隔（最后一个目录后不要加 `;`），其它可以采用默认值。

* 在 FE 中添加所有 BE 节点
BE节点需要先在FE中添加，才可加入集群。可以使用mysql-client连接到FE：`./mysql-client -h host -P port -uroot`，其中 host 为 FE 所在节点 ip；port 为 fe/conf/fe.conf 中的 query_port；默认使用root账户，无密码登录。
登录后，执行以下命令来添加每一个BE：`ALTER SYSTEM ADD BACKEND "host:port";`，其中 host 为 BE所在节点 ip；port 为 be/conf/be.conf 中的 heartbeat_service_port。

* 启动BE
`sh bin/start_be.sh`
BE进程将启动并进入后台执行。日志默认存放在 be/log/目录下。如启动失败，可以通过查看 be/log/be.log 或者 be/log/be.out 查看错误信息。

* 查看BE状态
使用 mysql-client 连接到 FE，并执行`SHOW PROC '/backends';`查看 BE 运行情况。如一切正常，`isAlive` 列应为 `true`。


### 3.3 （可选）FS_Broker部署

Broker以插件的形式，独立于 Palo 部署。如果需要从第三方存储系统导入数据，需要部署相应的 broker，默认提供了读取HDFS和百度云BOS的 fs_broker。fs_broker 是无状态的，建议每一个 fe 和 be 节点都部署一个 broker。

* 拷贝源码fs_broker的 output目录下的相应broker目录到需要部署的所有节点上。
建议和 be或者 fe 目录保持同级。

* 修改相应broker配置
在相应broker/conf/ 目录下对应的配置文件中，可以修改相应配置。

* 添加broker
要让 palo 的 fe 和 be 知道 broker 在哪些节点上，通过 sql 命令添加 broker 节点列表。
使用 mysql-client 连接启动的 FE，执行以下命令：`ALTER SYSTEM ADD BROKER broker_name "host1:port1","host2:port2",...;` 其中 host 为broker 所在节点 ip；port 为 broker 配置文件中的 broker_ipc_port。

* 查看broker 状态
使用 mysql-client 连接任一已启动的 FE，执行以下命令查看 Broker 状态：`SHOW PROC "/brokers";`

## 4. 常见问题

* 执行 `sh build-thirdparty.sh` 报错: source: not found

    请使用 /bin/bash 代替 sh 执行脚本。

* 编译 gperftools：找不到 libunwind

    创建到 libunwind.so.x 的软链：`cd thirdparty/installed/lib && ln -s libunwind.so.8 libunwind.so`，之后重新执行：`build-thirdparty.sh`

* 编译 thrift：找不到 libssl 或 libcrypto

    创建到系统 libssl.so.x 的软链： `cd thirdparty/install/lib`；`rm libssl.so libcrypto.so`；`ln -s /usr/lib64/libssl.so.10 libssl.so`； `ln -s /lib64/libcrypto.so.10 libcrypto.so` (系统库路径可能不相同，请对应修改)；在 thirdparty/build-thirdparty.sh 中，注释掉 build_openssl之后重新执行`build-thirdparty.sh`

* 编译 thrift：No rule to make target \`gen-cpp/Service.cpp', needed by \`Service.lo'.  Stop.

    重新执行：`sh build-thirdparty.sh`

* 编译 thrift：Boost.Context fails to build -> Call of overloaded 'callcc(...) is ambiguous' 

    如果你使用 gcc 4.8 或 4.9 版本，则可能出现这个问题。执行以下命令：`cd thirdparty/src/boost_1_64_0`；`patch -p0 < ../../patches/boost-1.64.0-gcc4.8.patch`；之后重新执行：`sh build-thirdparty.sh`；参考：https://github.com/boostorg/fiber/issues/121
 
* 编译 thrift：syntax error near unexpected token `GLIB,'

    检查是否安装了 pkg-config，并且版本高于 0.22。如果已经安装，但依然出现此问题，请删除 `thirdparty/src/thrift-0.9.3` 后，重新执行：sh build-thirdparty.sh`

* 编译 curl：libcurl.so: undefined reference to `SSL_get0_alpn_selected'

    在 thirdparty/build-thirdparty.sh 中确认没有注释掉 build_openssl。同时注释掉 build_thrift。之后重新执行：`sh build-thirdparty.sh`

* 编译 Palo FE 和 BE：[bits/c++config.h] 或 [cstdef] 找不到

    在 be/CMakeLists.txt 中修改对应的 CLANG_BASE_FLAGS 中设置的 include 路径。可以通过命令：`locate c++config.h`确认头文件的地址。

* 编译 Palo FE 和 BE：cstddef: no member named 'max_align_t' in the global namespace

    在 Ubuntu 16.04 环境下可能会遇到此问题。首先通过 `locate cstddef` 定位到系统的 cstddef 文件位置。打开 cstddef 文件，修改如下片段：
    ```
    namespace std {
      // We handle size_t, ptrdiff_t, and nullptr_t in c++config.h.
      +#ifndef __clang__
      using ::max_align_t;
      +#endif
    }
    ```
    参考：http://clang-developers.42468.n3.nabble.com/another-try-lastest-ubuntu-14-10-gcc4-9-1-can-t-build-clang-td4043875.html

* 编译 Palo FE 和 BE：/bin/bash^M: bad interpreter: No such file or directory

    对应的脚本程序可能采用了 window 的换行格式。使用 vim 打开对应的脚本文件，执行： `:set ff=unix` 保存并退出。

* FE 高可用

    FE分为 leader，follower 和 observer 三种角色。 默认一个集群，只能有一个 leader，可以有多个 follower 和 observer。其中 leader 和 follower 组成一个 Paxos 选择组，如果 leader 宕机，则剩下的 follower 会自动选出新的 leader，保证写入高可用。observer同步 leader 的数据，但是不参加选举。如果只部署一个 FE，则 FE默认就是 leader。
    
    第一个启动的 FE 自动成为 leader。在此基础上，可以添加若干 follower 和 observer。
    
    添加 follower 或 observer。使用 mysql-client 连接到已启动的 FE，并执行：ALTER SYSTEM ADD FOLLOWER "host:port"或ALTER SYSTEM ADD OBSERVER "host:port"; 其中 host 为 follower 或 observer 所在节点 ip；port 为其配置文件fe.conf中的 edit_log_port。
    
    配置及启动 follower 或 observer。follower 和 observer 的配置同 leader 的配置。第一次启动时，需执行以下命令：sh bin/start_fe.sh -helper host:port 其中host为 Leader 所在节点 ip；port 为 Leader 的配置文件 fe.conf 中的 edit_log_port。-helper 参数仅在 follower 和 observer 第一次启动时才需要。
    
    查看 follower 或 observer 运行状态。使用 mysql-client 连接到任一已启动的 FE，并执行：SHOW PROC '/frontend'; 可以查看当前已加入集群的 FE 及其对应角色。
