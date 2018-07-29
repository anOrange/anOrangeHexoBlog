---
title: solrcloud搭建记录
date: 2017-05-10 10:22:54
tags: [solr,zookeeper,solrcloud,搭建]
categories: solr
---
<!-- TOC -->

- [背景](#背景)
- [搭建流程](#搭建流程)
- [具体操作](#具体操作)
    - [1. 原料准备](#1-原料准备)
    - [2. webserver的配置](#2-webserver的配置)
    - [3. 在tomcat中部署solr](#3-在tomcat中部署solr)
    - [4. 部署zookeeper](#4-部署zookeeper)
    - [5. solr运行在zookeeper](#5-solr运行在zookeeper)
    - [6. 上传配置以及新建collection](#6-上传配置以及新建collection)
    - [7. 访问控制](#7-访问控制)
- [后续](#后续)
- [参考](#参考)

<!-- /TOC -->
# 背景
最近两个迭代需要优化搜索功能。因为最开始部门的搜索是lucene1.x或者2.x时代弄的，已经不能用了。去年的同事重新用solr-4.10.4搭了个服务，前几个月也走了，目前这个solr是处于没人维护的状态。而这个给上层提供基础全文检索的solr服务已经不能满足需求了，必须要改动。这次重新搭建了solrcloud服务,现在把搭建过程整理出来。  
迭代开始前，去了解了lucene和solr，关于这两个东西的认识：*Lucene就是一套提供基于倒排索引()来实现全文检索的java包，solr又是基于Lucene封装成能很方便的通过http api请求来操作的web服务(还要借助于tomcat或者jetty等web容器来实现)。*
<!-- more -->
根据之前的同事留下的配置，有3台搜索服务器，看了下php代码，实际只用了一台。上服务器看也是单点。因为不少业务依赖这个全文搜索，想到两种方案:
- 一种是把没有在负载上面的两台配置成主从，然后把负载切到主从上；最后配置好那台单点，全流量
- 还有一种就是solrcloud  

偶然在公司平台上发现实际上我们有超过10台虚拟机可以用来搭建搜索服务，那就一次到位，用solrcloud了。  
另外还有都是基于lucene的Elasticsearch和Solr的对比，可以参考下。

# 搭建流程
1. 原料准备:solr-6.5.0.tgz;jdk1.8;zookeeper;tomcat8;几个分词库
2. 配置webserver，jdk和tomcat
3. 在tomcat中部署solr
4. zookeeper部署
5. solr运行在zookeeper
6. 上传配置以及新建collection
7. 访问控制

*注:如果不需要配置solrcloud，前三步，再在solrconfig.xml里面配置主从同步就可以用了*

# 具体操作
## 1. 原料准备
  
  目前官方的最新lucene版本时6.5.1,solr的是6.5.0。拍拍脑袋就大胆用最新版本搭建了，官网直接下载solr-6.5.0.tar.gz包即可。官方链接:http://lucene.apache.org/solr/  
  solrcloud需要用到zookeeper、tomcat(也可以用自带的jetty，不过tomcat还是稍微熟悉点，配置也比较容易，出问题还可以问人)、jdk和一些扩展的jar包，大致如下：
  1. jdk1.8 : zookeeper、solr和tomcat都需要jdk,solr6.5.0依赖1.8以上的jdk。直接从官网上下jdk-8u131-macosx-x64.dmg或者jdk-8u131-linux-x64.tar.gz。
  2. zookeeper，在官网下载zookeeper3.4.10版本。
  4. 然后我部署分词库用的是ik，拼音用的是pinyin4j。这两个库都是开源的，可以下载下来定制。
  3. 关于tomcat，wiki上面是这样说的:
    > No Longer Supported
    > Beginning with Solr 5.0, Support for deploying Solr as a WAR in servlet containers like Tomcat is no longer supported.
    > For information on how to install Solr as a standalone server, please see Installing Solr.
  
  
  网上说用tomcat8不会出问题，这里用的是tomcat8。
  都下载好后，分别解压到一个目录，备用。

## 2. webserver的配置  
  * 先配置jdk1.8(已经有的忽略)。
  * 如果是mac环境，官网下载好的是一个dmg包，双击自动安装即可。
  * 如果是Linux环境，将安装包解压后，配置环境变量$JAVA_HOME和$PATH，记得用source指令生效:
    ```
    JAVA_HOME=/home/work/webserver/jdk
    JAVA_BIN=/home/work/webserver/jdk/bin
    PATH=$PATH:$JAVA_BIN
    CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    export JAVA_HOME JAVA_BIN PATH CLASSPATH
    ```
  * 启动tomcat，同样解压tomcat的tar包，如果不做特别的配置，直接进入tomcat的bin目录，执行./startup.sh启动tomcat服务。
  * 没什么问题的话，浏览器访问8080端口就能看到tomcat的欢迎页了。


## 3. 在tomcat中部署solr  
  solr 的安装也很简单。如果不用tomcat,solr5之后，solr都自带jetty，里面有已经配置好，可以直接执行jetty的start.jar来启动solr服务。  
  如果需要在 tomcat 中启动，我们只需要将solr的站点和jar包从jetty的项目下面拷贝到tomcat，稍微做些配置就可以了。

  ** 具体步骤 ：**
  1. 拷贝solr 站点（提供 solr 的web服务主要就靠它了）。进入solr 包解压目录下, 将server/solr-webapp 目录下面的webapp文件夹拷贝到tomcat的webapp目录中，命名为solr（就是站点名字，可根据自己需求命名）;
  2. 拷贝依赖的jar包，各种用到的方法都封装在里面了。将下面这些 jar 包拷贝到tomcat/webapps/solr/WEB-INF/lib 中:
  1) solr/server/lib/ext 中的*.jar
  2) solr/server/lib/metrics*
  3) 所使用分词库和拼音处理的lib，这里是ik和pinyin4j
  3. 拷贝配置文件到 tomcat/webapps/solr/WEB-INF/classes 里的
  1) solr/resource/solr-6.5.1/server/resources/log4j.properties
  2) 如果使用了ik分词库,将分词库中的 IKAnalyzer.cfg  文件拷贝过来并配置词库路径(词库位置是相对于classes目录)
  4. 配置 tomcat/webapp/solr/WEB-INF/web.xml 中的 solr/home 节点，如下
  ```
  <env-entry>
  <env-entry-name>solr/home</env-entry-name>
  <env-entry-value>~/solr-cores</env-entry-value>
  <env-entry-type>java.lang.String</env-entry-type>
  </env-entry>
  ```
  5. 创建 ~/solr-cores 文件夹(用来保存cores的) ，并将 solr 包路径下面的 server/solr/solr.xml 文件拷贝到 ~/solr-cores文件夹中

  完成上面几步，重新启动tomcat，访问localhost:8080/solr/index.html，如果一切正常，就可以进入solr的web后台了。  
  如果访问出错，可以根据 log4j.properties 中配置的 log 名称，找到运行日志，查看报错；或者可以就通过 tomcat/bin/catalina.sh run 来运行，直接在控制台中查看报错。  
  **这时候单点的 solr 服务就算安装完毕了。**

  *注:solr6已经不需要在tomcat中手动添加tomcat-user.xml中来进行权限控制，可以通过security.json文件来配置。*

## 4. 部署zookeeper  

  进入zookeeper安装目录, 执行 
  ```
  cp ./conf/zoo_sample.cfg ./conf/zoo.cfg 
  ```
  然后编辑 ./conf/zoo.cfg ,我的配置如下：
  ```
  tickTime=2000
  initLimit=10
  syncLimit=5
  dataDir=/home/work/data/zookeeper-data
  dataLogDir=/home/work/data/zookeeper-data/logs
  server.1=BOE-server-08:2888:3888
  server.2=BOE-server-07:2888:3888
  #zookeeper的节点列表，格式如下
  #server.节点编号=节点host:通信端口:选主端口
  #...
  server.10=LOC-search-03:2888:3888
  server.11=LOB-search-04:2888:3888
  clientPort=2181
  ```
  新建下面目录：  
  ```
  /home/work/data/zookeeper-data  
  /home/work/data/zookeeper-data/logs
  ```
  然后新建myid:
  ```
  echo '1' >> /home/work/data/zookeeper-data/myid
  ```
  期中 1 为该机器的节点编号。
  
  启动 zookeeper/bin 目录下面的 zookeeper 服务 ( zookeeper.out文件默认在启动目录下生成 ):
  ```
  [work@LOB-search-04 bin]$ ./zkServer.sh start
  ZooKeeper JMX enabled by default
  Using config: /home/work/zookeeper/bin/../conf/zoo.cfg
  Starting zookeeper ... STARTED
  ```
  查看 zookeeper 运行启动状态，下面状态就说名正常了，如果不正常，会提示没有启动
  ```
  [work@LOB-search-04 bin]$ ./zkServer.sh status
  ZooKeeper JMX enabled by default
  Using config: /home/work/zookeeper/bin/../conf/zoo.cfg
  Mode: follower
  ```
  最后各个节点按照相同的方法部署即可。     

## 5. solr运行在zookeeper  
  以cloud模式启动solr的方法很多，我是通过在tomcat启动脚本添加参数来实现的，具体做法如下：  
  在 tomcat/bin 目录下，将 catalina.sh 和 startup.sh 两个文件分别拷贝一份，命名为 catalina_zk.sh 和 startup_zk.sh。
  修改startup_zk.sh的 EXECUTABLE 变量，改为：
  ```
  EXECUTABLE=catalina_zk.sh
  ```
  在 catalina_zk.sh 中添加下面参数:
  ```
  JAVA_OPTS="$JAVA_OPTS -DzkHost=BOE-server-08:2181,BOE-server-07:2181,EON-server-06:2181"
  ```
  使用startup_zk.sh 脚本启动solr，就可以以cloud模式启动solr了。  
  此时再进入 solr 管理后台，Logging选项下面就会多出一个cloud分栏的选项了。
  
## 6. 上传配置以及新建collection  
  
  1. 建立配置  
  进入 solr 解压目录，将 server/solr/configsets/basic_configs 目录拷贝一份，改名为 myexampleconfig。  
  根据需要可以修改myexampleconfig/conf文件夹中的managed-schema文件和solrconfig.xml，也可以后到solradmin中修改。
  2. 上传配置  
  有了配置之后，需要上传到 zookeeper 上让 solr 可以读取到。使用solr提供的工具，可以很方便的上传配置文件。是调用org.apache.solr.cloud.ZkCLI这个类的upconfig这个命令来上传的。  
  我将上传的方法写到一个如下的配置文件：
  ```
  #!/bin/sh
  configName=$1
  zkHost=BOE-server-08:2181,BOE-server-07:2181,EON-server-06:2181
  if [ -z "$configName" ]; then
    echo "配置名为空"
    echo "exit 1"
    exit 1
  fi
  configPath=$(cd `dirname $0`; pwd)"/../configsets/"$configName"/conf"
  if [ ! -d "$configPath" ]; then
    echo "配置目录\""$configPath"\"不存在"
    echo "exit 2"
    exit 2
  fi
  echo "开始上传配置,配置名:"$configName
  java -classpath .:$(cd `dirname $0`; pwd)/../tomcat8/webapps/solr/WEB-INF/lib/* \
    org.apache.solr.cloud.ZkCLI -cmd upconfig \
    -zkhost $zkHost \
    -confdir $configPath -confname $configName
  ```
  这个脚本放在相对于配置文件目录上级目录的./script/目录下面，命名为uploadconfig.sh，执行该脚本将配置上传到zookeeper:
  ```
  panzhuodeMacBook-Pro:script pan$ ./uploadConfig.sh myexampleconfig
  ```
  上传完毕后，点开 solrAdmin 中的 cloud 栏目的 Tree 子栏目，里面可以查看zookeeper节点中的目录，应该能在configs中看到刚才上传的配置文件。
  3. 新建collection  
  网上很多都是使用webapi的方式访问url新建，官网上文档说的比较详细。我这里用的比较懒的办法，直接在solrAdmin的collections分栏下面点击“Add collection”新建。
  
    
## 7. 访问控制
   
  solr作为web服务运行，如果不做权限控制任何能访问这个web地址的人都能随意对上面数据进行增删改查，这是不能接受的。对此，有必要进行权限控制的配置:
  1. http访问控制
  http访问验证是solr5以后提供的，可以让用户以不同的身份访问solr时获得不同的权限级别。操作如下:  
  1) 新建security.json,内容如下
  ``` 
  {
  "authentication":{
    "class":"solr.BasicAuthPlugin",
    "blockUnknown": true,
    "credentials":{"solr":"IV0EHq1OnNrj6gvRCwvFwTrZ1+z1oBbnQdiVC3otuq0= Ndd7LKvVBAaZIF0QAVi1ekCfAJXr1GGfLtRUXhgrF8c="}
  },
  "authorization":{
    "class":"solr.RuleBasedAuthorizationPlugin",
    "permissions":[{"name":"security-edit",
    "role":"admin"}]
    "user-role":{"solr":"admin"}
  }}
  ```
  2) 上传到zookeeper根目录  
  在solr解压目录下，执行:
  ```
  ./bin/solr zk cp file:/home/work/solrspace/security.json zk:/security.json -z BOE-server-08:2181,BOE-server-07:2181
  ```
  3) 使用账户名为"solr"，密码为"SolrRocks"认证就可以以admin的角色访问solr访问solrAdmin。给予这个权限，可以使用webapi添加修改用户或者给用户各个节点不同的读写权限，具体配置可以看[文档](http://people.apache.org/~ctargett/RefGuidePOC/jekyll-full/rule-based-authorization-plugin.html)。
  
  2. zookeeper访问权限控制  
   
  solr支持zookeeper的acl权限控制，防止未经验证的用户修改zookeeper文件。  
  **注意,下面的操作需要在关闭solr服务的情况下进行。**  
  1) zookeeper的digest授权方式的账户密码是要讲密码先sha1，再将结果base64来编码，要先通过下面指令生成密码:  
    ```
    echo -n username:password | openssl dgst -binary -sha1 | openssl base64
    ```
    得到"+Ir5sN1lGJEEs8xBZhZXKvjLJ7c="这个字符串。
  2) 给目录添加digest权限  
    先登录zookeeper，进入zookeeper的bin目录，执行指令：
    ```
    ./zkCli.sh -server BOE-server-08
    ```
    完成后应该就登录到服务器设置权限了:
    ```
    [zk: BOE-server-08(CONNECTED) 0] setAcl /collections digest:username:+Ir5sN1lGJEEs8xBZhZXKvjLJ7c=:rwacd
    ```
    设置完成后可以用过下面命令查看刚才设置的权限:
    ```
    [zk: BOE-server-08(CONNECTED) 1] getAcl /collections
    'digest,'username:+Ir5sN1lGJEEs8xBZhZXKvjLJ7c=
    : cdrwa
    ```
    出现上面信息就是设置好了，接着用同样方法可以对/security.json和/configs设置权限。  
    
  3) 在catalina_zk.sh和uploadConfig.sh中添加登录信息，修改如下:
    ```
    #catalina_zk.sh
    #在之前的$JAVA_OPTS变量下面再加上:
    JAVA_OPTS="$JAVA_OPTS -DzkDigestUsername=username -DzkDigestPassword=password"   #ZK ACLS
    ```
    uploadConfig.sh更改后的文件为：
    ```
    #!/bin/sh
    configName=$1
    zkHost=BOE-server-08:2181,BOE-server-07:2181,EON-server-06:2181
    if [ -z "$configName" ]; then
      echo "配置名为空"
      echo "exit 1"
      exit 1
    fi
    configPath=$(cd `dirname $0`; pwd)"/../configsets/"$configName"/conf"
    if [ ! -d "$configPath" ]; then
      echo "配置目录\""$configPath"\"不存在"
      echo "exit 2"
      exit 2
    fi
    echo "开始上传配置,配置名:"$configName
    java -classpath .:$(cd `dirname $0`; pwd)/../tomcat8/webapps/solr/WEB-INF/lib/* \
    -DzkACLProvider=org.apache.solr.common.cloud.VMParamsAllAndReadonlyDigestZkACLProvider \
    -DzkCredentialsProvider=org.apache.solr.common.cloud.VMParamsSingleSetCredentialsDigestZkCredentialsProvider \
    -DzkDigestUsername=admin-user -DzkDigestPassword=密码 \
    org.apache.solr.cloud.ZkCLI -cmd upconfig \
    -zkhost $zkHost \
    -confdir $configPath -confname $configName
    ```
  4) 配置solr-cores下面的solr.xml,主要修改zkCredentialsProvider和zkACLProvider这两个节点
    ```
    <?xml version="1.0" encoding="UTF-8" ?>
    <solr>
    <solrcloud>
    <str name="host">${host:}</str>
    <int name="hostPort">${jetty.port:8080}</int>
    <str name="hostContext">${hostContext:solr}</str>
    <bool name="genericCoreNodeNames">${genericCoreNodeNames:true}</bool>
    <int name="zkClientTimeout">${zkClientTimeout:30000}</int>
    <int name="distribUpdateSoTimeout">${distribUpdateSoTimeout:600000}</int>
    <int name="distribUpdateConnTimeout">${distribUpdateConnTimeout:60000}</int>
    <str name="zkCredentialsProvider">org.apache.solr.common.cloud.VMParamsSingleSetCredentialsDigestZkCredentialsProvider</str>
    <str name="zkACLProvider">org.apache.solr.common.cloud.VMParamsAllAndReadonlyDigestZkACLProvider</str>
    </solrcloud>
    <shardHandlerFactory name="shardHandlerFactory"
    class="HttpShardHandlerFactory">
    <int name="socketTimeout">${socketTimeout:600000}</int>
    <int name="connTimeout">${connTimeout:60000}</int>
    </shardHandlerFactory>
    </solr>
    ```
  上面步骤完成后，启动solr服务，如果能正常在solr管理后台cloud栏目的Tree下面看到zookeeper刚才配置权限的内容，就说明配置成功了。
  3. 防火墙  
   除了上面两个访问权限控制，限制访问solr集群的ip，可以更简单粗暴的阻止不相干的人直接访问到solr。防火墙的方案很多，这个可以根据具体环境选择了。
   

# 后续
 完成上面的操作，solr的主要功能都可以正常使用了。
对于配置后一些可能遇到的坑，下面这篇博文可以参考下[《solrcloud使用中遇到的问题及解决方式》](http://www.cnblogs.com/davidwang456/p/5241429.html)。  
另外还发现一篇超详细的使用手册，内容很多，没怎么看过，有兴趣的可以看看[《SolrCloud部署和使用手册》](http://blog.csdn.net/thunder4393/article/details/44992485)。


# 参考
- [Solr6.5在Centos6上的安装与配置 (一)](http://www.cnblogs.com/wander1129/p/6658318.html)
- [Solr6.5配置中文分词IKAnalyzer和拼音分词pinyinAnalyzer (二)](http://www.cnblogs.com/wander1129/archive/2017/04/05/6658828.html)
- [SolrCloud 分布式集群安装部署（solr4.8.1 + zookeeper +tomcat）](http://blog.csdn.net/zwx19921215/article/details/38428099)
- [Solr6与Zookeeper在Tomcat环境做SolrCloud集群](http://blog.csdn.net/jiangchao858/article/details/52572635)
- [SolrCloudTomcat](https://wiki.apache.org/solr/SolrCloudTomcat)
- [http://shiyanjun.cn/archives/100.html](http://shiyanjun.cn/archives/100.html)
- [solrcloud和zookeeper的搭建、使用、心得、教训](http://www.656463.com/article/IFZjqm.htm)
- [Securing Solr](https://cwiki.apache.org/confluence/display/solr/Securing+Solr)
- [Basic Authentication Plugin](http://people.apache.org/~ctargett/RefGuidePOC/jekyll-full/basic-authentication-plugin.html#BasicAuthenticationPlugin-EnableBasicAuthentication)
- [Authentication and Authorization Plugins](https://cwiki.apache.org/confluence/display/solr/Authentication+and+Authorization+Plugins) 
- [ZooKeeper Access Control](https://cwiki.apache.org/confluence/display/solr/ZooKeeper+Access+Control)
- [solr入门之搭建具有安全控制和权限管理功能的SolrCloud集群](http://blog.csdn.net/sqh201030412/article/details/51396427)
- [Zookeeper权限管理之坑](http://www.itnose.net/detail/6512994.html)

