---
layout: post
title:  "[Hibernate] load-by-multiple-id API 를 사용할때 알아야할 것들"
date:   2016-08-05 10:55:52 +0900
categories: Hibernate
---
Hibernate 5.1 Release 에 추가된 load-by-multiple-id API 는 여러가지로 유용하게 사용할 수 있다. <br />
[Hibernate 5.1 Release](http://in.relation.to/2016/02/10/hibernate-orm-510-final-release/)

그리고 이 API 를 소개하는 글이 올라와서 이외에 조심해야 되는 부분, 그리고 5.2.2 에서 추가되는 옵션에 대해서 간단히 정리하고자 한다.__ [How to fetch muliple entities by id with Hibernate 5](http://www.thoughts-on-java.org/fetch-multiple-entities-id-hibernate/)

## load-by-multiple-id API 란? (요약)

JPA 나 Hibernate 5.1 이전 버전를 사용할 때 database 에서 다수의 entity들을 조회하기 위한 방법으로 2가지를 생각할 수 있다.

1. 조회하고자 하는 id 각각을 EntityManager.find 메소드를 사용하여 Query를 각각 실행한다.

   > 조회하고자 하는 Entity 가 많다면 너무 많은 Query 를 실행하게 되서 Application 의 성능 저하가 발생할 수 있다.
 
2. IN statement를 사용하여 query 를 만들어서 실행한다.

   {% gist mhyeon-lee/3cbecf69967a624697b178a99b299882 JpaQuery.java %}

   > Oracle과 같은 몇개의 database 는 IN statement의 파라미터 갯수에 제한을 둔다. <br />
   > 많은 엔티티를 한번의 batch 로 조회하게 되면 성능 이슈가 발생할 수 있다. <br />
   > Hibernate 1차 Cache(Session) 에 이미 올라와 있는 엔티티를 체크하지 않고 모든 엔티티를 database 에서 조회한다. <br />

