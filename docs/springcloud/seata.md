### Seata分布式事务

------

##### 原理

- 角色：TC seata、TM 全局事务的发起者(即注解了@@GlobalTrancactional的微服务)、RM 微服务。
- 模式：
  - AT(默认)：
  - TCC：
  - SAGA：
  - XA事务：需要数据库支持，两段式提交。
- 

##### 使用

1. 修改配置文件file.conf：

   ~~~xml
   修改service中分组名称tx-service-group="aaa"；修改store中存储模式为db，并设置数据库连接信息。
   ~~~

2. 创建数据库表

   ~~~
   seata数据库使用db_store.sql脚本，产生表branch_table、global_table、look_table
   业务数据库使用db_undo_log.sql脚本，产生表undo_log
   ~~~

3. 修改配置文件registry.conf:

   ~~~
   修改registry节点的的注册方式为nacos，并配置nacos的serverAddr、namespace、cluster。
   ~~~

4. 先启动nacos、再启动seata TC

5. 各个微服务seata相关设置RM

   ~~~java
   1.在application.yml文件中配置spring.cloud.alibaba.seata.tx-service-group=aaa //第一步配置的
   2.新建file.conf、registry文件(拷贝seata中的)
   3.增加配置文件DataSourceConfig、XXXXConfiguration，在启动注解中排除@SpringBootApplication(exclude = DataSourceAutoConfiguration.class) //即把数据库操作交给seata代理；
   ~~~

6. 全局事务TM

   ~~~java
   TM即全局事务的发起方，在其方法上注解@GlobalTrancactional(rollbackFor=Exception.class)
   ~~~