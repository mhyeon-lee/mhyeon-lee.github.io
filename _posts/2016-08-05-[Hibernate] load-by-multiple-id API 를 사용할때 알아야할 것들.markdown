---
layout: post
title:  "[Hibernate] load-by-multiple-id API 를 사용할때 알아야할 것들"
date:   2016-08-05 10:55:52 +0900
categories: Hibernate
---
Hibernate 5.1 Release 에 추가된 load-by-multiple-id API 는 여러가지로 유용하게 사용할 수 있다. <br />
[Hibernate 5.1 Release](http://in.relation.to/2016/02/10/hibernate-orm-510-final-release/)

아래 글에 API 에 대한 간단한 소개가 있어 이에 대한 요약과 이외에 조심해야 되는 부분, 그리고 5.2.2 에서 추가되는 옵션에 대해서 간단히 정리하고자 한다. <br /> [How to fetch muliple entities by id with Hibernate 5](http://www.thoughts-on-java.org/fetch-multiple-entities-id-hibernate/)

## load-by-multiple-id API (요약)

JPA 나 Hibernate 5.1 이전 버전를 사용할 때 database 에서 다수의 entity들을 조회하기 위한 방법으로 2가지를 생각할 수 있다.

1. 조회하고자 하는 id 각각을 EntityManager.find 메소드를 사용하여 Query를 각각 실행한다.

   > 조회하고자 하는 Entity 가 많다면 너무 많은 Query 를 실행하게 되서 Application 의 성능 저하가 발생할 수 있다.
 
2. IN statement를 사용하여 query 를 만들어서 실행한다.

   {% gist mhyeon-lee/3cbecf69967a624697b178a99b299882 JpaQuery.java %}

   > Oracle과 같은 몇개의 database 는 IN statement의 파라미터 갯수에 제한을 둔다. <br />
   > 많은 엔티티를 한번의 batch 로 조회하게 되면 성능 이슈가 발생할 수 있다. <br />
   > Hibernate 1차 Cache(Session) 에 이미 올라와 있는 엔티티를 체크하지 않고 모든 엔티티를 database 에서 조회한다. <br />
   
* Load multiple entities by their primary key

Hibernate 5.1 은 다수의 엔티티들을 한번의 API 실행하고 위와 같은 문제점들을 회피할 수 있는 새로운 API 를 제공한다.

{% gist mhyeon-lee/45da2f5d6a90e981e67401872c43b587 MultiLoad.java %}

Hibernate Session 에서 위와 같은 API 를 사용하면 id 가 1L, 2L, 3L 인 PersionEntity 클래스를 조회할 수 있다.
Hiberante 는 3개의 Id 를 파라미터로 IN statement Query 를 실행한다.

{% gist mhyeon-lee/e3cf3031818c58d3da9cfa5fefbc70d9 MultiLoad.sql %}

직접 IN statement 를 사용해서 JPQL Query를 만드는 것과 같은 SQL 이 실행된다.

* Load entities in muliple batches

Hibernate 는 기본 batch size 를 databse dialect 에 맞춰서 사용한다.
그러나 특정 상황에서 batch size 를 변경해야할 경우 'withBatchSize(int batchSize)' 메소드를 사용할 수 있다.

{% gist mhyeon-lee/bf94e2fe99fd336a43192be2a6481bbd MultiLoadBatch.java %}
{% gist mhyeon-lee/cf215b039b7b85e2f9f59911d5721bda MultiLoadBatch.sql %}

* Don't fetch entities already stored in 1st level cache

JPQL Query 를 사용하여 entity 리스트를 조회하면 Hibernate 는 database 로 부터 대상이 되는 모든 entity 를 조회한 후에 현재 Session 의 1차 Cache 에서 이미 관리중인 entity 인지 체크한다.

'MultiIdentifierLoadAccess interface' 의 'enableSessionCheck(boolean enabled)' 를 true 로 설정하면 Hibernate 가 database 에 Query 를 실행하기 전에 1차 Cache 를 체크하고 파라미터를 결정하게 할 수 있다.  (default : false)

{% gist mhyeon-lee/a6f69fa0d57938b9f11658cb5e32ebe2 MultiLoadCache.java %}
{% gist mhyeon-lee/4137dc7ef86f575efaf72dff222b7962 %}
