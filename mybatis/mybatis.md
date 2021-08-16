1.mybatis-config配置文件

2.加载配置文件到内存

3.创建Mapper和Configuration

4 1解析配置文件  2创建sqlSessionFactory对象

5 创建sqlSessionFactory及其实现类

6 创建sqlSession及其实现类

7 创建executor接口及其实现类

传入sqlid,包装成mappedStatement

statementHandler解析sql,parameterHandler解析参数

解析好的sql参数全部传入jdbc的preparedStament,执行拿到结果resultSet

通过resultSetHandler解析resultSet,识别resultType/resultMap,反射包装成需要的对象类型

最终返回上级



Mybatis一级缓存(sqlSession)

每个sql方法对应一个session

如果有增删改的操作,并提交了事务,会刷新一级缓存



二级缓存,是mapper级别,多个sqlSession共享

二级缓存 缓存的是查出来的对象的数据,不是对象

二级缓存有可能存在硬盘中,读取的时候需要反序列化,因此对象要实现serializable

分布式环境下,可以使用redis实现二级缓存,有官方实现的redisCache,从resource目录下的redis.properties文件读取redis配置



mybatis插件原理,主要是通过拦截这四个对象,在拦截器里调用方法,添加增强逻辑

实现mybatis的interceptor接口并复写intercept方法,然后在给插件编写注解,指定要拦截哪一个接口的哪些方法,最后再配置文件中配置编写的插件

Executor   StatementHandler  ParameterHandler  ResultSetHandler



sqlSession-->executor-->statementHandler-->parameterHandler-->resultsetHandler-->



mybatis构建configuration是使用的构建者模式,一步步传入大量的其他对象,最终构建复杂的configuration对象











自定义金额,传金额过来创建record,id照常传过去



明确点:  1.财务展示,这个财务格式是直接在后台上传excel还是通过展示一个表单,一点点的填数据再上传

2 新闻搜索,