---
title: Spring Boot JPA - master slave 분기 처리 - transactional 방식
date: "2019-07-31T00:00:00.000+0900"
template: "post"
draft: false
slug: "/posts/spring-boot-jpa-master-slave"
category: "Spring"
tags:
  - "Spring Boot"
  - "JPA"
  - "MySQL"
  - "Java"
description: "Spring Boot JPA에서 기본은 master로 처리하고, 원할 때 slave로 처리하자."
---
스프링 부트에서 JPA는 기본 셋팅은 1개의 datasource만 설정하게 되어 있다. 하지만, master / slave replication이 되어 있는 디비를 둘 다 연결하고 싶을 때에는 조금 까다롭게 설정해야 한다.  
두 가지 방식 정도로 할 수 있는데, @Transactional방식과 @AOP를 이용한 방식이 있다. 일단 이 글에서는 Transactional방식을 먼저... 

build.gradle
```groovy
plugins {
    id 'org.springframework.boot' version '2.1.6.RELEASE'
	id 'java'
}

apply plugin: 'io.spring.dependency-management'

group = 'com.mudchobo.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	compileOnly 'org.projectlombok:lombok'
	runtimeOnly 'mysql:mysql-connector-java'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

application.properties
```properties
spring.datasource.master.jdbc-url=jdbc:mysql://localhost:3306/master?serverTimezone=Asia/Seoul
spring.datasource.master.username=root
spring.datasource.master.password=
spring.datasource.slave.jdbc-url=jdbc:mysql://localhost:3306/slave?serverTimezone=Asia/Seoul
spring.datasource.slave.username=root
spring.datasource.slave.password=
spring.jpa.database=mysql
spring.jpa.hibernate.use-new-id-generator-mappings=false
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=create-drop
``` 

slave에 아래 테이블과 데이터를 넣어서 마스터랑 슬레이브를 테스트를 위해 구분하자. 마스터테이블은 create-drop으로 생성.  

test.sql
```sql
create table hello (
id bigint not null auto_increment,
world varchar(255),
primary key (id)
) engine=MyISAM;

INSERT INTO `slave`.`hello` (`world`) VALUES ('jared.s');
```

Transactional readOnly에 따라 분기하는 CustomRoutingDataSource  

ReplicationRoutingDataSource.java
```java
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                ? "slave"
                : "master";
    }
}
```

마스터 데이터소스와 슬레이브 데이터소스를 정의하고, 그걸 분기하는걸 만들어놓은 설정이다.

DataSourceConfig.java
```java
@Configuration
@EnableAutoConfiguration(exclude = {DataSourceAutoConfiguration.class})
@EnableTransactionManagement
@EnableJpaRepositories(basePackages = {"com.mudchobo.example.masterslave"})
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    public DataSource routingDataSource(@Qualifier("masterDataSource") DataSource masterDataSource,
                                        @Qualifier("slaveDataSource") DataSource slaveDataSource) {
        var routingDataSource = new ReplicationRoutingDataSource();

        var dataSourceMap = new HashMap<>();
        dataSourceMap.put("master", masterDataSource);
        dataSourceMap.put("slave", slaveDataSource);
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(masterDataSource);

        return routingDataSource;
    }

    @Primary
    @Bean
    public DataSource dataSource(@Qualifier("routingDataSource") DataSource routingDataSource) {
        return new LazyConnectionDataSourceProxy(routingDataSource);
    }
}
```
기본 DataSource 설정을 제외하고 커스텀한걸로 @Bean으로 만들어서 바꿈.

Hello.java
```java
@Entity
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class Hello {
    @GeneratedValue
    @Id
    long id;

    @Column
    String world;
}
```

HelloRepository.java
```java
public interface HelloRepository extends CrudRepository<Hello, Long> {
}
```

HelloService.java
```java
@Transactional
@Service
public class HelloService {

    private final HelloRepository helloRepository;

    public HelloService(HelloRepository helloRepository) {
        this.helloRepository = helloRepository;
    }

    public Optional<Hello> get(long id) {
        return helloRepository.findById(id);
    }

    public Hello save(Hello hello) {
        return helloRepository.save(hello);
    }
}
```

슬레이브 서비스에는 readOnly=true를 넣는다.

HelloSlaveService.java
```java
@Transactional(readOnly = true)
@Service
public class HelloSlaveService {
    private final HelloRepository helloRepository;

    public HelloSlaveService(HelloRepository helloRepository) {
        this.helloRepository = helloRepository;
    }

    public Optional<Hello> get(long id) {
        return helloRepository.findById(id);
    }
}
```

샘플로 save 후 helloService에서 가져올 때와 helloSlaveService에서 가져올 때 로그를 찍었다.

MasterSlaveApplication.java
```java
@Slf4j
@SpringBootApplication
public class MasterSlaveApplication {

	public static void main(String[] args) {
		SpringApplication.run(MasterSlaveApplication.class, args);
	}

	@Bean
	public CommandLineRunner demo(HelloService helloService, HelloSlaveService helloSlaveService) {
		return args -> {
			var savedHello = helloService.save(Hello.builder().world("mudchobo").build());
			log.info("savedHello = {}", savedHello);

			var hello1 = helloService.get(1);
			log.info("hello1 = {}", hello1);

			var hello2 = helloSlaveService.get(1);
			log.info("hello2 = {}", hello2);
		};
	}
}
```

결과
```bash
savedHello = Hello(id=1, world=mudchobo)
hello1 = Optional[Hello(id=1, world=mudchobo)]
hello2 = Optional[Hello(id=1, world=jared.s)]
```

이 방식의 단점은 쓸데없이 @Transactional을 붙여야 한다. 이걸 붙이지 않으면, findById만 하는 곳에서는 readOnly로 취급해서 무조건 slave를 가져오게 된다. 그래서 AOP방식으로 쓰는 게 더 나을 것 같다.
 