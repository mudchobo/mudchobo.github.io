---
title: Spring Boot JPA - MySQL 컬럼 주석 달기.
date: "2019-07-04T00:00:00.000+0900"
template: "post"
draft: false
slug: "/posts/spring-boot-jpa-mysql-comment"
category: "Spring"
tags:
  - "Spring Boot"
  - "JPA"
  - "MySQL"
  - "Java"
description: "Spring Boot JPA를 이용해서 MySQL 칼럼에 comment를 달자."
---
JPA에서는 comment를 따로 지원하지 않는다. 이게 이유를 찾아보니 일부 데이터베이스가 컬럼 comment를 지원하지 않는 것 같다. 하지만, MySQL에 comment를 달아두고 싶었다(사내 보안 정책이 꼭 달아야한다고 해서 기록해둘라고....)  

columnDifinition 속성을 이용하면 된다. 그리고 OneToMany관계에 있는 칼럼에 comment를 달고 싶다면 JoinColumn을 활용하면 된다.  
  
사람1 : 폰N 을 구성하려면 아래와 같이 하면 된다.

Person.java
```java
@Entity
public class Person {

    @Id
    @GeneratedValue
    private long id;

    @Column(columnDefinition = "varchar(10) not null comment '이름'")
    private String name;

    @OneToMany
    @JoinColumn(name = "person_id", columnDefinition = "bigint not null comment '사람 아이디'")
    private List<Phone> phones;
}

```

Phone.java
```java
@Entity
public class Phone {
    @Id
    @GeneratedValue
    private long id;

    @Column(columnDefinition = "varchar(10) not null comment '휴대폰 이름'")
    private String name;

}

```

아래와 같이 DDL이 생김.
```sql
create table if not exists test.person
(
	id bigint not null
		primary key,
	name varchar(10) not null comment '이름'
)
engine=MyISAM;

create table if not exists test.phone
(
	id bigint not null
		primary key,
	name varchar(10) not null comment '휴대폰 이름',
	person_id bigint not null comment '사람 아이디'
)
engine=MyISAM;

create index FKs31jcc7fn0j8g0j9truq1fj23
	on test.phone (person_idddd);


```

