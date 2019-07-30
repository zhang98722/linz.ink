---
layout: post
title:  一次失败的shardingSphere实践
date:   2019-04-26
categories: 
  - shardingSphere
  - 分表
comments: false
---

> 最近参与了一个股市量化项目，其中我主要负责其中的行情数据的接入，简单来说，就是把交易中心的数据拉到本地，然后进行数据分析和统计。其中一个点就是数据持久化的选型，最终从开发、运维、性能等方面将选型定为mysql+分表，虽然之前做了很多次分库分表，均是通过自己硬编码编写分库分表策略，所以这次想尝试一下shardingSphere。

<!-- more -->

## 背景

在选型之前我们做了一次容量预估，大概内容如下：

```
参考内存，我们目前只持久化逐笔交易的分钟线数据和快照数据，每日的数据量：

- 逐笔交易：60*24*3000 =4.32M 条
- 快照数据：20*60*24*3000=8.64M 条

参考网络的基本数据结构,估测每天的存储空间大概在:

4.32M*400+8.65M*50≈2.5G
考虑到索引及其他占用，通过压缩之后，应该在2G~5G存储空间
```

在最初期望用大数据来存储，例如MangoDb、ES、HBase等，不过最后还是因为性能、运维成本等因素，选择了mysql。具体通过股票或者期货编码进行分表，细分下来单表年数据量在千万上下，刚好接近mysql单表性能瓶颈，再通过归档，也能符合mysql的性能优化限制。ORM使用了常规的mybatis，也能和shardingSphere完美整合。

## 实践

shardingSphere的分表其实蛮简单的，按照官方文档，非常容易就能实现我们期望的分表策略：

```
@Bean
    @ConfigurationProperties(prefix = "spring.datasource.main")
    public HikariConfig hikariConfig(){
        return new HikariConfig();
    }

    @Bean
    public DataSource hikariDataSource(){
        return new HikariDataSource(hikariConfig());
    }

    @Bean
    public TableRuleConfiguration barTableRuleConfig(){
        TableRuleConfiguration tableRuleConfig = new TableRuleConfiguration("stock_bar_stockcode_granularity");
        tableRuleConfig.setCheckActualDataNode(false);
        tableRuleConfig.setDatabaseShardingStrategyConfig(new NoneShardingStrategyConfiguration());
        tableRuleConfig.setTableShardingStrategyConfig(new ComplexShardingStrategyConfiguration("stock_code,granularity", ((availableTargetNames, shardingValue) -> {
            ...
            return Arrays.asList(...);
        })));
        return tableRuleConfig;
    }

    @Bean
    public TableRuleConfiguration snapshotRuleConfig(){
        TableRuleConfiguration tableRuleConfig = new TableRuleConfiguration("tick_stockcode");
        tableRuleConfig.setCheckActualDataNode(false);
        tableRuleConfig.setDatabaseShardingStrategyConfig(new NoneShardingStrategyConfiguration());
        tableRuleConfig.setTableShardingStrategyConfig(new InlineShardingStrategyConfiguration("stock_code", "tick_${stock_code}"));
        return tableRuleConfig;
    }

    @Bean
    @Primary
    public DataSource shardingDatasource(TableRuleConfiguration[] tableRuleConfigurations) throws SQLException {
        Map<String, DataSource> dataSourceMap = new HashMap<>();
        dataSourceMap.put("main", hikariDataSource());

        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
        shardingRuleConfig.getTableRuleConfigs().addAll(Arrays.asList(tableRuleConfigurations));

        Properties properties = new Properties();

        properties.put(ShardingPropertiesConstant.CHECK_TABLE_METADATA_ENABLED, false);

        return ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig, properties);
    }

```

基本按照官方文档就可以很容易的配置出支持一个分库分表的DataSource，这里可以看到多了一个官方文档没有的配置项：

```
properties.put(ShardingPropertiesConstant.CHECK_TABLE_METADATA_ENABLED, false);
```

这个的意思是不加载数据库表的metaData，什么意思呢？

原来shardingSphere为了支持一些sql，最简单的例如分页（官方文档也有阐述)，shardingSphere有两个概念：

 - logic table ：抽象出来的虚拟表，即n张子表逻辑上抽象出来的表，例如以字表order_1, order_2合并出来的order表
 - actual table ：实际的子表
 
 以分页为例，例如用从order表按照1000条数据拿第三页，shardingSphere是先确定在哪些order的字表执行完全一样的原始sql，分别拿1000条，然后把各个字表拿到的各自的1000条数据放到一起，再进行排序，拿前1000条。所以shardingSphere会提前加载子表的表结构，进行预先的验证和分析。具体的业务逻辑参考：
 
 > org.apache.shardingsphere.core.execute.metadata.TableMetaDataInitializer#load(org.apache.shardingsphere.core.rule.ShardingRule)

而在开发过程中，我们也是发现启动的时候加载时间越来越长，最后才定位到全是卡在这个loadMetaData上，在github上查了一遍issue，发现已经有人已经在处理了，不过没有具体的实现，也通过修改源码返回了一个空的meta来跳过，测试也没有什么问题。

同时在阅读加载metaData的过程中发现，在shardingSphere初始化的过程中，至少需要明确数据库表存在的信息，即在初始化的过程中，会构建logic table和actual table的映射关系，而如果子表信息在运行时的过程中发生了变化，shardingSphere感知不到的，具体参考代码：

 > org.apache.shardingsphere.core.execute.metadata.TableMetaDataLoader#isTableExist

而我们的数据库分表在运行时的时候，我们是需要动态增加新的字表的，这导致shardingSphere完全不可用。

## 最后的解决方案

在放弃shardingSphere来实现分表之后，仍然觉得以前的硬编码分表策略比较蠢，既然不能在DataSource这一层进行分表，于是又去尝试在orm，即mybatis这一层，在sql代码的生成这个阶段去解决分表的问题。

虽然mybatis提供interceptor甚至plugin的模式来支持扩展，然而在具体的实现才发现这又是一个深坑，最大的问题就是如何预编译的sql（statement/preparedStatement）中获取分表策略的参数，例如逻辑表名和分表策略需要的字段值，mybatis并没有做sql解析，仅仅是简单的按照顺序进行适配，而如果自己要拿到想要的值，则需要进行sql的解析，而这是一个比分表大n倍的天坑啊...

而这个时候突然想到shardingSphere就是通过sql解析来获取相关的值信息的啊，能不能只使用shardingSphere这部分sql解析的内容，然后嫁接到mybatis呢？这样就能够完美适配完美的应用场景，在一顿操作之后，发现这个成本巨高，基本接近不可能...毕竟shardingSphere有超过一半的代码都是在进行sql解析...

最后，一切又回到了原点，原来还是以前最简单最直接最暴力的硬编码数据库表的路由策略才是最简单的...

## 总结

shardingSphere是个很好的分库分表工具，极大的简化了大部分场景下的分库分表实现，但是它也不是一个万能的工具，在使用之前还是需要先检查一下自己的应用场景：

 - 数据库表有没有运行时的更新，否则在一些语句的执行过程中，可能会落不到期望的字表中
 - 表的数量尽可能的少，在执行一些复杂的sql的时候，以分页查询为例，如果有1000张表，每个表也许仅仅只获取1000条数据，汇总在一起也有100w数据，可能运行时就OOM了
 