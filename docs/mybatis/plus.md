## MyBatis Plus插件

##### 安装

1. Maven依赖

   ~~~xml
   <dependency>
       <groupId>com.baomidou</groupId>
       <artifactId>mybatis-plus-boot-starter</artifactId>
       <version>3.3.2</version>
   </dependency>
   ~~~

   

2. 配置 MapperScan 注解

   ```java
   @MapperScan("dao层接口包路径")
   ```

##### 注解

1. @TableName-表名注解，value=数据库中表名称，autoResultMap=

2. @TableId-主键注解，value=数据库中表主键名称，type=IdType.NONE

   | 值          | 描述                                                         |
   | ----------- | ------------------------------------------------------------ |
   | NONE        | 无状态，该类型为未设置主键类型，类似INPUT                    |
   | AUTO        | 数据库ID自增                                                 |
   | INPUT       | insert前自行设置主键值                                       |
   | ASSIGIN_ID  | 分配ID(主键类型为Number(Long和Integer)或String)(since 3.3.0),使用接口`IdentifierGenerator`的方法`nextId`(默认实现类为`DefaultIdentifierGenerator`雪花算法) |
   | ASSIGN_UUID | 分配UUID,主键类型为String(since 3.3.0),使用接口`IdentifierGenerator`的方法`nextUUID`(默认default方法) |

3. @TableField

4. @Version-乐观锁注解

5. @EnumValue-枚举注解

6. @TableLogic-表字段逻辑处理注解

7. @SqlParser-租户注解，支持method以上级mapper接口上

8. @KeySequence-序列主键策略

#### 核心功能

------

##### BaseMapper<T>

- 通过继承该接口，plus插件默认实现基本查询、新增、修改、删除等方法。

  ```java
  public interface UserMapper extends BaseMapper<User> {
      List<List<?>> findResultByInfo(Select select);
  }
  ```

- 自定义实现多表查询，在接口上定义查询方法，然后在mapper文件中定义同名的select：

  ~~~xml
  <mapper namespace="com.looyeagee.web.mapper.StudentMapper">
      <resultMap id="ResultMap" type="com.looyeagee.web.bean.Result"/>
      <resultMap id="RecordsCount" type="integer"/>
      <select id="findResultByInfo" resultMap="ResultMap,RecordsCount"
              parameterType="com.looyeagee.web.bean.Select" resultType="java.util.List">
          SELECT SQL_CALC_FOUND_ROWS
          `student`.`id` AS `id`,
          `student`.`stuName` AS `stuName`,
          `student`.`stuAge` AS `stuAge`,
          `student`.`graduateDate` AS `graduateDate`,
          `facultylist`.`facultyName` AS `facultyName`
          FROM
          ( `facultylist` JOIN `student` )
          WHERE
          (
          `facultylist`.`id` = `student`.`facultyId`)
          -- 标题模糊搜索
          <if test="stuName != null">
              AND `student`.`stuName` LIKE CONCAT('%',#{stuName},'%')
          </if>
          -- &gt;=是大于等于
          <if test="minAge!=null">
              AND `student`.`stuAge`&gt;= #{minAge}
          </if>
          -- &lt;=是小于等于
          <if test="maxAge!=null">
              AND `student`.`stuAge` &lt;= #{maxAge}
          </if>
  
          -- 没毕业 毕业时间大于现在
          <if test="isGraduate  != null and isGraduate ==false">
              AND `student`.`graduateDate`&gt;=NOW()
          </if>
  
          -- 毕业了 毕业时间小于现在
          <if test="isGraduate  != null and isGraduate ==true">
              AND `student`.`graduateDate`&lt;=NOW()
          </if>
          <if test="orderBy!=null and orderBy!=''">
  
              <if test="highToLow ==null or highToLow ==false">
                  ORDER BY ${orderBy} ASC,`student`.`id` ASC -- 加id ASC是为了保证分页结果的唯一性 mysql排序是不稳定的 https://www.jianshu.com/p/1e8a19738ae4
              </if>
              <if test="highToLow !=null and highToLow ==true">
                  ORDER BY ${orderBy} DESC,`student`.`id` ASC
              </if>
          </if>
          -- 分页查询
          LIMIT
          #{pageNumber},#{pageSize};
          -- 接着查询符合条件个数
          SELECT FOUND_ROWS();
      </select>
  </mapper>
  ~~~

  