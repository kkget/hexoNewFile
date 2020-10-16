---
title: 数据库中间件系列[Mycat+Sharding-Sphere]
date: 2020-09-30 10:38:32
tags: 中间件
---

数据库链接池过多，或者导致数据库挂了，如果主备机切换，需要更改配置
java应用与数据库紧耦合
![图1](http://47.105.116.133:8080/upload/dbcenter/1.png)
官网:http://www.mycat.io/
![图1](http://47.105.116.133:8080/upload/dbcenter/2.png)
主从复制，主备切换，双主双从
![图1](http://47.105.116.133:8080/upload/dbcenter/3.png)
![图1](http://47.105.116.133:8080/upload/dbcenter/4.png)
Mycat原理:拦截
安装
Springboot整合
```java
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.0.0</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```
application.yml配置文件：
#设置应用端口
```java
server:
  port: 8080

spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    # Mycat连接
    url: jdbc:mysql://localhost:8066/springboot?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&useSSL=false&serverTimezone=GMT%2B8
    username: root
    password: 123456

# MyBatis
mybatis:
    type-aliases-package: com.example.springbootmybatismycat.domain
    mapper-locations: classpath:/mybatis/*.xml
    #sql打印配置
    configuration:
        log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```
一、基本概念
Sharding-Sphere简介
Sharding-Sphere官网：http://shardingsphere.apache.org/index_zh.html
![图1](http://47.105.116.133:8080/upload/dbcenter/5.png)
一套开源的分布式数据库中间件解决方案。
有三个产品，Sharding-JDBC、Sharding-Proxy 和 Sharding-Sidecar。
ShardingSphere 定位为关系型数据库中间件，合理地在分布式环境下使用关系型数据库操作。
分库分表
数据据库数据量是不可控的，随着时间和业务发展，造成表里面数据越来越多，如果再去对数据库表curd操作时，就会有性能问题。
解决办法：
方案1：从硬件上
方案2：分库分表
为了解决由于数据量过大而造成数据库性能降低问题。
![图1](http://47.105.116.133:8080/upload/dbcenter/6.png)
分库分表的方式
分库分表有两种方式：垂直切分和水平切分。
垂直切分：垂直分表和垂直分库
水平切分：水平分表和水平分库
垂直拆字段，水平拆记录。
垂直切分的库和表结构是不同的，而水平切分的库和表是相同的。
垂直分表
操作数据库中某张表，把这张表中一部分字段数据存到一张新表里面，再把这张表另部分字段数据存到另外一张表里面。
垂直拆分成的两张表，每张表的数据量和原先的表的数据量相同。
![图1](http://47.105.116.133:8080/upload/dbcenter/7.png)
![图1](http://47.105.116.133:8080/upload/dbcenter/8.png)
垂直分库
把单一数据库按照业务进行划分，专库专表。
![图1](http://47.105.116.133:8080/upload/dbcenter/9.png)
水平分库
![图1](http://47.105.116.133:8080/upload/dbcenter/10.png)
水平分表
![图1](http://47.105.116.133:8080/upload/dbcenter/12.png)
分库分表的应用和问题
应用
在数据库设计时候就要考虑垂直分库和垂直分表。
随着数据库数据量增加，不要马上考虑做水平切分。首先考虑缓存处理，读写分离，使用索引等等方式。如果这些方式不能根本解决问题了，再考虑做水平分库和水平分表。
分库分表问题
跨节点连接查询问题(分页、排序)。
多数据源管理问题。
二、Sharding-JDBC 分库分表操作
ShardingSphere-JDBC 简介
Sharding-JDBC是轻量级的Java框架，是增强的JDBC驱动。
Sharding-JDBC的功能：主要做数据分片和读写分离，不是做分库分表，分库分表由我们自己做。
主要的目的：简化分库分表后数据的相关操作。
Sharding-JDBC 实现水平分表
搭建环境
技术：SpringBoot2.2.1 + MybatisPlus + Sharding-JDBC + Druid连接池
创建项目

修改SpringBoot的版本
```java
pom.xml引入相关的依赖
	<dependency>
	    <groupId>com.alibaba</groupId>
	    <artifactId>druid-spring-boot-starter</artifactId>
	    <version>1.1.20</version>
	</dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.18</version>
    </dependency>
    
    <dependency>
        <groupId>org.apache.shardingsphere</groupId>
        <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
        <version>4.0.0-RC1</version>
    </dependency>
    
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.0.5</version>
    </dependency>
    
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
```
按照水平分表的方式创建数据库、数据表
创建数据库 course_db。
在数据库中创建两张表 course_1 和 course_2。
数据存放约定规则：添加的数据id为偶数放 course_1 表中，id为奇数放 course_2 表中。
```java
create database course_db;

use course_db;

create table course_1 (
	`cid` bigint(20) primary key,
	`cname` varchar(50) not null,
	`user_id` bigint(20) not null,
	`cstatus` varchar(10) not null
);

create table course_2 (
	`cid` bigint(20) primary key,
	`cname` varchar(50) not null,
	`user_id` bigint(20) not null,
	`cstatus` varchar(10) not null
);
```
编写代码实现对分库分表数据的操作
在shardingjdbcdemo包下创建 entity.Course 实体类
```java
@Data
public class Course {
    private Long cid;
    private String cname;
    private Long userId;
    private String cstatus;
}


创建mapper.CourseMapper 接口
@Repository
public interface CourseMapper extends BaseMapper<Course> {}

```
在启动类上加上@MapperScan("com.angenin.shardingjdbcdemo.mapper")注解
配置水平分表策略
官网数据分片配置：https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-jdbc/usage/sharding/spring-boot-starter/
```java
在 application.properties 配置文件中配置：（下面数据库密码记得写）
# shardingjdbc 水平分表策略
# 配置数据源，给数据源起别名
spring.shardingsphere.datasource.names=m1

# 一个实体类对应两张表，覆盖
spring.main.allow-bean-definition-overriding=true

# 配置数据源的具体内容，包含连接池，驱动，地址，用户名，密码
spring.shardingsphere.datasource.m1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.m1.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.m1.url=jdbc:mysql://localhost:3306/course_db?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.m1.username=root
spring.shardingsphere.datasource.m1.password=

# 指定course表分布的情况，配置表在哪个数据库里，表的名称都是什么 m1.course_1,m1.course_2
spring.shardingsphere.sharding.tables.course.actual-data-nodes=m1.course_$->{1..2}

# 指定 course 表里面主键 cid 的生成策略 SNOWFLAKE
spring.shardingsphere.sharding.tables.course.key-generator.column=cid
spring.shardingsphere.sharding.tables.course.key-generator.type=SNOWFLAKE

# 配置分表策略    约定 cid 值偶数添加到 course_1 表，如果 cid 是奇数添加到 course_2 表
spring.shardingsphere.sharding.tables.course.table-strategy.inline.shardingcolumn=cid
spring.shardingsphere.sharding.tables.course.table-strategy.inline.algorithmexpression=course_$->{cid % 2 + 1}

# 打开 sql 输出日志
spring.shardingsphere.props.sql.show=true
```

编写测试代码
在test包中的测试类中进行测试
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ShardingjdbcdemoApplicationTests {

    @Autowired
    private CourseMapper courseMapper;
    
    //添加课程
    @Test
    public void addCourse(){
        Course course = new Course();
        //cid由我们设置的策略，雪花算法进行生成（至少70年内生成的id不会重复）
        course.setCname("java");
        course.setUserId(100L);
        course.setCstatus("Normal");
    
        courseMapper.insert(course);
    }
    
    //查询课程
    @Test
    public void findCourse(){
        QueryWrapper<Course> wrapper = new QueryWrapper<>();
        wrapper.eq("cid", 509755853058867201L);
        courseMapper.selectOne(wrapper);
    }
}

```
addCourse方法执行结果：

findCourse方法执行结果：

Sharding-JDBC 实现水平分库
需求分析

创建数据库，数据表
```java
create database edu_db_1;
create database edu_db_2;

use edu_db_1;

create table course_1 (
	`cid` bigint(20) primary key,
	`cname` varchar(50) not null,
	`user_id` bigint(20) not null,
	`cstatus` varchar(10) not null
);

create table course_2 (
	`cid` bigint(20) primary key,
	`cname` varchar(50) not null,
	`user_id` bigint(20) not null,
	`cstatus` varchar(10) not null
);

use edu_db_2;

create table course_1 (
	`cid` bigint(20) primary key,
	`cname` varchar(50) not null,
	`user_id` bigint(20) not null,
	`cstatus` varchar(10) not null
);

create table course_2 (
	`cid` bigint(20) primary key,
	`cname` varchar(50) not null,
	`user_id` bigint(20) not null,
	`cstatus` varchar(10) not null
);

```

配置水平分库策略
下面数据库的密码填自己的。
```java
# shardingjdbc 水平分库分表策略
# 配置数据源，给数据源起别名
# 水平分库需要配置多个数据库
spring.shardingsphere.datasource.names=m1,m2

# 一个实体类对应两张表，覆盖
spring.main.allow-bean-definition-overriding=true

# 配置第一个数据源的具体内容，包含连接池，驱动，地址，用户名，密码
spring.shardingsphere.datasource.m1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.m1.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.m1.url=jdbc:mysql://localhost:3306/edu_db_1?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.m1.username=root
spring.shardingsphere.datasource.m1.password=

# 配置第二个数据源的具体内容，包含连接池，驱动，地址，用户名，密码
spring.shardingsphere.datasource.m2.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.m2.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.m2.url=jdbc:mysql://localhost:3306/edu_db_2?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.m2.username=root
spring.shardingsphere.datasource.m2.password=

# 指定数据库分布的情况和数据表分布的情况
# m1 m2   course_1 course_2
spring.shardingsphere.sharding.tables.course.actual-data-nodes=m$->{1..2}.course_$->{1..2}

# 指定 course 表里面主键 cid 的生成策略 SNOWFLAKE
spring.shardingsphere.sharding.tables.course.key-generator.column=cid
spring.shardingsphere.sharding.tables.course.key-generator.type=SNOWFLAKE

# 指定分库策略    约定 user_id 值偶数添加到 m1 库，如果 user_id 是奇数添加到 m2 库
# 默认写法（所有的表的user_id）
#spring.shardingsphere.sharding.default-database-strategy.inline.sharding-column=user_id
#spring.shardingsphere.sharding.default-database-strategy.inline.algorithm-expression=m$->{user_id % 2 + 1}
# 指定只有course表的user_id
spring.shardingsphere.sharding.tables.course.database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.course.database-strategy.inline.algorithm-expression=m$->{user_id % 2 + 1}

# 指定分表策略    约定 cid 值偶数添加到 course_1 表，如果 cid 是奇数添加到 course_2 表
spring.shardingsphere.sharding.tables.course.table-strategy.inline.sharding-column=cid
spring.shardingsphere.sharding.tables.course.table-strategy.inline.algorithm-expression=course_$->{cid % 2 + 1}

# 打开 sql 输出日志
spring.shardingsphere.props.sql.show=true



编写测试代码
	//添加课程
    @Test
    public void addCourseDb(){
        Course course = new Course();
        //cid由我们设置的策略，雪花算法进行生成（至少70年内生成的id不会重复）
        course.setCname("javademo");
        //分库根据user_id
        course.setUserId(100L);
        course.setCstatus("Normal");
        courseMapper.insert(course);

        course.setCname("javademo2");
        course.setUserId(111L);
        courseMapper.insert(course);
    }
    
    //查询课程
    @Test
    public void findCourseDb(){
        QueryWrapper<Course> wrapper = new QueryWrapper<>();
        //设置user_id的值
        wrapper.eq("user_id", 100L);
        //设置cid的值
        wrapper.eq("cid", 509771111076986881L);
        Course course = courseMapper.selectOne(wrapper);
        System.out.println(course);
    }

```
addCourseDb方法执行结果：








findCourseDb方法执行结果：

Sharding-JDBC 实现垂直分库
需求分析


需要查询用户信息的时候，不需要查到课程信息。
创建用户数据库、数据表
```java
create database user_db;

use user_db;

create table t_user(
	`user_id` bigint(20) primary key,
	`username` varchar(100) not null,
	`ustatus` varchar(50) not null
);

编写User代码
创建user实体类和对应的mapper
@Data
@TableName("t_user")    //指定对应的表名
public class User {
    private Long userId;
    private String username;
    private String ustatus;
}



@Repository
public interface UserMapper extends BaseMapper<User> {}
```

配置垂直分库策略
记得写数据库密码。
```java
# shardingjdbc 垂直分库策略
# 配置数据源，给数据源起别名
# 水平分库需要配置多个数据库
# m0为用户数据库
spring.shardingsphere.datasource.names=m1,m2,m0

# 一个实体类对应两张表，覆盖
spring.main.allow-bean-definition-overriding=true

# 配置第一个数据源的具体内容，包含连接池，驱动，地址，用户名，密码
spring.shardingsphere.datasource.m1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.m1.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.m1.url=jdbc:mysql://localhost:3306/edu_db_1?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.m1.username=root
spring.shardingsphere.datasource.m1.password=

# 配置第二个数据源的具体内容，包含连接池，驱动，地址，用户名，密码
spring.shardingsphere.datasource.m2.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.m2.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.m2.url=jdbc:mysql://localhost:3306/edu_db_2?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.m2.username=root
spring.shardingsphere.datasource.m2.password=

# 配置user数据源的具体内容，包含连接池，驱动，地址，用户名，密码
spring.shardingsphere.datasource.m0.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.m0.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.m0.url=jdbc:mysql://localhost:3306/user_db?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.m0.username=root
spring.shardingsphere.datasource.m0.password=
# 配置user_db数据库里面t_user  专库专表
spring.shardingsphere.sharding.tables.t_user.actual-data-nodes=m0.t_user
# 配置主键的生成策略
spring.shardingsphere.sharding.tables.t_user.key-generator.column=user_id
spring.shardingsphere.sharding.tables.t_user.key-generator.type=SNOWFLAKE
# 指定分表策略
spring.shardingsphere.sharding.tables.t_user.table-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_user.table-strategy.inline.algorithm-expression=t_user

# ... 其他配置同上

```

编写测试代码
```java
    //添加用户
    @Test
    public void addUserDb(){
        User user = new User();
        user.setUsername("张三");
        user.setUstatus("a");
        userMapper.insert(user);
    }

    @Test
    //查询用户
    public void findUserDb(){
        QueryWrapper<User> wrapper = new QueryWrapper<>();
        wrapper.eq("user_id", 509793334663839745L);
        User user = userMapper.selectOne(wrapper);
        System.out.println(user);
    }

```
addUserDb方法执行结果：




findUserDb方法执行结果：

Sharding-JDBC 公共表
公共表概念
存储固定数据的表，表数据很少发生变化，查询时经常要进行关联。
在每个数据库中都创建出相同结构公共表。
操作公共表时，同时操作添加了公共表的数据库中的公共表，添加记录时，同时添加，删除时，同时删除。
在多个数据库中创建相同结构的公共表
```java
use user_db;
#use edu_db_1;
#use edu_db_2;

create table t_udict(
	`dictid` bigint(20) primary key,
	`ustatus` varchar(100) not null,
	`uvalue` varchar(100) not null
);



公共表配置
# 其他配置同上

# ...

# 公共表配置
spring.shardingsphere.sharding.broadcast-tables=t_udict
# 配置主键的生成策略
spring.shardingsphere.sharding.tables.t_udict.key-generator.column=dictid
spring.shardingsphere.sharding.tables.t_udict.key-generator.type=SNOWFLAKE

# ...


编写公共表的实体类及mapper
@Data
@TableName(value = "t_udict")
public class Udict {
    private Long dictid;
    private String ustatus;
    private String uvalue;
}


@Repository
public interface UdictMapper extends BaseMapper<Udict> {}


编写测试代码
    //添加
    @Test
    public void addDict(){
        Udict udict = new Udict();
        udict.setUstatus("a");
        udict.setUvalue("已启用");
        udictMapper.insert(udict);
    }

    //删除
    @Test
    public void deleteDict(){
        QueryWrapper<Udict> wrapper = new QueryWrapper<>();
        wrapper.eq("dictid", 509811689974136833L);
        udictMapper.delete(wrapper);
    }


addDict方法执行结果：




deleteDict方法执行结果：
```


Sharding-JDBC 实现读写分离
读写分离概念



读写原理



Sharding-JDBC 读写分离


Sharding-JDBC通过sql语句语义分析，当sql语句有insert、update、delete时，Sharding-JDBC就把这次操作在主数据库上执行；当sql语句有select时，就会把这次操作在从数据库上执行，从而实现读写分离过程。但Sharding-JDBC并不会做数据同步，数据同步是配置MySQL后由MySQL自己完成的。
MySQL 一主一从读写分离配置
我采用的是docker来实现的，单独写在这篇文章里：https://blog.csdn.net/qq_36903261/article/details/108457759
使用docker后，需要在主服务器上新建原先的数据库数据表。
因为记录的文件名以及位点每次重启或刷新都会改变，所以以下命令放在这里，方便查看。
主mysql：
#确认位点 记录下文件名以及位点（重启或者刷新都会改变）
show master status;

1
2
从mysql：
#先停止同步
STOP SLAVE;
```java
#修改从库指向到主库，使用上一步记录的文件名以及位点
# master_host docker容器linux的ip地址
# master_port 主mysql暴露的端口
# master_user 主mysql的用户名
# master_password 主mysql的密码
#（最后两项修改成刚刚从主mysql查到的，主mysql每次刷新权限或者重启时，这两个值都会改变，所以每次都需要查看是否相同）
CHANGE MASTER TO
master_host = '10.211.55.26',
master_port = 33060,
master_user = 'db_sync',
master_password = 'db_sync',
master_log_file = 'mysql-bin.000001',
master_log_pos = 823;

#启动同步
START SLAVE;

#查看Slave_IO_Runing和Slave_SQL_Runing字段值都为Yes，表示同步配置成功。
show slave status \G;


Sharding-JDBC 操作
配置读写分离策略
也使用docker的，复制的时候记得改一下ip地址
# 配置数据源，给数据源起别名
# m0为用户数据库
spring.shardingsphere.datasource.names=m0,s0

# 一个实体类对应两张表，覆盖
spring.main.allow-bean-definition-overriding=true

#user_db 主服务器
spring.shardingsphere.datasource.m0.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.m0.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.m0.url=jdbc:mysql://10.211.55.26:33060/user_db?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.m0.username=root
spring.shardingsphere.datasource.m0.password=123456

#user_db 从服务器
spring.shardingsphere.datasource.s0.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.s0.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.s0.url=jdbc:mysql://10.211.55.26:33061/user_db?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.s0.username=root
spring.shardingsphere.datasource.s0.password=123456

# 主库从库逻辑数据源定义 ds0 为 user_db
spring.shardingsphere.sharding.master-slave-rules.ds0.master-data-source-name=m0
spring.shardingsphere.sharding.master-slave-rules.ds0.slave-data-source-names=s0

# 配置user_db数据库里面t_user  专库专表
#spring.shardingsphere.sharding.tables.t_user.actual-data-nodes=m0.t_user
# t_user 分表策略，固定分配至 ds0 的 t_user 真实表
spring.shardingsphere.sharding.tables.t_user.actual-data-nodes=ds0.t_user

# 配置主键的生成策略
spring.shardingsphere.sharding.tables.t_user.key-generator.column=user_id
spring.shardingsphere.sharding.tables.t_user.key-generator.type=SNOWFLAKE
# 指定分表策略
spring.shardingsphere.sharding.tables.t_user.table-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_user.table-strategy.inline.algorithm-expression=t_user

# 打开 sql 输出日志
spring.shardingsphere.props.sql.show=true
1


测试代码使用垂直分库时编写的addUserDb和findUserDb方法。
addUserDb方法执行结果：




findUserDb方法执行结果：
```
三、Sharding-Proxy 分库分表操作
ShardingSphere-Proxy 简介



ShardingSphere-Proxy定位为透明的数据库代理端。
Sharding-Proxy独立应用，使用安装服务，进行分库分表或者读写分离配置、启动。
下载安装 Sharding-Proxy
https://shardingsphere.apache.org/document/current/cn/downloads/




下载解压后，需要把lib目录下后缀不全的jar包名补全。

配置 Sharding-Proxy（分库配置）
进入conf目录
修改 server.yaml 文件（此文件为Sharding-Proxy的配置），去除掉authentication和props的注释。

修改 config-sharding.yaml 配置文件（此文件为分库分表的配置）


文件里提示说，如果使用mysql，需要把mysql的驱动jar包放到lib目录下。（到maven下载的jar包里找即可）


在主mysql上新建一个edu_1数据库（create database edu_1;）
然后去掉文件中关于mysql的配置代码的注释，然后进行修改。（这里我连的是docker上的主mysql）
```java
schemaName: sharding_db

dataSources:
 ds_0:
   url: jdbc:mysql://10.211.55.26:33060/edu_1?serverTimezone=UTC&useSSL=false
   username: root
   password: 123456
   connectionTimeoutMilliseconds: 30000
   idleTimeoutMilliseconds: 60000
   maxLifetimeMilliseconds: 1800000
   maxPoolSize: 50

shardingRule:
 tables:
   t_order:
     actualDataNodes: ds_${0}.t_order_${0..1}
     tableStrategy:
       inline:
         shardingColumn: order_id
         algorithmExpression: t_order_${order_id % 2}
     keyGenerator:
       type: SNOWFLAKE
       column: order_id
 bindingTables:
   - t_order
 defaultDatabaseStrategy:
      inline:
     shardingColumn: user_id
     algorithmExpression: ds_${0}
 defaultTableStrategy:
      none:

```
启动 Sharding-Proxy 服务
从终端进入到bin目录，然后./start.sh启动服务。（空格然后加上端口号，可以指定端口，默认3307）


如果打开logs目录里的stdout.log文件，显示下面这种，说明启动成功。


通过终端进行连接：
mysql -uroot -proot -P3307
如果出现下面这种错误，可以尝试连接命令加上-h127.0.0.1。


mysql -uroot -proot -h127.0.0.1 -P3307

新建一张表，并插入一条数据。
```java
use sharding_db;

create table if not exists ds_0.t_order(`order_id` bigint primary key,`user_id` int not null,`status` varchar(50));

insert into t_order(`order_id`,`user_id`,`status`)values(11,1,'test');

```


按照order_id进行分配，因为id是奇数所以被分到了t_order_1表里。

配置 Sharding-Proxy（读写分离）
Sharding-Proxy与Sharding-JDBC一样，并不会进行主从复制，主从复制依然是有MySQL自己完成。
mysql主从复制配置
把上面mysql主从复制的配置复制下来，方便查看：https://blog.csdn.net/qq_36903261/article/details/108457759
使用docker后，需要在主服务器上新建原先的数据库数据表。
因为记录的文件名以及位点每次重启或刷新都会改变，所以以下命令放在这里，方便查看。
主mysql：
#确认位点 记录下文件名以及位点（重启或者刷新都会改变）
show master status;

从mysql：
#先停止同步
STOP SLAVE;
```java
#修改从库指向到主库，使用上一步记录的文件名以及位点
# master_host docker容器linux的ip地址
# master_port 主mysql暴露的端口
# master_user 主mysql的用户名
# master_password 主mysql的密码
#（最后两项修改成刚刚从主mysql查到的，主mysql每次刷新权限或者重启时，这两个值都会改变，所以每次都需要查看是否相同）
CHANGE MASTER TO
master_host = '10.211.55.26',
master_port = 33060,
master_user = 'db_sync',
master_password = 'db_sync',
master_log_file = 'mysql-bin.000001',
master_log_pos = 823;

#启动同步
START SLAVE;

#查看Slave_IO_Runing和Slave_SQL_Runing字段值都为Yes，表示同步配置成功。
show slave status \G;



老师这里只演示读写分离，并没有mysql的主从复制，我写的是主从复制，读写分离。
#主mysql
create database master_slave_order;

1
2
Sharding-Proxy 配置
修改 config-master_slave.yaml 文件（此文件为读写分离的配置）
schemaName: master_slave_db

dataSources:
 master_ds:
   url: jdbc:mysql://10.211.55.26:33060/master_slave_order?serverTimezone=UTC&useSSL=false&allowPublicKeyRetrieval=true
   username: root
   password: 123456
   connectionTimeoutMilliseconds: 30000
   idleTimeoutMilliseconds: 60000
   maxLifetimeMilliseconds: 1800000
   maxPoolSize: 50
 slave_ds_0:
   url: jdbc:mysql://10.211.55.26:33061/master_slave_order?serverTimezone=UTC&useSSL=false&allowPublicKeyRetrieval=true
   username: root
   password: 123456
   connectionTimeoutMilliseconds: 30000
   idleTimeoutMilliseconds: 60000
   maxLifetimeMilliseconds: 1800000
   maxPoolSize: 50

masterSlaveRule:
 name: ms_ds
 masterDataSourceName: master_ds
 slaveDataSourceNames:
   - slave_ds_0
#   - slave_ds_1



测试
启动Sharding-Proxy 服务 ./start.sh

出现这个错误：


在url上加上&allowPublicKeyRetrieval=true即可。
连接sharding-proxy

创建数据表
use master_slave_db;

create table if not exists master_slave_order.t_order(`order_id` bigint primary key,`user_id` int not null,`status` varchar(50));

insert into t_order(`order_id`,`user_id`,`status`)values(11,1,'test');





只修改从mysql中的数据

查询数据
select * from t_order;



```
说明，写入的是主mysql，然后主从复制，从mysql也有了数据，读取的是从mysql。











