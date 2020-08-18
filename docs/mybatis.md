##### 概述

实现了数据持久化的开源框架，简单理解就是对JDBC进行封装

- 提供XML标签，支持编写动态SQL语句
- 提供映射标签，支持对象与数据库的ORM自动关系映射

缺点：SQL语句编写工作量较大，尤其是字段多、关联表多时。

核心接口和类

​    SqlSessionFactoryBuilder、SqlSessionFactory、Sqlsession

开发方式

​    使用原生接口、Mapper代理实现自定义接口

##### 应用

1. pom引入。定义resource标签，指定java目录下的mapper配置文件夹下所有文件，否则框架只支持resouces文件夹下的配置文件读取

2. 新建表

3. 新建实体类

4. 创建MyBatis的配置文件mybatis-config.xml，配置JDBC事务管理、POOLED配置数据源连接池

5. 使用原生接口

   1. 自定义SQL语句，写在Mapper.xml文件中。推荐使用<font color="#8B0000">#{}</font>，不要用${}存在注入风险

   2. 在全局mybatis-config.xml文件中注册mapper.xml

   3. 使用Sqlsession

      ~~~java
      public void test() throws IOException {
      
          // 根据 mybatis-config.xml 配置的信息得到 sqlSessionFactory
          String resource = "mybatis-config.xml";
          InputStream inputStream = Resources.getResourceAsStream(resource);
          SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
          // 然后根据 sqlSessionFactory 得到 session
          SqlSession session = sqlSessionFactory.openSession();
      
          // 模糊查询
          List<Student> students = session.selectList("findStudentByName", "三颗心脏");
          for (Student student : students) {
              System.out.println("ID:" + student.getId() + ",NAME:" + student.getName());
          }
      }
      ~~~

6. Mapper代理接口

   1. 自定义接口，定义业务相关方法。

      ~~~java
      //无需实现接口，框架会根据规则自动创建接口的实现类的代理对象。
      ~~~

   2. 编写方法相对应的Mapper.xml文件。

      ~~~xml
      <mapper namespace="com.yuan.dao.UserDao">
          <select id="findStudentByName" parameterMap="java.lang.String" resultType="Student">
          SELECT * FROM student WHERE name LIKE '%${value}%' 
          </select>
      </mapper>
      ~~~

      ~~~java
      //id=接口中方法名称，parameterMap对应接口中方法参数类型，resultType对应接口中方法参数类型。
      ~~~

##### 详解

- Mapper.xml

  - satement标签：select、update、delete、insert

  - parameterType：

  - resultType：

  - 动态SQL：if、choose-when-otherwise、trim, where, set、foreach

    ~~~xml
    <select id="findActiveBlogWithTitleLike"
         resultType="Blog">
      SELECT * FROM BLOG 
      WHERE state = ‘ACTIVE’ 
      <if test="title != null">
        AND title like #{title}
      </if>
    </select>
    ~~~

    ~~~xml
    <choose>
        <when test="title != null">
          AND title like #{title}
        </when>
        <when test="author != null and author.name != null">
          AND author_name like #{author.name}
        </when>
        <otherwise>
          AND featured = 1
        </otherwise>
    </choose>
    ~~~

    

