---
title: Spring Boot JPA - multiple database 설정.
date: "2019-08-01T00:00:00.000+0900"
template: "post"
draft: false
slug: "/posts/spring-boot-jpa-multiple-database"
category: "Spring"
tags:
  - "Spring Boot"
  - "JPA"
  - "MySQL"
  - "Java"
description: "Spring Boot JPA에서 여러 개의 데이터베이스를 연동 셋팅. 이 구조로 master / slave 처리도 할 수 있음."
---
리플리케이션 되어 있는 디비에서 서비스쪽에서는 전부 마스터, 통계와 같이 슬로우쿼리가 걸리는 애들은 슬레이브로 쿼리를 날리기 위해서 여기까지 왔다. 역시 이 방법이 제일 맘에 드는 것 같다.  
일단 db1: master, db2: slave로 해서 multiple database 설정.

### 설정파일
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

마스터와 슬레이브가 다른지 테스트해봐야하니까 슬레이브에 테스트 데이터를 넣는다.

test.sql
```sql
create table hello (
id bigint not null auto_increment,
world varchar(255),
primary key (id)
) engine=MyISAM;

INSERT INTO `slave`.`hello` (`world`) VALUES ('jared.s');
```

### 도메인 생성 (패키지 중요)
db와 연결할 엔티티를 작성해야하는데, db가 2개인데 다르면 다른패키지로 쓰면 되고 master/slave처럼 같으면 같은 패키지로 써도 된다.  
domain.Hello.java
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

### Repository 생성 (패키지 중요)
이 Repository 위치를 구분해서 마스터 / 슬레이브를 구분할 수 있다.  
masterrepository.HelloRepository.java
```java
public interface HelloRepository extends CrudRepository<Hello, Long> {
}
```
slaverepository.HelloSlaveRepository.java
```java
public interface HelloSlaveRepository extends CrudRepository<Hello, Long> {
}
```

### master datasource 설정
여기서 커스텀 datasource, entityManagerFactory 등을 설정.  
basePackages에 repository 패키지를 쓰면 되고, entityManagerFactory에서 엔티티 패키지를 지정해주면 된다. 그리고 꼭 @Primary를 붙여야 되더라.
  
MasterDataSourceConfig.java
```java
@Configuration
@EnableJpaRepositories(
        basePackages = "com.mudchobo.examples.multipledatabase.masterrepository"
)
public class MasterDataSourceConfig {
    private final JpaProperties jpaProperties;
    private final HibernateProperties hibernateProperties;

    public MasterDataSourceConfig(JpaProperties jpaProperties, HibernateProperties hibernateProperties) {
        this.jpaProperties = jpaProperties;
        this.hibernateProperties = hibernateProperties;
    }

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(EntityManagerFactoryBuilder builder) {
        var properties = hibernateProperties.determineHibernateProperties(
                jpaProperties.getProperties(), new HibernateSettings());
        return builder.dataSource(masterDataSource())
                .properties(properties)
                .packages("com.mudchobo.examples.multipledatabase.domain")
                .persistenceUnit("master")
                .build();
    }

    @Bean
    @Primary
    PlatformTransactionManager transactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(Objects.requireNonNull(entityManagerFactory(builder).getObject()));
    }
}
```

슬레이브 설정은 비슷한데, repository경로는 Slave쪽으로 바꾸면 된다. entity는 같은걸 쓸거니 같은 패키지를 보면 된다. entityManagerFactory, transactionManager는 기본값이 아닌 슬레이브껄로 사용하게 바꾸면 된다.  
  
SlaveDataSourceConfig.java
```java
@Configuration
@EnableJpaRepositories(
        basePackages = "com.mudchobo.examples.multipledatabase.slaverepository",
        entityManagerFactoryRef = "slaveEntityManagerFactory",
        transactionManagerRef = "slaveTransactionManager"
)
public class SlaveDataSourceConfig {
    private final JpaProperties jpaProperties;
    private final HibernateProperties hibernateProperties;

    public SlaveDataSourceConfig(JpaProperties jpaProperties, HibernateProperties hibernateProperties) {
        this.jpaProperties = jpaProperties;
        this.hibernateProperties = hibernateProperties;
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean slaveEntityManagerFactory(EntityManagerFactoryBuilder builder) {
        hibernateProperties.setDdlAuto("none");
        var properties = hibernateProperties.determineHibernateProperties(
                jpaProperties.getProperties(), new HibernateSettings());
        return builder.dataSource(slaveDataSource())
                .properties(properties)
                .packages("com.mudchobo.examples.multipledatabase.domain")
                .persistenceUnit("master")
                .build();
    }

    @Bean
    PlatformTransactionManager slaveTransactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(Objects.requireNonNull(slaveEntityManagerFactory(builder).getObject()));
    }
}
```

MultipleDatabaseApplication.java
```java
@Slf4j
@EnableTransactionManagement
@SpringBootApplication
public class MultipleDatabaseApplication {
    public static void main(String[] args) {
        SpringApplication.run(MultipleDatabaseApplication.class, args);
    }

    @Bean
    public CommandLineRunner demo(HelloRepository helloRepository, HelloSlaveRepository helloSlaveRepository) {
        return args -> {
            var savedHello = helloRepository.save(Hello.builder().world("mudchobo").build());
            log.info("savedHello = {}", savedHello);

            var masterHello = helloRepository.findById(1L);
            log.info("masterHello = {}", masterHello);

            var slaveHello = helloSlaveRepository.findById(1L);
            log.info("slaveHello = {}", slaveHello);
        };
    }
}
```

```
savedHello = Hello(id=1, world=mudchobo)
masterHello = Optional[Hello(id=1, world=mudchobo)]
slaveHello = Optional[Hello(id=1, world=jared.s)]
```

### 예제 github
<a href="https://github.com/mudchobo/mudchobo-examples/tree/master/multiple-database" target="_blank">https://github.com/mudchobo/mudchobo-examples/tree/master/multiple-database</a>  

끗.
