# MyBatis的Insert操作自增主键的实现，Mysql协议与JDBC实现

### 背景
```
Mybatis中配置了Insert 操作时，添加了 useGeneratedKeys = true 的配置，就可以在插入的model完成后获取到主键的值，用于业务
1.有些场景，插入表单完需要返回id作，后续操作
```

### 例子
```


/**
 * @param
 * @Author: zhuangjiesen
 * @Description:
 * @Date: Created in 2018/6/19
 */
@Mapper
public interface UserMapper {

    @Insert("INSERT INTO `dragsunweb`.`user`(`user_id`, `user_name`, `user_password`) VALUES (#{id}, #{name}, #{password})")
    @Options(useGeneratedKeys = true)
    public Integer insert(User user);

}
```

#### 之后 user.getId() , 可以获取到对应的主键


### 原理/源码

```
Mybatis都是讲mappper类扫描后，通过组装(BeanDefinition)成 MapperFactoryBean.class，
然后最后通过MapperProxy.class(代理) 将mapper实例化，
给人的体验就是可以直接调用mapper操作数据库
一个insert执行顺序(一步步打断点):
1.UserMapper.insert();
2.MapperProxy.invoke();
3.mapperMethod.execute();
4.DefaultSqlSession.insert();
5.BaseExecutor.update()
6.SimpleExecutor.doUpdate();
7.StatementHandler.update();
8.PreparedStatementHandler.update();
在8中就可以发现执行完sql 后，调用的KeyGenerator
```

#### PreparedStatementHandler.update()代码片段：
```

public class PreparedStatementHandler extends BaseStatementHandler {


  @Override
  public int update(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    int rows = ps.getUpdateCount();
    Object parameterObject = boundSql.getParameterObject();
    //调用 KeyGenerator
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
    return rows;
  }


}

```
### KeyGenerator.class , 在每个sql初始化时，在mybatis会初始化成 MappedStatement.class实例

```
/**
 * @author Clinton Begin
 */
public interface KeyGenerator {

  void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

  void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

}

```

#### Tips:
```
这里有个问题其实 KeyGenerator.class 可以用来作拦截器的工作，或者拓展，
可是在 MappedStatement.class的内部构造类 Builder.class 中，只能默认使用2个类
```

#### 其实这个类Mybatis中默认只有2个:
```
NoKeyGenerator.class (如果不配置useKeyGenerator)

Jdbc3KeyGenerator.class (本次博客的主角)

```

#### Jdbc3KeyGenerator.class 源码解析，核心 processBatch() 方法，也挺简单的，就是类似拦截器，在prepareStatment执行完作了操作

```

/**
 * @author Clinton Begin
 * @author Kazuki Shimizu
 */
public class Jdbc3KeyGenerator implements KeyGenerator {

  /**
   * A shared instance.
   * @since 3.4.3
   */
  public static final Jdbc3KeyGenerator INSTANCE = new Jdbc3KeyGenerator();

  @Override
  public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    // do nothing
  }

  @Override
  public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    processBatch(ms, stmt, getParameters(parameter));
  }

  public void processBatch(MappedStatement ms, Statement stmt, Collection<Object> parameters) {
    ResultSet rs = null;
    try {
    //这里获取到执行的sql结果 ， 获取主键的字段值(实体类的映射)，然后set进去很简单
      rs = stmt.getGeneratedKeys();
      final Configuration configuration = ms.getConfiguration();
      final TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
      final String[] keyProperties = ms.getKeyProperties();
      final ResultSetMetaData rsmd = rs.getMetaData();
      TypeHandler<?>[] typeHandlers = null;
      if (keyProperties != null && rsmd.getColumnCount() >= keyProperties.length) {
        for (Object parameter : parameters) {
          // there should be one row for each statement (also one for each parameter)
          if (!rs.next()) {
            break;
          }
          final MetaObject metaParam = configuration.newMetaObject(parameter);
          if (typeHandlers == null) {
            typeHandlers = getTypeHandlers(typeHandlerRegistry, metaParam, keyProperties, rsmd);
          }
          //获取插入的对象参数，把主键值插入回对象属性
          populateKeys(rs, metaParam, keyProperties, typeHandlers);
        }
      }
    } catch (Exception e) {
      throw new ExecutorException("Error getting generated key or setting result to parameter object. Cause: " + e, e);
    } finally {
      if (rs != null) {
        try {
          rs.close();
        } catch (Exception e) {
          // ignore
        }
      }
    }
  }

  private Collection<Object> getParameters(Object parameter) {
    Collection<Object> parameters = null;
    if (parameter instanceof Collection) {
      parameters = (Collection) parameter;
    } else if (parameter instanceof Map) {
      Map parameterMap = (Map) parameter;
      if (parameterMap.containsKey("collection")) {
        parameters = (Collection) parameterMap.get("collection");
      } else if (parameterMap.containsKey("list")) {
        parameters = (List) parameterMap.get("list");
      } else if (parameterMap.containsKey("array")) {
        parameters = Arrays.asList((Object[]) parameterMap.get("array"));
      }
    }
    if (parameters == null) {
      parameters = new ArrayList<Object>();
      parameters.add(parameter);
    }
    return parameters;
  }

  private TypeHandler<?>[] getTypeHandlers(TypeHandlerRegistry typeHandlerRegistry, MetaObject metaParam, String[] keyProperties, ResultSetMetaData rsmd) throws SQLException {
    TypeHandler<?>[] typeHandlers = new TypeHandler<?>[keyProperties.length];
    for (int i = 0; i < keyProperties.length; i++) {
      if (metaParam.hasSetter(keyProperties[i])) {
        TypeHandler<?> th;
        try {
          Class<?> keyPropertyType = metaParam.getSetterType(keyProperties[i]);
          th = typeHandlerRegistry.getTypeHandler(keyPropertyType, JdbcType.forCode(rsmd.getColumnType(i + 1)));
        } catch (BindingException e) {
          th = null;
        }
        typeHandlers[i] = th;
      }
    }
    return typeHandlers;
  }

  private void populateKeys(ResultSet rs, MetaObject metaParam, String[] keyProperties, TypeHandler<?>[] typeHandlers) throws SQLException {
    for (int i = 0; i < keyProperties.length; i++) {
      String property = keyProperties[i];
      TypeHandler<?> th = typeHandlers[i];
      if (th != null) {
        Object value = th.getResult(rs, i + 1);
        metaParam.setValue(property, value);
      }
    }
  }

}

```

### 问题
```
这里就会发现其实insert操作完获取主键直接可以通过 ResultSet 获取到。而 KeyGenerator.class其实就是个设计模式封装而已。

通过断点发现JDBC 的 ResultSetImpl.class (ResultSet.class 接口实现) ，其实是有字段 resultId 、 updateId (mysql注释: /** Value generated for AUTO_INCREMENT columns */ -> 就是自增主键的值 )

还有个字段是 updateCount (mysql注释: /** How many rows were affected by UPDATE/INSERT/DELETE? */ -> 就是影响行数 )

```

### 由此推断不是mybatis实现的获取自增主键的功能，而是JDBC原生实现了这个功能


#### 然后就用过原始的JDBC程序测试:
```

    public static void main(String[] args ) throws Exception {
        Connection conn = getConn();
        int i = 0;
        String sql = " INSERT INTO `dragsunweb`.`user` ( `user_id`, `user_name`, `user_password` ) VALUES ( null , 'hahaha',  '2233' )";
        PreparedStatement pstmt;
        try {
            pstmt = (PreparedStatement) conn.prepareStatement(sql);
            long id = 0L;
        
            //这里可以获取执行行数
            i = pstmt.executeUpdate();
            //这里可以查看到最后插入的主键值
            id = pstmt.getLastInsertID();
            System.out.println(String.format("1. id : %d , rows : %d " , id  , i));

            i = pstmt.executeUpdate();
            id = pstmt.getLastInsertID();
            System.out.println(String.format("2. id : %d , rows : %d " , id  , i));
            pstmt.close();
            conn.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }


------
结果就是如我所说，打印出了对应的主键值
```

### 结果就是如我所说，打印出了对应的主键值，验证了我所说，mybatis只是封装了这个功能


#### 现在的困惑是：
```
1.是1条sql(insert)就可以获取到主键的值吗，平时在可视化工具中执行的时候只能获取影响行数；
2.通过mysql配置或是多条语句
```

#### 通过打JDBC底层实现的代码的断点
```
代码就不贴了，很抽象
步骤是
1.序列化参数
2.序列化sql
3.格式化mysql 通信协议
4.获取mysql 连接
5.发送mysql请求
6.获取返回报文

----以下是源码执行顺序-----
PreparedStatement.execute();
//搜这个关键代码块
Buffer sendPacket = fillSendPacket();
fillSendPacket();//用来封装发送的mysql报文协议
executeInternal();方法中发送了请求
ConnectionImpl.execSQL()
//MysqlIO这里有mysql的协议封装 
MysqlIO.sqlQueryDirect();//发送请求
MysqlIO.readAllResults()//解析返回值报文

```

#### 结论:
```
mysql的协议中，服务器响应报文是对于insert/update/delete是有返回主键值的，只是应用层中没有给出接口


4.3.1 OK 响应报文
客户端的命令执行正确时，服务器会返回OK响应报文。
MySQL 4.0 及之前的版本
字节	说明
1	OK报文，值恒为0x00
1-9	受影响行数（Length Coded Binary）
1-9	索引ID值（Length Coded Binary）
2	服务器状态
n	服务器消息（字符串到达消息尾部时结束，无结束符）
```

### mysql协议博客 
[MySQL协议分析](https://www.cnblogs.com/davygeek/p/5647175.html)

[mysql网络协议官网](https://dev.mysql.com/doc/internals/en/binary-protocol-value.html)

