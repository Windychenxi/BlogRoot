---
title: ELK 专题一 （IK分词器源码升级改造实现热更新机制）
top_img: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/stars-2643089_1920.jpg
cover: https://butterfly-1316798368.cos.ap-nanjing.myqcloud.com/images/stars-2643089_1920.jpg
tag:
 - IK 分词器
 - ElasticSearch
 - ELK
categories: 
 - ELK
---

# 前言

 IK 分词器源码下载：https://github.com/medcl/elasticsearch-analysis-ik

> 本案例以 ES7.6.1 和 Mysql 数据库 5.7 为例进行配置

# 推荐阅读

- [ELK专题一 IK 分词器源码升级改造实现热更新机制](https://windychenxi.github.io/2023/02/12/ELK/IK%E5%88%86%E8%AF%8D%E5%99%A8%E6%BA%90%E7%A0%81%E5%8D%87%E7%BA%A7%E6%94%B9%E9%80%A0%E5%AE%9E%E7%8E%B0%E7%83%AD%E6%9B%B4%E6%96%B0%E6%9C%BA%E5%88%B6/)
- [ELK专题二 FileBeat 日志收集](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86/)
- [ELK专题三 LogStash 数据清洗与被压机制](https://windychenxi.github.io/2023/02/12/ELK/LogStash%20%E6%95%B0%E6%8D%AE%E6%B8%85%E6%B4%97%E4%B8%8E%E8%83%8C%E5%8E%8B%E6%9C%BA%E5%88%B6/)
- [ELK专题四 FileBeat + LogStash 整合](https://windychenxi.github.io/2023/02/12/ELK/FileBeat%20+%20LogStash%20%E6%95%B4%E5%90%88/)
- [ELK专题五 Google 浏览器插件 ElasticSeach-head 安装](https://windychenxi.github.io/2023/02/12/ELK/Google%E6%B5%8F%E8%A7%88%E5%99%A8%E6%8F%92%E4%BB%B6ElasticSearch-head%E5%AE%89%E8%A3%85/)
- [ELK专题六 ELK + FileBeat 整合](https://windychenxi.github.io/2023/02/12/ELK/ELK%20+%20FileBeat%20%E6%95%B4%E5%90%88/)
- [ELK专题七 ElasticSearch 优化](https://windychenxi.github.io/2023/02/12/ELK/ElasticSearch%20%E4%BC%98%E5%8C%96/)

### 修改源码步骤

1 修改 maven 依赖的 es 版本号

使用工具打开 IK 源码后，打开 pom.xml 文件，修改 elasticsearch 版本号为 7.6.1

```xml
<properties>
    <elasticsearch.version>7.6.1</elasticsearch.version>
    ...
</properties>
```

2 引入 MySQL 驱动到项目中

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
</dependency>
```

3 开始修改源码

在项目中找到 Dictionary 类，找到 Dictionary 单例类的初始化方法 initial 方法，在初始化方法中新启动一个线程，用来执行远程词库的热更新，在修改之前，先在 Dictionary 类同目录下新建一个类 HotDictReloadThread，代码如下：

```java
public class HotDictReloadThread {

    private static final Logger log = ESPluginLoggerFactory.getLogger(
        HotDictReloadThread.class.getName()
    );
    public void initial() {
        while(true) {
            log.info("正在调用 HotDictReloadThread...");
            Dictionary.getSingleton().reLoadMainDict();
        }
    }
}
```

上述代码含义为：获取字典单实例，并执行它的 reLoadMainDict 方法。



完成上述操作后，就开始修改 initial 方法，改动如下，创建上面新建的类并调用它的 initial 方法，从而执行 DIctionary 类的 reLoadMainDict 方法；改动代码如下，在字典实例初始化完成后新奇一个线程来执行字典的热更新操作；

```java
// 新启动一个线程用来加载数据库
pool.execute(() -> new HotDictReloadThread().initial());
```

```java
/**
 * 词典初始化 由于IK Analyzer的词典采用Dictionary类的静态方法进行词典初始化
 * 只有当Dictionary类被实际调用时，才会开始载入词典， 这将延长首次分词操作的时间 该方法提供了一
 * 个在应用加载阶段就初始化字典的手段
 * 
 * @return Dictionary
 */
public static synchronized void initial(Configuration cfg) {
    if (singleton == null) {
        synchronized (Dictionary.class) {
            if (singleton == null) {

                singleton = new Dictionary(cfg);
                singleton.loadMainDict();
                singleton.loadSurnameDict();
                singleton.loadQuantifierDict();
                singleton.loadSuffixDict();
                singleton.loadPrepDict();
                singleton.loadStopWordDict();

                // 新启动一个线程用来加载数据库
                pool.execute(() -> new HotDictReloadThread().initial());

                if(cfg.isEnableRemoteDict()){
                    // 建立监控线程
                    for (String location : singleton.getRemoteExtDictionarys()) {
                        // 10 秒是初始延迟可以修改的 60是间隔时间 单位秒
                        pool.scheduleAtFixedRate(
                            new Monitor(location), 10, 60, TimeUnit.SECONDS);
                    }
                    for (String location : singleton.getRemoteExtStopWordDictionarys()) 
                    {
                        pool.scheduleAtFixedRate(
                            new Monitor(location), 10, 60, TimeUnit.SECONDS);
                    }
                }

            }
        }
    }
}
```

在 reLoadMainDict 方法中，可以看到有 2 个方法：

- tmpDict.loadMainDict()	维护的是扩展词库
- tmpDict.loadStopWordDict()维护的是停用词库

先看对扩展词库的维护。

在方法 tmpDict.loadMainDict() 中，在最后一行加载远程自定义词库后面新增一个方法 this.loadMySQLExtDict()，用于加载 MySQL 词库，在加载 MySQL 词库之前，需要先准备下 MySQL 相关的配置以及 SQL 语句；在数据库中新建一张表，用户维护扩展词和停用词，表结构如下：

```mysql
CREATE TABLE `es_lexicon`  (
  `lexicon_id` bigint(8) NOT NULL AUTO_INCREMENT COMMENT '词库id',
  `lexicon_text` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '词条关键词',
  `lexicon_type` int(1) NOT NULL DEFAULT 0 COMMENT '0扩展词库 1停用词库',
  `lexicon_status` int(1) NOT NULL DEFAULT 0 COMMENT '词条状态 0正常 1暂停使用',
  `del_flag` int(1) NOT NULL DEFAULT 0 COMMENT '作废标志 0正常 1作废',
  `create_time` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`lexicon_id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = 'ES远程扩展词库表' ROW_FORMAT = Dynamic;
```

然后我们在项目的根路径的config目录下新建配置文件jdbc-reload.properties，内容如下

```properties
# 数据库地址
jdbc.url=jdbc:mysql://127.0.0.1:3306/test?serverTimezone=GMT&autoReconnect=true&useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useAffectedRows=true&useSSL=false
# 数据库用户名
jdbc.user=root
# 数据库密码
jdbc.password=123456
# 数据库查询扩展词库sql语句
jdbc.reload.sql=select gel.lexicon_text as word from es_lexicon gel where gel.lexicon_type = 0 and gel.lexicon_status = 0 and gel.del_flag = 0 order by gel.lexicon_id desc 
# 数据库查询停用词sql语句
jdbc.reload.stopword.sql=select gel.lexicon_text as word from ges_lexicon gel where gel.lexicon_type = 1 and gel.lexicon_status = 0 and gel.del_flag = 0 order by gel.lexicon_id desc 
# 数据库查询间隔时间 每隔10秒请求一次
jdbc.reload.interval=10
```

完成了这些基础配置之后，我们再一同看看关于同步MySql词库的方法loadMySQLExtDict()；代码较长，粘贴如下

```java
/**
 * 从MySql中加载动态词库
 */
private void loadMySQLExtDict() {
  Connection conn = null;
  Statement stmt = null;
  ResultSet rs = null;
  try {
     Path file = PathUtils.get(getDictRoot(), "jdbc-reload.properties");
     props.load(new FileInputStream(file.toFile()));

     logger.info("[==========]jdbc-reload.properties");
     for(Object key : props.keySet()) {
        logger.info("[==========]" + key + "=" + props.getProperty(String.valueOf(key)));
     }

     logger.info("[==========]query hot dict from mysql, " + props.getProperty("jdbc.reload.sql") + "......");
//       Class.forName(props.getProperty("jdbc.className"));
     conn = DriverManager.getConnection(
           props.getProperty("jdbc.url"),
           props.getProperty("jdbc.user"),
           props.getProperty("jdbc.password"));
     stmt = conn.createStatement();
     rs = stmt.executeQuery(props.getProperty("jdbc.reload.sql"));

     while(rs.next()) {
        String theWord = rs.getString("word");
        logger.info("[==========]正在加载Mysql自定义IK扩展词库词条: " + theWord);
        _MainDict.fillSegment(theWord.trim().toCharArray());
     }

     Thread.sleep(Integer.valueOf(String.valueOf(props.get("jdbc.reload.interval"))) * 1000);
  } catch (Exception e) {
     logger.error("erorr", e);
  } finally {
     if(rs != null) {
        try {
           rs.close();
        } catch (SQLException e) {
           logger.error("error", e);
        }
     }
     if(stmt != null) {
        try {
           stmt.close();
        } catch (SQLException e) {
           logger.error("error", e);
        }
     }
     if(conn != null) {
        try {
           conn.close();
        } catch (SQLException e) {
           logger.error("error", e);
        }
     }
  }
}
```

在上述代码中，通过加载配置文件，获取数据库连接，执行扩展词sql，将结果集添加到扩展词库中；



同理，同步MySql停用词的逻辑也是一样的，这里我直接把代码粘贴过来；停用词方法调用顺序为tmpDict.loadStopWordDict()，在方法后面，新增一个方法调用this.loadMySQLStopwordDict()，新方法中处理通用词逻辑，代码如下

```java
/**
* 从MySql中加载远程停用词库
*/
private void loadMySQLStopwordDict() {
  Connection conn = null;
  Statement stmt = null;
  ResultSet rs = null;

  try {
     Path file = PathUtils.get(getDictRoot(), "jdbc-reload.properties");
     props.load(new FileInputStream(file.toFile()));

     logger.info("[==========]jdbc-reload.properties");
     for(Object key : props.keySet()) {
        logger.info("[==========]" + key + "=" + props.getProperty(String.valueOf(key)));
     }
     logger.info("[==========]query hot stopword dict from mysql, " + props.getProperty("jdbc.reload.stopword.sql") + "......");
//       Class.forName(props.getProperty("jdbc.className"));
     conn = DriverManager.getConnection(
           props.getProperty("jdbc.url"),
           props.getProperty("jdbc.user"),
           props.getProperty("jdbc.password"));
     stmt = conn.createStatement();
     rs = stmt.executeQuery(props.getProperty("jdbc.reload.stopword.sql"));

     while(rs.next()) {
        String theWord = rs.getString("word");
        logger.info("[==========]正在加载Mysql自定义IK停用词库词条: " + theWord);
        _StopWords.fillSegment(theWord.trim().toCharArray());
     }
     Thread.sleep(Integer.valueOf(String.valueOf(props.get("jdbc.reload.interval"))) * 1000);
  } catch (Exception e) {
     logger.error("erorr", e);
  } finally {
     if(rs != null) {
        try {
           rs.close();
        } catch (SQLException e) {
           logger.error("error", e);
        }
     }
     if(stmt != null) {
        try {
           stmt.close();
        } catch (SQLException e) {
           logger.error("error", e);
        }
     }
     if(conn != null) {
        try {
           conn.close();
        } catch (SQLException e) {
           logger.error("error", e);
        }
     }
  }
}
```

完成这些，整体代码改造完毕；在上述代码中，有很多的地方是可以进一步优化的，比如扩展词和停用词的大量重复代码，以及读取本地配置文件项可以做到只读取一次等，这个大家可以自行优化；



完成了这些之后，我们就可以开始打包插件了；直接使用maven package命令进行打包，在target/releases/elasticsearch-analysis-ik-7.8.0.zip文件，我们将Mysql驱动mysql-connector-java.jar也放到这个压缩包里面；

**安装插件**

完成上述步骤后，拿到elasticsearch-analysis-ik-7.8.0.zip插件，我们将其放在ES安装目录下的plugins目录下，新建一个ik文件夹，将其解压到ik文件夹下；目录结构大概如下，记得要有MySql驱动mysql-connector-java.jar

![img](https://pic1.zhimg.com/80/v2-73f17eaa457c573eaff75a112faca3b8_720w.webp)



完成上述步骤后，我们就可以启动ES了，在启动过程中，可以看到关于IK热更新MySql词库相关的日志输出；在实际过程中，可能会报很多的异常，下面是我所遇到的一些问题以及解决方案；







**常见问题**

**1、异常1**：java.sql.SQLException: Column 'word' not found.

此异常是因为编写sql时，查询的数据库字段需要起别名为 word，修改一下sql即可解决这个问题；



**2、异常2**：Could not create connection to database server

此异常通常是因为引用的mysql驱动和mysql版本号不一致导致的，只需要替换成对应的版本号即可解决，另外，数据库连接我们不需要再额外的去配置显示加载，即不需要写 Class.forName(props.getProperty("jdbc.className"));



**3、异常3**：no suitable driver found for jdbc:mysql://...

此异常我们需要在环境的JDK安装目录的jre\lib\ext目录下添加mysql驱动mysql-connector-java.jar；比如我本地的是C:\Java\jdk_8u_231\jre\lib\ext 目录，服务器上是/data/jdk1.8.0_181/jre/lib/ext



**4、异常4**：AccessControlException: access denied ("java.net.SocketPermission" "127.0.0.1:3306" "connect,resolve")

这个异常，我们修改jdk安装路径下的C:\Java\jdk_8u_231\jre\lib\security目录下的文件**java.policy**，在下面新增一行即可解决

```text
permission java.net.SocketPermission "*", "connect,resolve";
```



![img](https://pic2.zhimg.com/80/v2-67909acb527460a226b09072e3f6d3bd_720w.webp)

