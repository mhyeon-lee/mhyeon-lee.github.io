---
layout: post
title:  "[Hibernate] load-by-multiple-id API 를 사용할때 알아야할 것들"
date:   2016-08-05 10:55:52 +0900
categories: Hibernate
---
Hibernate 5.1 Release 에 추가된 load-by-multiple-id API 는 여러가지로 유용하게 사용할 수 있다. <br />
[Hibernate 5.1 Release](http://in.relation.to/2016/02/10/hibernate-orm-510-final-release/)
그럼에도 불구하고, 아직은 사용하지 않기를 권한다.

이 글에 대한 결론 부터 이야기하고 내용을 풀어보겠다.

**load-by-multiple-id API 는 여러 옵션에 따라서 기대되는 실행 결과가 달라진다. 그러니 옵션들과 그에 따른 영향들을 잘 모르겠으면 차라리 사용하지 말자. JPQL 로 충분하다. (현재 버전 Hibernate 5.2.2)**

아래 글에 API 에 대한 간단한 소개가 있어 이에 대한 요약과 이외에 조심해야 되는 부분, 그리고 5.2.2 에서 추가되는 옵션에 대해서 간단히 정리하고자 한다. <br /> [How to fetch muliple entities by id with Hibernate 5](http://www.thoughts-on-java.org/fetch-multiple-entities-id-hibernate/)

## How to fetch muliple entities by id with Hibernate 5 요약

JPA 나 Hibernate 5.1 이전 버전를 사용할 때 database 에서 다수의 entity들을 조회하기 위한 방법으로 2가지를 생각할 수 있다.

1. 조회하고자 하는 id 각각을 EntityManager.find 메소드를 사용하여 Query를 각각 실행한다.

   > 조회하고자 하는 Entity 가 많다면 너무 많은 Query 를 실행하게 되서 Application 의 성능 저하가 발생할 수 있다.
 
2. IN statement를 사용하여 query 를 만들어서 실행한다.

   {% gist mhyeon-lee/3cbecf69967a624697b178a99b299882 JpaQuery.java %}

   > Oracle과 같은 몇개의 database 는 IN statement의 파라미터 갯수에 제한을 둔다. <br />
   > 많은 엔티티를 한번의 batch 로 조회하게 되면 성능 이슈가 발생할 수 있다. <br />
   > Hibernate 1차 Cache(Session) 에 이미 올라와 있는 엔티티를 체크하지 않고 모든 엔티티를 database 에서 조회한다. <br />
   
**Load multiple entities by their primary key**

Hibernate 5.1 은 다수의 엔티티들을 한번의 API 실행하고 위와 같은 문제점들을 회피할 수 있는 새로운 API 를 제공한다.

{% gist mhyeon-lee/45da2f5d6a90e981e67401872c43b587 MultiLoad.java %}

Hibernate Session 에서 위와 같은 API 를 사용하면 id 가 1L, 2L, 3L 인 PersionEntity 클래스를 조회할 수 있다.
Hiberante 는 3개의 Id 를 파라미터로 IN statement Query 를 실행한다.

{% gist mhyeon-lee/e3cf3031818c58d3da9cfa5fefbc70d9 MultiLoad.sql %}

직접 IN statement 를 사용해서 JPQL Query를 만드는 것과 같은 SQL 이 실행된다.

**Load entities in muliple batches**

Hibernate 는 기본 batch size 를 databse dialect 에 맞춰서 사용한다.
그러나 특정 상황에서 batch size 를 변경해야할 경우 `withBatchSize(int batchSize)` 메소드를 사용할 수 있다.

{% gist mhyeon-lee/bf94e2fe99fd336a43192be2a6481bbd MultiLoadBatch.java %}
{% gist mhyeon-lee/cf215b039b7b85e2f9f59911d5721bda MultiLoadBatch.sql %}

**Don't fetch entities already stored in 1st level cache**

JPQL Query 를 사용하여 entity 리스트를 조회하면 Hibernate 는 database 로 부터 대상이 되는 모든 entity 를 조회한 후에 현재 Session 의 1차 Cache 에서 이미 관리중인 entity 인지 체크한다.

`MultiIdentifierLoadAccess interface` 의 `enableSessionCheck(boolean enabled)` 를 true 로 설정하면 Hibernate 가 database 에 Query 를 실행하기 전에 1차 Cache 를 체크하고 파라미터를 결정하게 할 수 있다.  (default : false)

> enableSessionCheck(true) 를 설정하면, 이미 load 되어 1차 Cache 에 관리되고 있는 entity 의 id 는 IN statement 에 포함하지 않는다.

{% gist mhyeon-lee/a6f69fa0d57938b9f11658cb5e32ebe2 MultiLoadCache.java %}
{% gist mhyeon-lee/4137dc7ef86f575efaf72dff222b7962 MultiLoadCache.log %}

## Bug? Woking as Designed? (Hibernate v5.1.1)

위에 글에서 `load-by-multiple-id API` 의 특징들에 대해 잘 소개되어 있었다.
여기서 언급되지 않았지만, 사용하면서 경험한 몇가지 특징들을 추가적으로 적어보겠다.

`load-by-multiple-id API` 는 Hibernate 5.1 Release 를 기다리던 이유 중에 하나였다.
비록 JPA 에서 제공되는 API 가 아니라 Hibernate Session 을 unWrap 해야 하지만, 성능 최적화에 활용할 수 있어 기다릴만한 가치가 있었던 feature 였다.
실제 프로젝트에 `load-by-multiple-id API` 를 사용하면서 몇가지 이상한? 동작을 발견할 수 있었다.

**flush 되지 않은 entity 들이 결과에 반영되지 않는다.**

Hibernate 는 기본적으로 JPQL(HQL) Query 를 실행할 때 Session 을 먼저 `flush` 시킨 후에 Query 를 실행한다.
Session의 상태와 관계없이 일관된(consistency) 결과를 낼 수 있도록 database 와 먼저 동기화를 진행하는 것이다.
하지만, `multiLoad API` 는 Query 실행 전에 Session 을 `flush` 시키지 않는다. 이에 따라 일반적(?)이지 않은 결과가 나올때가 있다.

{% gist mhyeon-lee/8e5152369d66640f440286c11ae923e8 MultiLoadWithNotFlushed.java %}
   
id 가 1L 인 PersonEntity 가 생성되어 persist 되었고 지연 쓰기 기능으로 아직 `flush` 되지 않은 상태이다. <br />
multiLoad 메소드에 생성된 id 를 포함해서 조회하면 Query 가 실행되고 결과에 id 가 1L 인 entity 는 존재하지 않는다.
   
`enableSessionCheck == false` 이기 때문에 Session 에 존재하고 있음에도 체크하지 않고, `flush` 도 시키지 않기 때문에 당연히 database 에서 조회한 Query 결과에는 포함되지 않게 된다.
   
{% gist mhyeon-lee/f386ed5d2d839eb46658f3955c317444 MultiLoadWithNotFlushed2.java %}
   
remove 도 마찬가지다. id 1L 로 조회한 엔티티를 remove 했음에도 불구하고, 아직 `flush` 되지 않아 `multiLoad` 에는 결과를 반환한다. 이것도 일반적(?)이지 않게 동작하는 부분이다. <br />
remove 후에 `em.find()` 로 조회하면 아직 1차 Cache 에는 관리되고 있지만, 내부적으로 `REMOVED` 로 mark 되어 결과를 return 하지 않는다. 마찬가지로 remove 후에 JPQL 을 실행하면 flush 를 먼저 실행하여 delete Query 가 수행된 후 select Query 가 실행되기 때문에 마찬가지로 결과를 return 하지 않는다.
`em.find()` 나 JPQL 을 사용하나 내부는 다르게 동작하더라도 일관된 결과를 리턴하지만, `multiLoad` 는 다른 결과를 반환한다.

**enableSessionCheck == true 로 설정해도 일반적(?)이지 않다.**

{% gist mhyeon-lee/d6c36983a55c69da24757a2b5f5548df MultiLoadEnableSessionCheckTrue.java %}

`enableSessionCheck` 를 true 로 설정하면 `multiLoad` Query 를 만들때, Session 을 체크하기 때문에 위의 코드는 persist 된 PersonEntity 를 결과로 반환한다. 기대되는 결과다. <br />

{% gist mhyeon-lee/bc1c2c777188ef2eba441da5504318a8 MultiLoadEnableSessionCheckTest2.java %}

`enabelSessionCheck == true` 이기 때문에 remove 된 entity 가 결과에 반환되지 않을거 같다. 하지만 결과는 id가 1L 인 entity 를 반환한다. `em.find()` 의 경우 위에서 언급했듯이, 1차 Cache 에 존재하더라도 내부적으로 `REMOVED` 로 mark 되어 결과를 return 하지 않는다. 하지만 `multiLoad` 는 Session 에서 체크 후 `REMOVED` mark 상태를 확인하지 않고 entity 를 반환한다. <br />
`enableSessionCheck == true` 라도 일관되지 않은 결과가 나온다.

기존의 `em.find() / JPQL` 의 동작 결과와 일관성을 유지하고 친숙한 동작 방식으로 아래와 같이 고쳐져야 할 것 같았다.

1. `enableSessionCheck` 값은 true 가 default 여야 되지 않을까?
   > `em.find() / session.byId().load()` 의 내부동작과 비교했을때 `session.byMultipleIds().multiLoad()` 는 sessionCheck 를 하는게 default 일것 같았다.

2. `enabelSessionCheck == true` 일때, Entity 의 `REMOVED` 상태는 반환하지 않아야 되지 않을까?
   > `em.find() / session.byId().load()` 에서도 1차 Cache에 entity 가 있어도 `REMOVED` 상태면 반환하지 않는다. `session.byMultipleIds().multiLoad()` 도 마찬가지로 동작해야 될 것 같다.

3. `enableSessionCheck == false` 이면, Query 를 실행시키기 전에 `flush` 를 시켜줘야 되지 않을까?
   > 1차 Cache 를 확인하지 않는 JPQL 실행과 마찬가지로 `flush` 를 먼저 시켜줘야 일관된 결과를 반환할 것 같다.

1번 은 Improvement , 2번 3번은 bug fix 로 생각해서 Issue 를 남기고 Pull-Request 를 보냈다. <br />
[Issue](https://hibernate.atlassian.net/browse/HHH-10984) , 
[Pull-Request](https://github.com/hibernate/hibernate-orm/pull/1489)

1번은 파라미터로 넘어온 id 리스트가 클 경우 세션 체크를 위해 순회하는 오버헤드가 걱정된다.
2번은 `REMOVED` 를 체크해서 반환하지 않는 옵션을 추가하기로 결정 (default = false)
3번은 "Working as Designed" 라면서 Reject (원래 저렇게 되도록 설계했다는 것이다.)

내 Pull-Request 는 reject 시키고, 2번만 자기들이 옵션 추가하는 방향으로 개발해서 Commit했더라.... <br />


## 결론

JPA / Hibernate 에서 조회를 위해 사용하는 `em.find / session.byId().load() / JPQL` 과 다른 결과를 반환하는 점을 발견하고 Bug 라고 판단하여 Issue + Pull-Request 를 보냈지만 "Working as Designed" 로 결론. <br />
2번은 옵션을 추가하여 Hibernate 5.2.2 에 반영되었고, 3번은 아직 남아있다. <br />
따라서 `multiLoad` 를 사용할 때는 JPQL 과 같이 `flush` 하지 않고 remove 된 결과가 반환될 수 있다는 것을 알고 사용해야 한다. (5.2.2 에서 2번은 해결되었다.)

**이런 저런 옵션과 상황에 따라 달라지는 결과가 복잡하고 이해할 수 없다면 multiLoad 를 사용하지 말고 그냥 JPQL 을 사용하는게 속편하다.**


## Hibernate 5.2.2 에서 multiLoad 에 추가된 옵션

5.2.2 에서 `multiLoad` 와 관련하여 2가지 옵션이 추가된다.

1. `enableReturnOfDeletedEntities` (default : false)
    > 2번에 대한 이슈 대안으로 추가된 옵션이다. <br />
    > default 는 false 이며, true 로 변경 시 Deleted 된(non-flushed) entity 도 결과에 반환한다.

2. `enableOrderedReturn` (default : true)
    > 파라미터로 보낸 id 의 순서로 정렬되어 결과 List 를 반환한다. <br />
    > id 파라미터 중 결과가 없는 id 의 index 에는 null 로 셋팅되어 반환한다. (결과를 순회할 때 NPE 를 조심해야 한다.) <br />
    > false 로 변경하면 쿼리 결과 순서로 결과를 반환한다. (`enableSesscionCheck == true` 이면, 1차 Cache 에 존재하는 Entity 들이 결과 List 의 앞에 배치된다.)

[@72e9485](https://github.com/mhyeon-lee/hibernate-orm/commit/72e948514e95cbc2c7e8713a36ed461845d8c89e)
