**SpringBoot mybatis 拦截器配置读写分离**
上一篇mysql 在linux 下配置 主从复制,这一篇代码层实现读写分离

实现方式有很多，但是不外乎分为内部配置和使用中间件，下面列举几个常用的方法： 

1.配置多个数据源，根据业务需求访问不同的数据，指定对应的策略：增加，删除，修改操作访问   对应数据，查询访问对应数据，不同数据库做好的数据一致性的处理。由于此方法相对易懂，简单， 不做过多介绍。                                                                                                                                                     
2. 动态切换数据源，根据配置的文件，业务动态切换访问的数据库:此方案通过Spring的AOP，AspactJ来实现动态织入，通过编程继承实现Spring中的AbstractRoutingDataSource,来实现数据库访问的动态切换，不仅可以方便扩展，不影响现有程序，而且对于此功能的增删也比较容易。
3. 通过mycat来实现读写分离:使用mycat提供的读写分离功能，mycat连接多个数据库，数据源只需要连接mycat，对于开发人员而言他还是连接了一个数据库(实际是mysql的mycat中间件)，而且也不需要根据不同业务来选择不同的库，这样就不会有多余的代码产生。
**当然我使用的这种也不知道是第一种还是第二种反正两个都有体现:**
**第一步:**
得有一个DynamicDataSource(动态数据源)继承AbstractRoutingDataSource重写determineCurrentLookupKey方法,原理是这样的:
首先一直往上面找绝对会实现DataSource这个接口,这里就不看了,getConnection() 返回determineTargetDataSource()
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190712150611807.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzc5MTUyOA==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190712151216976.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzc5MTUyOA==,size_16,color_FFFFFF,t_70)
 这里用到了我们需要进行实现的抽象方法determineCurrentLookupKey()，该方法返回需要使用的DataSource的key值，然后根据这个key从resolvedDataSources这个map里取出对应的DataSource，如果找不到，则用默认的resolvedDefaultDataSource![在这里插入图片描述](https://img-blog.csdnimg.cn/20190712152112164.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzc5MTUyOA==,size_16,color_FFFFFF,t_70)
 我能理解的也只有这么多了,太深了还没达到那个层次,如有不足请指出,多多包涵
 继承实现抽象方法
  ```java
  public class DynamicDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceHolder.getDbType();
    }
}
```
**第二步**
DynamicDataSourceHolder 动态返回数据源的作用
```java
public class DynamicDataSourceHolder {
    private static Logger logger= LoggerFactory.getLogger(DynamicDataSourceHolder.class);
    /**
     * 使用ThreadLocal保存数据源的key
     */
    public static final String MASTER_DB="master";
    public static final String SLAVE_DB="slave";
    private static ThreadLocal<String> threadLocal=new ThreadLocal<String>();
    public static String getDbType(){
        //返回当前线程的唯一的序列
        String db=threadLocal.get();
        if (db==null){
            db=MASTER_DB;
        }
        return db;
    }

    /**
     * 设置线程的数据源
     * @param dbType
     */
    public static void setDbType(String dbType){
        logger.debug("所使用的数据源为"+dbType);
        threadLocal.set(dbType);
    }
    /**
     * 清洗数据源
     */
    public static void cleanDbType(){
        threadLocal.remove();
    }
}
```
这个时候就感觉有点像Aop这里使用Mybatis拦截器了
**第三步**
动态数据源拦截器 根据不同的Sql语句,切换不同的数据源,主写从读
这里是最难的需要一定Mybatis 基础,也是最难理解的代码的注释我写的非常详细可以参考理解,如有不足多多包涵
代码如下:
```java
@Intercepts({@Signature(type = Executor.class,method = "update",args = {MappedStatement.class,Object.class}),
             @Signature(type = Executor.class,method = "query",args = {MappedStatement.class,Object.class, RowBounds.class, ResultHandler.class})})
public class DynamicDataSourceInterceptor implements Interceptor {
    private static Logger logger=LoggerFactory.getLogger(DynamicDataSourceInterceptor.class);
    private static String REGEX=".*insert\\u0020.*|.*update\\u0020.*|.*delete.*";
    /**
     * 拦截目标对象
     * @param invocation
     * @return
     * @throws Throwable
     */
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        //判断是否是新事务，如果是新事务，则需要把事务属性存放到当前线程中
        boolean synchronizationActive=TransactionSynchronizationManager.isActualTransactionActive();
        //获取mybatis转换过来的CRUD参数
        Object[] objects=invocation.getArgs();
        System.out.println("转换过后的参数------"+objects);
        //MappedStatement维护了一条<select|update|delete|insert>节点的封装
        MappedStatement mappedStatement= (MappedStatement) objects[0];
        //默认主库写入操作
        String dbKey=DynamicDataSourceHolder.MASTER_DB;
        if (synchronizationActive!=true){
            /**
             * SqlCommandType.SELECT Sql的类型  select|update|insert|delete
             */
            if (mappedStatement.getSqlCommandType().equals(SqlCommandType.SELECT)){
                /**dss
                 * 有些时候数据必须从主库读取比如获取自增长主键,利用主键对数据进行update
                 * 检查一个字符串中是否包含想要查找的值，可以使用string.contains方法，
                 * 使用方法为:被查找的字符串.contains（要查找的字符串），返回类型为boolean。
                 * selectKey 为自增主键(SELECT LAST_INSERT_ID) 使用主库
                 */
                if (mappedStatement.getId().contains(SelectKeyGenerator.SELECT_KEY_SUFFIX)){
                    System.out.println("调用SELECT LAST_INSERT_ID 方法------"+SelectKeyGenerator.SELECT_KEY_SUFFIX);
                    dbKey=DynamicDataSourceHolder.MASTER_DB;
                }else {
                    /**
                     * BoundSql表示动态生成的SQL语句以及相应的参数信息
                     * getSqlSource()  sql对象中对应的sql语句
                     */
                    BoundSql boundSql=mappedStatement.getSqlSource().getBoundSql(objects[1]);
                    //将字符装换为简体中文的小写
                    String sql = boundSql.getSql().toLowerCase(Locale.CHINA).replaceAll("\\t\\n\\r", " ");
                    if (sql.matches(REGEX)){
                        dbKey=DynamicDataSourceHolder.MASTER_DB;
                    }else {
                        dbKey=DynamicDataSourceHolder.SLAVE_DB;
                    }
                }
            }
        }else {
            //不受事物管理的默认主库
            dbKey=DynamicDataSourceHolder.MASTER_DB;
        }
        System.out.println(mappedStatement.getId());
        System.out.println(dbKey);
        System.out.println(mappedStatement.getSqlCommandType().name());
        logger.debug("设置方法为 [{}] 使用的是 [{}], sql类型SqlCommandType [{}].. ",mappedStatement.getId(),dbKey,mappedStatement.getSqlCommandType().name());
         DynamicDataSourceHolder.setDbType(dbKey);
        //执行拦截对象真正的方法
        System.out.println("执行拦截对象真正的方法"+invocation.proceed());
         return invocation.proceed();
    }

    /**
     * 包装目标对象 为目标对象创建动态代理
     * MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护
     * @param target
     * @return
     */
    @Override
    public Object plugin(Object target) {
        System.out.println("目标对象------"+target);
        if (target instanceof Executor){
            System.out.println("创建的代理对象------"+target);
            return Plugin.wrap(target,this);
        }else {
            return target;
        }

    }

    /**
     * 获取插件初始化参数
     * @param properties
     */
    @Override
    public void setProperties(Properties properties) {

    }
}
```
这里简单的说一下这个Mybatis 核心的对象
```java
SqlSessionFactory     产生sqlsession对象
SqlSession            作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能
Executor              MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护
MappedStatement   MappedStatement维护了一条<select|update|delete|insert>节点的封装， 
SqlSource            负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回。          
BoundSql             表示动态生成的SQL语句以及相应的参数信息
Configuration        MyBatis所有的配置信息都维持在Configuration对象之中。
StatementHandler   封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合。
ParameterHandler   负责对用户传递的参数转换成JDBC Statement 所需要的参数，
ResultSetHandler    负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；
#这里大概描述核心对象的作用有机会在写一篇Mybatis的执行顺序
,比如怎样找到XXXMapper.xml---->怎样通过XmlMapperBuilder找到<select><update>这种标签
又封装在Map<String,MappedStatement>---->怎样加载到Configuration,
又是怎样通过SqlSessionFactoryBuilder---->到SqlSessionFactory ---->.... 等
```
**第四步**  
配置数据源
我这里采用的方式比较low,本来想通过yml实现花费了好长时间没弄好,暂时通过这种方式实现
```java
jdbcDriver=com.mysql.jdbc.Driver
jdbcMasterUrl=jdbc:mysql://192.168.0.111:3306/wdssm?useUnicode=true&characterEncoding=utf-8
jdbcSlaveUrl=jdbc:mysql://192.168.0.222:3306/wdssm?useUnicode=true&characterEncoding=utf-8
jdbcUser=root
jdbcPassword=123456
```
代码如下:
```java
@Import({DynamicDataSourceInterceptor.class})
@Configuration
@PropertySource("classpath:jdbc.properties")
@MapperScan(basePackages = "com.wd.springboot.springbootshoping.dao", sqlSessionFactoryRef = "SqlSessionFactory")
public class DataSourceConfig {
    @Autowired
    private DynamicDataSourceInterceptor dynamicDataSourceInterceptor;
    @Bean(name = "master")
    @Primary
    public static DataSource newDataSourceMaster(Environment environment){
        DruidDataSource druidDataSource=new DruidDataSource();
        druidDataSource.setDriverClassName(environment.getProperty("jdbcDriver"));
        druidDataSource.setUrl(environment.getProperty("jdbcMasterUrl"));
        druidDataSource.setUsername(environment.getProperty("jdbcUser"));
        druidDataSource.setPassword(environment.getProperty("jdbcPassword"));
        druidDataSource.setMaxActive(100);
        return druidDataSource;
    }
    @Bean(name = "slave")
    public static DataSource newDataSourceSlave(Environment environment){
        DruidDataSource druidDataSource=new DruidDataSource();
        druidDataSource.setDriverClassName(environment.getProperty("jdbcDriver"));
        druidDataSource.setUrl(environment.getProperty("jdbcSlaveUrl"));
        druidDataSource.setUsername(environment.getProperty("jdbcUser"));
        druidDataSource.setPassword(environment.getProperty("jdbcPassword"));
        druidDataSource.setMaxActive(100);
        return druidDataSource;
    }
    @Bean("dynamicDataSource")
    public DynamicDataSource newDynamicDataSource(@Qualifier("master")DataSource dataSourceMaster,
                                                  @Qualifier("slave") DataSource dataSourceSlave){
        Map<Object,Object> targetDataSources=new HashMap<>(5);
        targetDataSources.put("master",dataSourceMaster);
        targetDataSources.put("slave",dataSourceSlave);
        DynamicDataSource dynamicDataSource=new DynamicDataSource();
        dynamicDataSource.setDefaultTargetDataSource(dataSourceMaster);
        dynamicDataSource.setTargetDataSources(targetDataSources);
        return dynamicDataSource;
    }
    @Bean(name = "SqlSessionFactory")
    @ConditionalOnMissingBean
    public SqlSessionFactory newSqlSessionFactory(@Qualifier("dynamicDataSource") DataSource dynamicDataSource)
            throws Exception {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        /**
         * 自定义拦截器
         */
        factoryBean.setPlugins(new Interceptor[]{dynamicDataSourceInterceptor});
        Resource [] mapperLocations=new PathMatchingResourcePatternResolver().getResources("classpath:mapper/*Mapper.xml");
        factoryBean.setMapperLocations(mapperLocations);
        factoryBean.setDataSource(dynamicDataSource);
        return factoryBean.getObject();
    }

}
```
这个相对来说比较简单,只需要明白每个注解的意思即可
这里对注解进行部分说明一下
```
 1.  @Import注解
在应用中，有时没有把某个类注入到IOC容器中，但在运用的时候需要获取该类对应的bean，此时就需要用到@Import注解
 2.  @Qualifier
 Spring的Bean注入配置注解,该注解指定注入的Bean的名称
 3. @Primary
 自动装配时当出现多个Bean时，@Primary的Bean将作为默认Bean
 4. @ConditionalOnMissingBean
  结合使用注解@ConditionalOnMissingBean和@Bean,可以做到只有特定名称或者类型的Bean不存在于BeanFactory中时才创建某个Bean,这个很深我个人水平感觉解释的并不是太好,大家可以参考进行理解
  @ConditionalOnMissingBean注解的理解参考:
  https://blog.csdn.net/xcy1193068639/article/details/81517456
```
到此为止就可以了代码克隆地址我会提交后方法gitHub 或者码云上面,如有不足请指出,多多包涵谢谢