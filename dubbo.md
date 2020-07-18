# 1. dubbo服务功能介绍 

* 根据身份证号码获取身份信息 

* 从disconf获取redis配置信息，并且当配置更新时会自动推送更新信息 

# 2. 服务框架搭建 

##2.1 dubbo服务框架搭建 

整个demo主要分为三个子模块： 

* dubbo-api：为provider和consumer提供公共的接口 

* dubbo-provider：提供dubbo服务 

* dubbo-consumer：dubbo服务的消费者，外部请求入口 

首先在dubbo-provider中编写准备提供的dubbo服务，然后利用@Service注解标识为dubbo服务，注意这里的注解是dubbo注解不是spring注解。编写完后在application.properties中配置dubbo服务信息： 

``` xml
dubbo.application.name=provider 
#zookeeper服务地址 
dubbo.registry.address=zookeeper://116.62.133.177:2181 
dubbo.protocol.name=dubbo 
dubbo.protocol.port=20880 
#服务路径 
dubbo.scan.base-packages=com.chenjie.ServerImpl 
#web服务运行端口，这里目的使提供者和消费者运行在不同的端口 
server.port=8088 
```

在服务消费者着中需要利用@Reference注解注入dubbo服务实例，并在application配置文件中配置dubbo服务，具体配置信息和provider一样。需要修改的就是服务名，web服务运行端口，并且不需要提供dubbo.scan.base-packages配置信息。 

## 2.2 disconf配置 

disconf是百度开源的一个分布式配置中心，搭建使用disconf主要包含两部分内容： 

### 2.2.1 搭建disconf-web服务器 

disconf-web会依赖redis，mysql，nginx，zookeeper和tomcat，这里我们就不一一搭建相关的依赖了，具体的搭建步骤可参考[disconf文档](https://disconf.readthedocs.io/zh_CN/latest/install/src/02.html)，这里我们利用现有的docker镜像进行搭建，编写下面的docker-compose.yml.文件。 

```
version: '2' 
services: 
  disconf_redis_1: 
    image: daocloud.io/library/redis 
    restart: always 
  disconf_redis_2: 
    image: daocloud.io/library/redis 
    restart: always 
  disconf_zookeeper: 
    image: zookeeper:3.3.6 
    restart: always
  disconf_mysql:
    image: bolingcavalry/disconf_mysql:0.0.1 
    environment: 
      MYSQL_ROOT_PASSWORD: 123456 
      restart: always 
  disconf_tomcat: 
    image: bolingcavalry/disconf_tomcat:0.0.1 
    links: 
      - disconf_redis_1:redishost001 
      - disconf_redis_2:redishost002 
      - disconf_zookeeper:zkhost 
      - disconf_mysql:mysqlhost 
    restart: always 
  disconf_nginx: 
    image: bolingcavalry/disconf_nginx:0.0.1 
    links: 
    	- disconf_tomcat:tomcathost 
    ports: 
    	- "80:80" 
    restart: always 
```

新建disconf文件夹，将该docker-compose文件放在该文件目录下，然后在该目录下打开终端输入一下命令： 

```
docker-compose up -d
```

docker会根据compose文件内容下载相应的镜像文件并启动，文件中nginx开放80端口用于外部连接。如果想要停止disconf服务，可以在该文件目录下执行一下命令： 

```
docker-compose stop
```

服务搭建完成后在本地浏览器输入服务所在主机的ip地址就会显示disconf-web界面，可以通过该界面提供远程配置文件。 

### 2.2.2 配置disconf-client 

* 在pom文件中添加disconf依赖 

* 添加disconf.properties文件 

* 添加disconf.xml文件 

* 编写配置信息相对应的javBean，利用注解@DisconfFile标注相关联的远程配置文件，并利用@DisconfUpdateService注册监听类，当远程配置文件发生改变时会自动推送并触发reload方法。在配置文件相对应的属性的get方法上利用注解@DisconfFileItem说明该注解和远程配置文件中的哪些属性相对应。 

***通过上面的方式配置好后，启动服务获取disconf远程配置时会报错：zkhost无法解析。zkhost是客户端从disconf-web端获取的zookeeper服务ip地址，但是在本地无法解析zkhost，所以需要在本地host文件中添加该host和服务所在ip地址的映射关系。*** 

##2.3 zookeeper配置 

在本demo中涉及到的disconf，kafka，dubbo都会依赖zookeeper，所以我们首先需要启动zookeeper服务。在上面搭建disconf服务时通过docker已经安装启动了zookpeer服务，但是在docker-compose文件中该docker实例并没有对外暴露端口，无法从docker环境外连接zookeeper服务。所以修改docker-compose文件提供zookeeper端口映射，具体操作如下： 

``` 
disconf_zookeeper: 
  image: zookeeper:3.3.6 
  ports: 
  	- "80:80" 
  restart: always 
```

## 2.4 redis缓存配置 

demo中为身份信息服务提供redis缓存，以身份证号码作为缓存key，身份信息为缓存值。 

* 首先安装启动redis服务，可使用docker镜像直接启动，也可以下载安装包安装启动 

* spring缓存配置 

```java 
  @Configuration 
  @EnableCaching 
  public class RedisConfig extends CachingConfigurerSupport {  

    @Bean 
    public CacheManager cacheManager(RedisTemplate<?,?> redisTemplate) { 
      CacheManager cacheManager = new RedisCacheManager(redisTemplate); 
      return cacheManager; 
    } 
    
    @Bean 
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory factory) { 
      RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>(); 
      RedisSerializer stringSerializer=new StringRedisSerializer(); 
      redisTemplate.setKeySerializer(stringSerializer); 
      redisTemplate.setHashKeySerializer(stringSerializer); 
      redisTemplate.setConnectionFactory(factory); 
      return redisTemplate; 
    } 
  } 
```

* 配置redis连接信息（如果上面的配置类中配置了redis连接就不需要了)

```
spring.redis.database=0 
spring.redis.host=116.62.133.177 
spring.redis.port=6379 
spring.redis.password=dezhixiang 
spring.redis.pool.max-active=8 
spring.redis.pool.max-wait=-1 
spring.redis.pool.max-idle=8 
spring.redis.pool.min-idle=0 
spring.redis.timeout=0 
```

* 在服务类上利用下面的注解配置标识需要缓存内容 

```java
//value表示缓存的配置Bean，key表示存储的key值，缓存的value值为方法的返回信息 
@Cacheable(value = "cacheManager",key="#cardId") 
```

## 2.5 配置kafka 

demo中通过身份证号码获取到的身份信息会写入kafka，供其他程序进行消费。 

* 添加pom依赖 

* 在application文件中添加kafka配置信息 

``` 
spring.kafka.bootstrap-servers=116.62.133.177:9092 
spring.kafka.consumer.group-id=Group 
```

* 在代码中利用自动注入注解注入kafkaTemplate对应的Bean，通过它进行kafka读写 

* 编写消费者进行测试,利用@KafkaListener标识监听的主题，当有信息时会自动运行监听方法 

```java
@KafkaListener(topics = "UserInfo") 
public void listenT1(ConsumerRecord<?, ?> cr) { 
  logger.info("listenT1收到消息！！ topic:>>> " + cr.topic() + " key:>> " + cr.key() + " value:>> " + cr.value()); 
} 
```

## 2.6 配置Hibernate 

通过身份证号码获取身份信息时我们需要根据前六位查询数据库获取地址信息，这里我们采用Hibernate作为数据库连接的持久化框架。 

* 添加pom依赖 

* 在application配置文件中添加数据库连接信息 

```
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.MySQLDialect 
spring.datasource.driverClassName = com.mysql.jdbc.Driver 
spring.datasource.url = jdbc:mysql://116.62.133.177:3306/test 
spring.datasource.username = root 
spring.datasource.password = root 
```

* 编写与数据库信息相对应的JavaBean类，@Entity注解表示这是一个数据库实体类，@Table标识对应的数据库表，@Column标识属性对应的数据库字段，@Id标识主键字段 

```java
@Entity 
@Table(name = "address_code") 
public class UserAddress { 
  @Id 
  @Column(name = "code")
  private String address_code;
  
  @Column(name = "address")
  private String address;
  
  public String getAddress_code() { 
  	return address_code; 
  } 
  
  public void setAddress_code(String address_code) { 
  	this.address_code = address_code; 
  } 
  
  public String getAddress() { 
  	return address; 
  } 
  
  public void setAddress(String address) { 
  	this.address = address; 
  } 
} 
```

* 编写数据库操作类  

```java
@Repository 
public interface AddressDao extends JpaRepository<UserAddress,String> { } 
```

## 2.7 java.lang.NoSuchMethodError错误处理方法 

***错误原因***：导入的包存在冲突 

***解决方法***：在idea中安装maven helper插件，打开pom文件点击会出现Dependency Analyzer功能，利用该功能可以找到存在冲突的依赖包，删除多余的依赖就行了。 

