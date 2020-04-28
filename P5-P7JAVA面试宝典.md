# 阿里系

=======================================================================================

## 	淘宝

=======================================================================================

## 	蚂蚁金服

=======================================================================================

# 腾讯系

=======================================================================================

## 		腾讯视频

=======================================================================================

## 	拍拍

=======================================================================================





# 百度系

=======================================================================================

## 	爱奇艺

=======================================================================================

## 	百度云

=======================================================================================

# 拉勾网

**1.使用mybatis添加数据后返回自增的主键如何操作? 其原理请画图描述?**

答案: 很多情况下, 我们insert一个数据, 由于主键自增 , 我们又需要返回的主键做其他的操作. 我们可以使用 标签 查询返回的主键

```
public interface UserMapper {
    //1.添加用户
    public void insertUser(User user);
}
```

xml方式

```
    <insert id="addUser" parameterType="com.lagou.pojo.User" >
        <!--目的:添加完用户之后,将自动增长的id返回给user对象
            selectKey : 里面可以编写查询语句,查询语句可以在下面insert语句之前或之后执行.产生一定的过.
            keyColumn : 查询数据库表的那个字段,此处指的是id
            keyProperty : java中POJO类中的那个属性
            resultType  :查询到的返回的结果类型 int
            order  : 两个sql语句的先后执行顺序,  SELECT  LAST_INSERT_ID();  INSERT INTO USER VALUES(NULL,#{username},#{birthday},#{sex},#{address});

        -->
        <selectKey keyColumn="id" keyProperty="id" resultType="java.lang.Integer"  order="AFTER">
            SELECT  LAST_INSERT_ID();
        </selectKey>

        <!--
            如果参数类型是对象类型:
            #{写的是对象的属性值}
        -->
        INSERT INTO USER VALUES(NULL,#{username},#{birthday},#{sex},#{address});
    </insert>
```

注解方式

```
// 返回主键字段id值
@Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
@Insert("INSERT INTO USER VALUES(NULL,#{username},#{birthday},#{sex},#{address});")
public interface UserMapper {
    //1.添加用户
    public void insertUser(User user);
}
```

测试:

```
    @Test
    public void insertUserTest() {
        //4.SqlSessionFactory.openSession();  ===> sqlSession
        SqlSession sqlSession = sqlSessionFactory.openSession(true);
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        User user = new User("志平",new Date(),"1","全真教"); 
         System.out.println("插入前: " + user);  // id为0 
        mapper.insertUser(u);
        System.out.println("插入后: " + user);  // id为1  获取自增的主键
        sqlSession.close();
    }
```

获取主键ID的实现原理

不论在xml映射器还是在接口映射器中，添加记录的主键值并非添加操作的返回值。实际上，在MyBatis中执行添加操作时只会返回当前添加的记录数。

```
package org.apache.ibatis.executor.statement;
public class PreparedStatementHandler extends BaseStatementHandler {
	@Override
    public int update(Statement statement) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        // 真正执行添加操作的SQL语句
        ps.execute();
        int rows = ps.getUpdateCount();
        Object parameterObject = boundSql.getParameterObject();
        KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
        // 在执行添加操作完毕之后，再处理记录主键字段值
        keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
        // 添加记录时返回的是记录数，而并非记录的主键字段值
        return rows;
    }
}
```

MyBatis添加操作的时序图 ![1587455632954](https://gitee.com/lagouedu/Java_L2_2020.04.20/raw/master/05_%E5%85%B6%E4%BB%96/4.21%E9%9D%A2%E8%AF%95%E8%AE%A8%E8%AE%BA%E9%A2%98/assets/1587455632954.png)



**2.MyBatis 实现一对一有几种方式?具体怎么操作的？**

答：两种方式. 第一种使用自动映射 resultType ; 第二种使用手动映射 resultMap

举例自动映射:

查询所有订单及订单下用户信息

```
import java.util.Date;
@Data
public class OrderUserVo {
    //oder的所有属性
    private int id;
    private int userId;  //我们如果属性和数据库字段名称不一致可能会有问题.
    private String number;
    private Date createtime;
    private String note;
    //user的属性
    private String username;
    private Date birthday;
    private String sex;
    private String address;
}
 <!--方式1:查询订单及关联的用户信息-->
    <select id="findOderAndUser" resultType="orderuservo">
      SELECT
        o.id,
        o.user_id as userId,
        o.number,
        o.createtime,
        o.note,
        u.username,
        u.birthday,
        u.sex,
        u.address
        FROM
        `order` o
        LEFT JOIN `user` u
        ON o.user_id = u.id
    </select>
   //1.查询所有订单
    @Test
    public void findOrderAndUser() {
        SqlSession sqlSession = factory.openSession();
        OrderMapper mapper = sqlSession.getMapper(OrderMapper.class);
        List<OrderUserVo> list = mapper.findOderAndUser();
        for (OrderUserVo orderUserVo : list) {
            System.out.println(orderUserVo);
        }
    }
```

举例手动映射

```
public class Order {
    private int id;
    private int userId;  //我们如果属性和数据库字段名称不一致可能会有问题.
    private String number;
    private Date createtime;
    private String note;
    private User user;

    public Order() {
    }
  //查询所有订单信息及关联的用户信息
    public List<Order> findOderAndUser02();
 <!--方式2:查询订单及关联的用户信息
        autoMapping true; 如果普通字段的sql的查询字段和 pojo的属性一致,就可以省略,自动映射
    -->
    <resultMap id="orderUserResultMap" type="order" autoMapping="true">
        <!--对主键进行映射  id-->
        <id column="id" property="id"/>
        <!--对其他字段进行映射  result-->
        <result column="user_id" property="userId"/>

        <!--关于order中的user属性的一个映射配置
            association 用来对对象的属性进行关联映射
            property 代表order类中的对象属性名 user
            javaType 代表order类中的user属性的类型
        -->
        <association property="user"  javaType="user" autoMapping="true">
            <!--将查询的字段映射到oder类中的user中的属性中去-->
            <id column="user_id" property="id"/>
        </association>
    </resultMap>
    
    <select id="findOderAndUser02" resultMap="orderUserResultMap">
        SELECT
        o.id,
        o.user_id,
        o.number,
        o.createtime,
        o.note,
        u.username,
        u.birthday,
        u.sex,
        u.address
        FROM
        `order` o
        LEFT JOIN `user` u
        ON o.user_id = u.id
    </select>
```

两种自动映射我们封装数据到一个Vo对象中直接使用resultType 指定vo对象即可, 如果是手动映射我们要建立pojo之间的包含关系,让后使用resultMap手动指定映射关系.

**3.谈谈对Mybatis的一级、二级缓存的认识**

1）一级缓存: 基于 PerpetualCache 的 HashMap 本地缓存，其存储作用域为 Session，当 Session flush 或 close 之后，该 Session 中的所有 Cache 就将清空，默认打开一级缓存。
2）二级缓存与一级缓存其机制相同，默认也是采用 PerpetualCache，HashMap 存储，不同在于其存储作用域为 Mapper(Namespace)，并且可自定义存储源，如 Ehcache。默认不打开二级缓存，要开启二级缓存，使用二级缓存属性类需要实现Serializable序列化接口(可用来保存对象的状态),可在它的映射文件中配置 <cache/> ；
3）对于缓存数据更新机制，当某一个作用域(一级缓存 Session/二级缓存Namespaces)的进行了C/U/D 操作后，默认该作用域下所有 select 中的缓存将被 clear。

**4.简述 Mybatis 的插件运行原理，以及如何编写一个插件**

Mybatis 仅可以编写针对 ParameterHandler、ResultSetHandler、StatementHandler、Executor 这 4 种接口的插件，Mybatis 使用 JDK 的动态代理，为需要拦截的接口生成代理对象以实现接口方法拦截功能，每当执行这 4 种接口对象的方法时，就会进入拦截方法，具体就是 InvocationHandler 的 invoke()方法，会拦截那些你指定需要拦截的方法。编写插件：实现 Mybatis 的 Interceptor 接口并复写 intercept()方法，然后在给插件编写注解，指定要拦截哪一个接口的哪些方法即可, 最后需要在核心配置文件中个配置插件,自定义的插件才会生效.

**5.简述Mybatis的Xml映射文件和Mybatis内部数据结构之间的映射关系？**

Mybatis将所有Xml配置信息都封装到All-In-One重量级对象Configuration内部。在Xml映射文件中，<parameterMap>标签会被解析为ParameterMap对象，其每个子元素会被解析为ParameterMapping对象。<resultMap>标签会被解析为ResultMap对象，其每个子元素会被解析为ResultMapping对象。每一个<select>、<insert>、<update>、<delete>标签均会被解析为MappedStatement对象，标签内的sql会被解析为BoundSql对象。

**6.Mybatis是如何将sql执行结果封装为目标对象并返回的？都有哪些映射形式？**

第一种是使用<resultMap>标签，逐一定义数据库列名和对象属性名之间的映射关系。
第二种是使用sql列的别名功能，将列的别名书写为对象属性名。
有了列名与属性名的映射关系后，Mybatis通过反射创建对象，同时使用反射给对象的属性逐一赋值并返回，那些找不到映射关系的属性，是无法完成赋值的。

**7.Mybatis是如何进行分页的？分页插件的原理是什么？**

Mybatis 使用 RowBounds 对象进行分页，它是针对 ResultSet 结果集执行的内存分页，而非物理分页。可以在 sql 内直接书写带有物理分页的参数来完成物理分页功能，也可以使用分页插件来完成物理分页。
分页插件的基本原理是使用 Mybatis 提供的插件接口，实现自定义插件，在插件的拦截方法内拦截待执行的 sql，然后重写 sql，根 据 dialect 方言，添加对应的物理分页语句和物理分页参数。

**8.MyBaits相对于JDBC和Hibernate有哪些优缺点？**

相对于JDBC
优点：

1. 基于 SQL 语句编程，相当灵活，不会对应用程序或者数据库的现有设计造成任何影响，SQL 写在 XML 里，解除 SQL 与程序代码的耦合，便于统一管理；提供XML标签，支持编写动态 SQL 语句，并可重用；
2. 
3. 与 JDBC 相比，减少了代码量，消除了 JDBC 大量冗余的代码，不需要手动开关连接；
4. 很好的与各种数据库兼容（因为 MyBatis 使用 JDBC 来连接数据库，所以只要 JDBC 支持的数据库 MyBatis 都支持）；
5. 提供映射标签，支持对象与数据库的 ORM 字段关系映射；提供对象关系映射标签，支持对象关系组件维护。
  缺点：
6. SQL 语句的编写工作量较大，尤其当字段多、关联表多时，对开发人员编写 SQL 语句的功底有一定要求；
7. SQL 语句依赖于数据库，导致数据库移植性差，不能随意更换数据库。
  相对于Hibernate
  优点:
  MyBatis 直接编写原生态 SQL，可以严格控制 SQL 执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，因为这类软件需求变化频繁，一但需求变化要求迅速输出成果。
  缺点:
  灵活的前提是 MyBatis 无法做到数据库无关性，如果需要实现支持多种数据库的软件，则需要自定义多套 SQL 映射文件，工作量大。

**9.从以下几个方面谈谈对mybatis的一级缓存**

   **1)mybaits中如何维护一级缓存**

   **2)一级缓存的生命周期**

   **3)mybatis 一级缓存何时失效**

   **4)一级缓存的工作流程**

 1)答案

	  BaseExecutor成员变量之一的PerpetualCache，是对Cache接口最基本的实现，

	  其实现非常简单，内部持有HashMap，对一级缓存的操作实则是对HashMap的操作。

 2)答案

	MyBatis一级缓存的生命周期和SqlSession一致;

	 MyBatis的一级缓存最大范围是SqlSession内部，有多个SqlSession或者分布式的环境下，数据库写操作会引起脏数据;

	MyBatis一级缓存内部设计简单，只是一个没有容量限定的HashMap，在缓存的功能性上有所欠缺

  3)答案

	  a. MyBatis在开启一个数据库会话时，会创建一个新的SqlSession对象，SqlSession对象中会有一个新的Executor对象，Executor对象中持有一个新的PerpetualCache对象；当会话结束时，SqlSession对象及其内部的Executor对象还有PerpetualCache对象也一并释放掉。

 	b. 如果SqlSession调用了close()方法，会释放掉一级缓存PerpetualCache对象，一级缓存将不可用；

        c. 如果SqlSession调用了clearCache()，会清空PerpetualCache对象中的数据，但是该对象仍可使用；

	d.SqlSession中执行了任何一个update操作update()、delete()、insert() ，都会清空PerpetualCache对象的数据

4)答案

	a.对于某个查询，根据statementId,params,rowBounds来构建一个key值，根据这个key值去缓存Cache中取出对应的key值存储的缓存结果；

	b. 判断从Cache中根据特定的key值取的数据数据是否为空，即是否命中；

	c. 如果命中，则直接将缓存结果返回；

	d. 如果没命中：去数据库中查询数据，得到查询结果；将key和查询到的结果分别作为key,value对存储到Cache中；将查询结果返回.

**10.Mybatis映射文件中，如果A标签通过include引用了B标签的内容，请问，B标签能否定义在A标签的后面，还是说必须定义在A标签的前面？**

答案：虽然Mybatis解析Xml映射文件是按照顺序解析的，但是，被引用的B标签依然可以定义在任何地方，Mybatis都可以正确识别。
原理是，Mybatis解析A标签，发现A标签引用了B标签，但是B标签尚未解析到，尚不存在，此时，Mybatis会将A标签标记为未解析状态，然后继续解析余下的标签，包含B标签，待所有标签解析完毕，Mybatis会重新解析那些被标记为未解析的标签，此时再解析A标签时，B标签已经存在，A标签也就可以正常解析完成了。

# 美团点评

# 滴滴

# 国美集团

## 	国美金控

## 	国美电商







### 





#