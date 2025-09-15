---
{"dg-publish":true,"permalink":"/Program/Mixed/哈罗面试：没开sql慢查询日志，怎么发现慢 sql？/","noteIcon":"","created":"2025-09-10T10:14:13.643+08:00"}
---

## 一：什么是 慢 SQL（慢查询） 日志？

MySQL的慢查询日志是MySQL提供的一种日志记录。

MySQL的慢查询日志 ，主要 用来记录在MySQL中响应的时间超过 **执行时长阈值**的语句， **执行时长阈值** 通过 一个 参数 long\_query\_time（默认是10秒）配置。

一个SQL的执行，只要 超过 这个 long\_query\_time 时长， 就会被判为慢查询， 会被记录到慢查询日志中。

## 二： SQL 到底 执行 多长时间，才为慢SQL ？

慢 SQL 时间 长度是   “执行耗时超出业务或数据库的合理阈值”，但具体定义需结合**数据库类型、业务场景、硬件配置**综合判断，无绝对统一标准。

慢 SQL 时间 长度 通常分为 通用阈值 和 业务定制阈值 两类：

### 第一个维度：通用参考阈值（以 MySQL 为例）

数据库层面默认有基础标准，最典型的是 MySQL 的`long_query_time`参数（慢查询日志的核心触发条件）：

**默认值：** **10 秒 （即执行时间≥10 秒的 SQL 会被标记为慢 SQL）；**

**实际场景：** 多数业务会将阈值下调至**1-2 秒**（因 10 秒远超用户可接受的响应时间，比如 Web 页面加载通常要求 < 2 秒，接口响应 < 500ms）。

### 第二个维度：业务定制阈值（更关键）

通用阈值仅作参考，真正的慢 SQL 定义需贴合业务场景，例如：

*   高频核心业务（如电商下单、支付接口）：阈值可能设为**500ms**（若 SQL 执行 1 秒，会直接导致用户卡顿）；
*   低频非核心任务（如每日数据统计）：阈值可放宽至**1 秒 - 20s**（后台执行，不影响用户体验）；
    

## 三：如何设置 慢SQL 相关参数？

下面是一些关键参数设置：

*   slow\_query\_log：控制慢查询日志的开关。

设置成 1，日志就开启；设置成 0，日志就关闭。
*   log-slow-queries（旧版本 5.6 以下使用）和slow-query-log-file（5.6 及以上版本使用）：
    

这两个参数用来指定慢查询日志的保存位置。如果不设置，系统会自动生成一个默认文件，名字是 “主机名 - slow.log”。

*   long\_query\_time：慢查询阈值：
    

它决定了什么样的查询算慢查询，只要 SQL 语句执行时间超过这里设置的时长，就会被记录到日志里。

*   log\_queries\_not\_using\_indexes：
    

这是个可选参数，打开后，**没命中索引的查询语句**也会被记录下来。

*   log\_output：用来选择日志保存方式。

有 FILE（快） , TABLE（满） 两种方式保持日志。设置成FILE，日志就存到文件里，这是默认方式；

设置成  TABLE，日志会存到数据库的mysql.slow\_log表里。

也可以同时用两种方式，写成FILE,TABLE。不过，存到数据库表比存到文件更费资源，如果既想开慢查询日志，又想数据库跑得快，建议优先把日志存到文件里。

slow\_query\_log 的 参数查看：

```bash  
  
mysql> show VARIABLES like '%slow_query_log%';  
+---------------------+-----------------------------------------------------------+  
| Variable_name       | Value                                                     |  
+---------------------+-----------------------------------------------------------+  
| slow_query_log      | OFF                                                       |  
| slow_query_log_file | d:\java\mysql-5.7.28-winx64\data\DEEP-2020AEBKQR-slow.log |  
+---------------------+-----------------------------------------------------------+  
2 rows in set, 1 warning (0.00 sec)  
  
```

long\_query\_time 的 参数查看：

```bash
mysql> show VARIABLES like 'long_query_time%';  
+-----------------+-----------+  
| Variable_name   | Value     |  
+-----------------+-----------+  
| long_query_time | 10.000000 |  
+-----------------+-----------+  
1 row in set, 1 warning (0.00 sec)  
```

设置启用慢查询日志：

```bash
  
SET GLOBAL slow_query_log = 'ON';  
SET GLOBAL long_query_time = 1;  
```  

注意：修改全局参数后，当前连接会话仍使用旧值，需新建会话才会生效，需要重新登录

查看修改后参数：

```bash
  
mysql> show VARIABLES like '%slow_query_log%';  
+---------------------+-----------------------------------------------------------+  
| Variable_name       | Value                                                     |  
+---------------------+-----------------------------------------------------------+  
| slow_query_log      | ON                                                        |  
| slow_query_log_file | d:\java\mysql-5.7.28-winx64\data\DEEP-2020AEBKQR-slow.log |  
+---------------------+-----------------------------------------------------------+  
2 rows in set, 1 warning (0.00 sec)  
mysql> show VARIABLES like 'long_query_time%';  
+-----------------+----------+  
| Variable_name   | Value    |  
+-----------------+----------+  
| long_query_time | 1.000000 |  
+-----------------+----------+  
1 row in set, 1 warning (0.00 sec)  
  
```

## 四：3大 慢 SQL 分析工具

慢查询分析一般分为三个步骤

1、 EXPLAIN2、 OPTIMIZER\_TRACE3、 PROFILE
上面三大工具的 具体使用方法，请参见尼恩之前的博客文章：

[凌晨2点，报警群炸了：一条sql 执行200秒！搞定之后，我总结了一个慢SQL查询、定位分析解决的完整套路](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247505363&idx=1&sn=ada0c821646c7d41eb4059e14fbaa92f&scene=21#wechat_redirect)

## 五、没开慢查询日志，怎么发现慢 SQL？ （4个方法）

慢查询日志是 “事后追溯” 的核心工具，但未开启时，可通过「实时监控」「资源排查」「业务反馈」三类手段主动发现慢 SQL，覆盖**实时问题定位**和**历史问题追溯**场景。

### 5.1 抓现行： 实时监控面板

首先借助 直观的实时监控， 借助可视化工具 ，监控慢sql：

*   **开源工具**：Prometheus + Grafana（配置数据库监控模板，可实时查看 “慢查询数”“SQL 执行耗时 TOP10”） ；
*   **云厂商工具**：阿里云 RDS 的「实时性能监控」、腾讯云 CDB 的「性能洞察」，可直接查看当前执行的慢 SQL 列表及资源占用。
    

### 5.2  抓现行： 实时 DB 定位

适用于 “业务反馈卡顿”“数据库 CPU/IO 飙升” 等紧急场景，直接查看数据库当前运行的 SQL，定位耗时久的语句。

用到命令：

*   `show processlist`
*   `` `information_schema.processlist``

这是最常用的实时排查命令，能查看当前所有数据库连接的执行状态、耗时、SQL 内容。

*   **基础命令**：`show processlist;`（仅显示前 100 条，适合快速查看）；
*   **完整命令**：`select * from information_schema.processlist where command != 'Sleep';`（显示所有非空闲连接，便于筛选）。

**关键字段解读**（重点关注 3 个字段）：

| **字段名** | **含义**                                                |     |
| ------- | ----------------------------------------------------- | --- |
| `Time`  | SQL 已执行的时间（单位：秒），若远超阈值（如 > 3 秒），大概率是慢 SQL             |     |
| `State` | SQL 当前的执行状态，如`Sending data`（数据传输，耗时可能长）、`Locked`（锁等待） |     |
| `Info`  | 完整的 SQL 语句（若过长，需用`show full processlist;`查看完整内容）      |     |

**示例**：

若发现`Time=15`（执行了 15 秒）、`State=Sending data`、`Info=select * from order where create_time < '2024-01-01'`，则该 SQL 是典型的慢 SQL。

### 5.3.  日志与链路追踪：从 “业务卡顿” 定位 SQL

业务层的异常反馈是慢 SQL 的重要线索，通过业务日志和链路追踪工具可反向找到对应的 SQL：

1. **步骤 1**：收集业务反馈（如 “用户提交订单时卡顿了 5 秒”），确定异常时间（如 2024-05-20 14:30）和业务接口（如`/order/submit`）；
2. **步骤 2**：查看应用日志（如 Java 的 Logback、Python 的 logging），找到该接口对应的 SQL 执行记录（若应用打印了 SQL 耗时，直接定位）；
3. **步骤 3**：若应用未打印 SQL，用链路追踪工具（如 SkyWalking ）查看接口调用链，找到 “数据库调用” 节点，查看耗时（如 “MySQL 调用耗时 4.5 秒”），并提取对应的 SQL 语句。

### 5.4 历史追溯： 查历史执行记录 发现 慢 SQL

若慢 SQL 已执行完毕（如夜间批量任务），可以 通过 “资源使用记录”“业务日志” 间接追溯。

#### 5.4 .1.  分析 历史执行记录表中 的 “执行统计信息”

部分数据库会留存近期的 SQL 执行统计信息，可通过系统表查询：

```bash
#（默认保留 1000 条历史 SQL，可通过setup\_consumers调整保留数）：
performance_schema.events_statements_history  
-- 筛选执行时间>3秒的历史SQL（单位：微秒，需转换为秒）  
select sql_text, timer_wait/1000000000 as exec_time   
from performance_schema.events_statements_history   
where timer_wait/1000000000 > 3   
order by exec_time desc;  

```



#### 5.4 .2. 分析  历史执行记录  “全表扫描SQL ”

还可主动扫描数据库中 “可能成为慢 SQL” 的语句（如无索引、全表扫描的 SQL）。

比如，全表扫描（`type=ALL`）是慢 SQL 的主要诱因，可通过`explain`分析 SQL 执行计划，或直接查询系统表筛选：

```sql  
-- 筛选出全表扫描且行数>1万的SQL（潜在慢SQL）  
select sql_text, rows_examined   
from performance_schema.events_statements_history   
where rows_examined > 10000   
  and (sql_text like 'select%' or sql_text like 'update%' or sql_text like 'delete%')  
order by rows_examined desc;  
```

其中：

*   `rows_examined`：SQL 扫描的行数，若远大于实际返回行数（`rows_sent`），说明存在低效扫描。