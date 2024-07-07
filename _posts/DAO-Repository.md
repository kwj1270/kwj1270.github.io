---
title: Dao vs Repository
author: woozi
date: 2024-03-13 00:00:00 +0800
categories: [Blogging, Tech]
tags: [Architechture, Pattern]
---

# 개요

# 1. DAO(Data Access Object)

## 1.1. DAO 란?

J2EE 의 권장 아키텍처는 다음과 같다.

- 도메인 레이어: EJB Session Bean
- 퍼시스턴스 레이어: EJB Entity Bean

![](/assets//img/blog/dao-repository/1.gif){: width="400" height="300"}

하지만, EJB Entity Bean 은 아래와 같은 문제로 인해 개발자들로부터 많은 외면을 받았다.

- CMP(Container-Managed Persistence): 매핑 제약
- BMP(Bean-Managed Persistence): 비유용성, 오버헤드, 빈약한 EJB QL 지원, 침투적인 인프라스트럭쳐 코드, 복잡도

시간이 지날 수록 EJB Entity Bean 사용 빈도는 줄어 들었고,
선언적으로 트랜잭션을 정의할 수 있는 CMT(Container-Managed Transaction) 기반의 Session Bean 이 사용되기 시작했다.

하지만 Entity Bean 없이 Session Bean 만을 사용하는 형태로 코드가 작성되기에  
비즈니스 로직만을 담아야할, **도메인 레이어인 Session Bean 내부로 Persistence 코드가 침투되기 시작했다.**

이를 해결하기 위해 **Persistence 저장소로 접근하는 모든 로직을 캡슐화하는 별도의 객체를 추가하여**  
**도메인 레이어와 퍼시스턴스 레이어를 명확하게 분리**하는 방법이 제시되었고 이것이 바로 **DAO(Data Access Objet) 이다.**

다시 정의하자면, DAO(Data Access Object)는
비즈니스 로직과 퍼시스턴스 로직의 명확한 분리를 위해  
**퍼시스턴스 로직을 캡슐화**하고 **도메인 레이어에 객체 지향적인 인터페이스를 제공하는 객체**이다.

## 1.2. DAO 기본 개념

| 참조 : https://www.oracle.com/java/technologies/dataaccessobject.html

DAO를 사용하여 데이터 소스에 대한 모든 액세스를 추상화하고 캡슐한다.
DAO는 데이터 소스와의 연결을 관리하여 데이터를 얻고 저장한다.

비즈니스 구성 요소를 다루는 클라이언트(비즈니스 오브젝트)는 DAO 인터페이스를 활용한다.  
DAO 는 **데이터 소스 구현 세부 사항을 완전히 숨기기 때문에**  
클라이언트에 노출하는 인터페이스는 비즈니스 구성 요소에 영향을 주지 않고 다양한 저장 체계에 적응할 수 있다.
**기본적으로 구성 요소와 데이터 소스 간의 어댑터 역할을 한다.**

![](/assets//img/blog/dao-repository/2.png){: width="650" height="300"}

## 1.3. DAO 구성 요소

![](/assets//img/blog/dao-repository/3.png){: width="650" height="500"}

### 1.3.1. BusinessObject

- 비즈니스 개체는 **데이터 클라이언트**를 나타낸다.
- 데이터를 가져오고 저장하기 위해 **데이터 소스에 액세스해야 하는 객체다.**
- 비즈니스 객체는 데이터 소스에 액세스하는 Servlet이나 Helper Bean 외에도  
  Session Bean, Entity Bean 또는 기타 Java 객체로 구현될 수 있습니다.

### 1.3.2. DataAccessObject

- DataAccessObject는 **DAO 패턴의 기본 객체다.**
- DataAccessObject는 Business Object의  
  **기본 데이터 액세스 구현을 추상화하여 데이터 소스에 대한 투명한 액세스를 가능하게 한다.**
- 즉, BusinessObject는 데이터 로드 및 저장 작업을 DataAccessObject에 위임한 것이다.

### 1.3.3. DataSource

- DataSource 구현을 나타낸다.
- RDBMS, OODBMS, XML 리포지토리, 플랫 파일 시스템 등과 같은 **데이터베이스일 수 있다.**
- **다른 시스템(레거시/메인프레임)**, 서비스(B2B 서비스 또는 신용 카드 뷰로) 또는 일종의 리포지토리(LDAP)일 수도 있다.

### 1.3.4. TransferObject(DataTransferObject)

- 데이터 캐리어로 사용되는 전송 객체를 나타낸다.
- DAO 는 DTO를 사용하여 클라이언트에 데이터를 반환할 수 있다.
- DAO 는 또한 DTO 에서 클라이언트로부터 데이터를 수신하여 데이터 소스의 데이터를 업데이트할 수도 있다.

---

# 2. REPOSITORY

## 2.1. REPOSITORY 란?

![](/assets//img/blog/dao-repository/2.gif){: width="650" height="500"}

Repository 는 도메인과 데이터 매핑 계층 사이를 중재하면서  
**인메모리 도메인 객체 컬렉션과 비슷하게 작동하는 패턴이다.**

객체의 컬렉션과 마찬가지로, Repository에도 객체를 추가하거나 제거할 수 있으며  
Repository에 의해 **캡슐화되는 매핑 코드가 적절한 작업을 내부적으로 처리한다.**  
클라이언트 객체는 **쿼리 사양을 선언적으로 구성한 다음 이를 제출해** 요건을 충족하는지 확인할 수 있다.  
요건이 충족한다면 Repository 는 조건에 알맞는 객체를 반환한다.

Repository는 데이터 저장소에 저장된 객체의 집합과 이를 대상으로 수행하는 작업을  
개념상으로 캡슐화해 **지속성 계층에 대한 좀 더 객체지향적인 관점을 제공한다.**  
또한, 도메인과 데이터 매핑 계층 간의 깔끔한 분리와 단방향 의존성의 목표를 달성하도록 지원한다.

## 2.2. 작동 원리

Repository 는 여러 다른 패턴들을 사용하는 매우 정교한 패턴이다.  
간단한 객체지향형 데이터베이스처럼 보이기도 하며, [Query Objet](https://martinfowler.com/eaaCatalog/queryObject.html)와 매우 비슷하다.  
Query Objet 와 함께 활용활 경우 ORM 계층의 실용성을 비교적 손쉽게 상당 수준으로 향상시킬 수 있다.

Repository 를 활용하는 클라이언트 객체는, '식별 조건 값'을 던져 객체를 조회할 수 있다.  
이는 도메인 객체로 이뤄진 간단한 인메모리 컬렉션을 사용하는 코드와 매우 비슷하다.  
**겉으로는 객체의 컬렉션처럼 보일 수 있지만, 수십만 개의 레코드를 포함하는 테이블과 매핑될 수도 있다.**

Repositorty 는 **데이터 매퍼의 특화된 검색기 메서드**를 **사양 기반 방식의 객체 선택으로 대체한다.**  
클라이언트 코드가 조건을 구성하고 Repository 에 전달해 **조건과 일치**하는 객체를 선택하도록 요청한다.  
클라이언트 코드의 관점에서 **'쿼리 실행'이라는 개념이 아닌, '쿼리 사양 충족'하는 객체가 선택된다.**  
학술적인 구분이지만, Repository 의 선언식 객체 상호 작용을 잘 나타내는 동작이다.

Repository 는 내부적으로 **메타데이터 매핑**과 **Query Object** 를 결합해 조건 기준으로 SQL 코드를 생성한다.  
조건이 자신을 쿼리로 추가할 수 있는지 여부, Query Object가 조건 객체를 통합하는 방법,  
메타 데이터 매핑이 상호작용을 제어하는 방법 등, **모두 구현 세부사항에 속하는 내용이다.**  
(Repository 자체가 본인에게 들어온 메시지를 처리할지 말지를 판단하는 형태를 의미한다)

그리고 Repository 가 **반환하는 객체 원본은 관계형 데이터베이스 데이터 객체가 아니어도 된다.**  
Repository 는 특화된 전략 객체를 통해 데이터 매핑 컴포넌트를 즉시 대체할 수 있도록 지원하므로  
다른 객체 원본을 사용하더라도 전혀 문제가 되지 않는다.**(NoSQL 심지어 Map 같은 자료구조도 괜찮다.)**  
DDD 에서의 내용을 일부 참조하자면 [Entity](https://martinfowler.com/bliki/EvansClassification.html) 를 반환하고 그 구현은 어떤 데이터 스토어든 상관 없다는 것이다.

Repository 는 쿼리를 집중적으로 사용하는 코드의 가독성과 명확성을 개선하는 좋은 매커니즘이다.  
적절한 Repository 로 기준을 제출하는 작업은 한 두 줄의 코드로 간단하게 처리가 가능하게 해주기 때문이다.

## 2.3. 사용 시점

다양한 유형의 도메인 객체와 다수의 쿼리가 사용되는 대규모 시스템에서 사용하면  
모든 쿼리를 처리하는 데 필요한 코드의 양을 줄일 수 있다.

Repository 는 쿼리를 순수한 객체지향 방식으로 수행할 수 있게  
객체 지향방식으로 수행할 수 있게 캡슐화하는 사양 패턴을 촉진한다.

즉, 특정한 사례에 맞기 쿼리 객체를 설정하는 코드를 제거하고.  
클라이언트 입장에서는 SQL 을 다루지 않고 순수한 객체의 관점에서 코드를 작성할 수 있다.

그리고 여러 데이터 원본을 사용하는 상황에서 Reposiory 는 확실한 역할을 할 수 있다.  
객체지향을 추구하면, 루트 객체를 기준으로 객체 그래프가(객체 참조) 형성된다.  
Repository 는 루트 객체를 반환하지만, 참조 연결된 객체들도 조합하여 이를 함께 반환해준다.  
또한, 이 과정에서 클라이언트는 모르지만 캐시와 같은 특정 데이터를 메모리에 저장시킬 수도 있다.

---

# 3. DAO vs REPOSITORY

DAO 는 J2EE 환경에서 Entity Bean의 퍼시스턴스 지원 기능을 대체하는 동시에  
비즈니스 로직과 퍼시스턴스 로직의 명확한 분리를 위해 퍼시스턴스 로직을 캡슐화하고  
**도메인 레이어에 객체 지향적인 인터페이스를 제공하는 객체이다.**  
다만 DAO 의 경우 **데이터 스토어 테이블(레코드)과 1대1 매핑되는 개념을 내포하고 있다.**

반면 Repository 는 **Enttiy 를 기반으로 하는 컬렉션과 비슷한 처리를 해주는 객체이며**  
내부 구현은 RDB 가 아니어도 상관없고, 여러 데이터 스토어의 집합체로 동작할 수도 있다.  
**쉽게 비유하자면, Repository 를 구현하기 위해 여러 DAO 를 내부 참조 객체로 사용할 수도 있다는 뜻이다.**

보다 자세한 설명을 위해, [백명석님이 정리한 글](https://github.com/msbaek/memo/blob/master/dao-vs-repository.md)을 통해 정리해보고자 한다.

- DAO

  - 데이터베이스의 CRUD 쿼리와 1:1 매칭되는 세밀한 단위의 오퍼레이션을 제공
  - 제공하는 오퍼레이션이 REPOSITORY 가 제공하는 오퍼레이션보다 더 세밀하다.
  - 데이터베이스 뿐만 아니라 B2B, LDAP, 메인 프레임, 레거시 시스템과 같은  
    다양한 종류의 외부 시스템과의 상호작용을 캡슐화하기 위해 사용될 수 있다.
    - 외부 시스템에 대한 GATEWAY 역할
  - 퍼시스턴스 레이어에 속한다.
  - TRANSACTION SCRIPT 패턴과 함께 사용된다.
  - 테이블 별로 하나의 DAO를 만든다(TABLE DATA GATEWAY 패턴) Entity Bean을 DAO로 대체하면서  
    테이블 당 하나의 Entity Bean을 만들어야 했던 EJB의 제약을 그대로 답습 추측한다.
  - **데이터베이스에 대한 의존도를 캡슐화할 수 있다고 하더라도 테이블 변경에 대해서는 투명하지 않는다.**
  - TABLE DATA GATEWAY 패턴에 따라 데이터베이스 테이블 별로 생성되는 것이 일반적이며  
    퍼시스턴스 레이어에 대한 FACADE 역할을 수행한다.
  - 식별 과정은 도메인 규칙 보다는 데이터베이스 테이블의 단위에 영향을 받는 경향이 강하다.

- REPOSITORY
  - 메모리에 로드된 객체 컬렉션에 대한 집합 처리를 위한 인터페이스를 제공
  - 제공하는 하나의 오퍼레이션이 DAO의 여러 오퍼레이션에 매핑되는 것이 일반적이다.
    - **내부에서 다수의 DAO를 호출하는 방식으로 구현할 수 있다. => [예제](https://www.baeldung.com/java-dao-vs-repository)**
  - 객체 컬렉션 처리에 관한 책임만을 가지고 있으며 DDD에서 외부 시스템과의 상호작용은 별도의 SERVICE가 담당한다.
  - 도메인 레이어에 속한다.
  - **AGGREGATE ROOT(또는 Entry Point)와 관련된 오퍼레이션만을 포함하며 테이블 변경에 대해서 투명하다.**
  - DOMAIN MDOEL 패턴과 함께 사용된다.
  - **도메인 모델링 단계에서 하나의 개념 단위로 묶인 AGGREGATE를 도출하는 과정에서 자동으로 식별된다.**

# 4. 참조

- [오라클 DAO 패턴 공식 문서](https://www.oracle.com/java/technologies/dataaccessobject.html)
- [엔티티빈 소개하기 - MonAmi 님](https://m.blog.naver.com/st95041/40004696284)
- [엔티티 클래스 설계와 퍼시스턴스 프레임워크 - 정상혁님](https://github.com/benelog/entity-dev/blob/38532634f8c71b1a368f5261edd69843a73db5ed/src/index.adoc#repository-vs-dao)
- [DAO vs REPOSTIROY - 엄범님](https://umbum.dev/2013/)
- [Query Object - 마틴파울러](https://martinfowler.com/eaaCatalog/queryObject.html)
- [Repository - 마틴파울러](https://martinfowler.com/eaaCatalog/repository.html)
- [DAO와 REPOSITORY 논쟁 - 백명석](https://github.com/msbaek/memo/blob/master/dao-vs-repository.md)
