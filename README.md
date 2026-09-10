# SAYREN Backend

> **일반 구매와 렌탈·구독을 하나의 주문 흐름으로 제공하는 커머스 플랫폼**

SAYREN은 일반 구매 상품과 렌탈·구독 상품을 함께 제공하며,  
**상품 선택 → 요금제 선택 → 장바구니/바로구매 → 주문 → 결제 → 배송 → 회수 → 알림**으로 이어지는
커머스 흐름을 구현한 팀 프로젝트입니다.

본 저장소는 **SAYREN Backend** 소스코드이며,  
팀 프로젝트 저장소를 기반으로 한 개인 Fork Repository입니다.

---

## 📌 Project Overview

| 항목 | 내용 |
| --- | --- |
| 프로젝트명 | SAYREN |
| 프로젝트 유형 | 팀 프로젝트 |
| 개발 기간 | 2025.09 ~ 2025.10 |
| Backend | Java 21, Spring Boot 3.5.5 |
| ORM / Query | Spring Data JPA, QueryDSL |
| Security | Spring Security 6, JWT |
| Database | MariaDB |
| Cache / Auth | Redis |
| Mapping | MapStruct |
| Infrastructure | AWS EC2, RDS, S3 |
| Frontend | Next.js 14 |
| API | REST API |

---

## 🙋‍♂️ My Role

팀 프로젝트에서 아래 도메인의 **Backend 비즈니스 로직, REST API 및 Frontend 연동**을 담당했습니다.

### 담당 도메인

- 요금제 `OrderPlan`
- 장바구니 `Cart`
- 주문 `Order`
- 주문 상품 `OrderItem`
- 주문 이력 `OrderHistory`
- 배송지 `Address`
- 배송 및 회수 `Delivery`
- 알림 `Notification`

### 담당 업무

- 담당 도메인의 Entity / DTO / Mapper / Repository / Service / Controller 구현
- 주문·배송 상태 변경 로직 구현
- 주문 상태 변경 이력 관리
- Spring Event 기반 도메인 후속 처리 연동
- Next.js Frontend와 REST API 연동
- TanStack Query를 이용한 서버 상태 관리 및 캐시 갱신 처리

---

## ✨ Main Features

### 1. 요금제 관리 - OrderPlan

상품 구매 방식을 일반 구매와 렌탈로 구분했습니다.

```text
PURCHASE
RENTAL
```

렌탈 상품의 경우 상품 선택 단계에서 다음과 같은 요금제를 선택할 수 있도록 구성했습니다.

```text
12개월
24개월
36개월
```

상품과 요금제 정보를 주문 생성 시점까지 유지하여  
장바구니와 주문에서 어떤 상품을 어떤 조건으로 선택했는지 식별할 수 있도록 구현했습니다.

---

### 2. 장바구니 - Cart

회원·상품·요금제 조합을 기준으로 장바구니 데이터를 관리했습니다.

#### 주요 기능

- 장바구니 상품 추가
- 회원별 장바구니 조회
- 개별 상품 삭제
- 장바구니 전체 비우기
- 동일 상품 + 동일 요금제 중복 등록 방지
- 구매 상품과 렌탈 상품을 하나의 장바구니에서 관리
- 주문 생성 후 장바구니 정리

#### API

```http
POST   /api/user/cart/add-item
GET    /api/user/cart
DELETE /api/user/cart/delete-item/{id}
DELETE /api/user/cart/clear-item
```

---

### 3. 주문 - Order

장바구니 주문과 바로구매 주문 흐름을 분리하여 구현했습니다.

#### 주요 기능

- 장바구니 기반 주문 생성
- 바로구매 주문 생성
- `Order` / `OrderItem` 생성
- 회원별 주문 목록 조회
- 주문 상세 조회
- 주문 상태 변경
- 주문 취소
- 주문 상태 변경 이력 저장
- 주문 생성 완료 후 후속 이벤트 발행

#### 주요 API

```http
POST /api/user/orders/create
POST /api/user/orders/direct-create

GET  /api/user/orders/my
GET  /api/user/orders/{id}

POST /api/user/orders/{id}/paid
POST /api/user/orders/{id}/cancel
```

---

## 💰 주문 가격 Snapshot

상품의 현재 가격만 참조할 경우 상품 가격이 변경되었을 때  
과거 주문 금액까지 영향을 받을 수 있는 문제가 있습니다.

이를 방지하기 위해 주문 생성 시점의 상품 가격을

```text
OrderItem.productPriceSnapshot
```

으로 별도 저장했습니다.

```text
상품 현재 가격
      ↓
주문 생성
      ↓
productPriceSnapshot 저장
      ↓
이후 상품 가격 변경
      ↓
기존 주문 가격에는 영향 없음
```

이를 통해 과거 주문 데이터의 금액 정합성을 유지했습니다.

---

## 📦 배송지 - Address

Checkout 과정에서 회원의 배송지를 사용할 수 있도록 배송지 도메인을 구현했습니다.

#### 주요 기능

- 배송지 등록
- 배송지 조회
- 배송지 수정
- 배송지 삭제
- 기본 배송지 설정
- 기존 배송지 선택
- 주문 과정에서 신규 배송지 등록

주문 생성 시 배송지 정보를 주문 데이터와 연결하여  
이후 배송 생성 과정에서 사용할 수 있도록 구성했습니다.

---

## 🚚 배송 / 회수 - Delivery

주문 이후 실제 상품의 배송 및 렌탈 상품 회수 흐름을 관리합니다.

### 배송 상태

```text
READY
  ↓
SHIPPING
  ↓
DELIVERED
```

| 상태 | 의미 |
| --- | --- |
| `READY` | 배송 준비 |
| `SHIPPING` | 배송 중 |
| `DELIVERED` | 배송 완료 |

### 회수 상태

```text
RETURN_READY
      ↓
IN_RETURNING
      ↓
RETURNED
```

| 상태 | 의미 |
| --- | --- |
| `RETURN_READY` | 회수 준비 |
| `IN_RETURNING` | 회수 진행 중 |
| `RETURNED` | 회수 완료 |

배송과 회수의 상태 전환 규칙을 분리하여  
정상적인 순서에 따라 상태가 변경되도록 구성했습니다.

---

## 🔗 Payment → Delivery 연동

결제 도메인은 팀원이 담당했으며,  
담당 도메인과의 연동은 결제 상태 변경 이벤트를 기준으로 처리했습니다.

```text
Payment
   │
   │ PAID
   ▼
PaymentStatusChangedEvent
   │
   ▼
Delivery 생성
   │
   ▼
READY
```

결제가 완료되면 주문 상품을 기준으로 `Delivery`와 `DeliveryItem`을 생성하고  
초기 배송 상태를 `READY`로 설정하도록 연결했습니다.

이를 통해 Payment Service와 Delivery Service를 직접 강하게 결합하기보다  
상태 변경 이벤트를 기준으로 후속 로직을 수행하도록 구성했습니다.

---

## 🔔 Notification

주문·결제·배송 상태 변화에 따라 사용자 알림으로 이어질 수 있도록  
Notification 영역과 이벤트 기반 연동 구조를 구성했습니다.

```text
결제 상태 변경
      ↓
배송 생성 / 상태 변경
      ↓
DeliveryStatusChangedEvent
      ↓
Notification
```

도메인 간 직접 호출을 줄이고  
상태 변화에 따른 후속 처리를 이벤트 중심으로 분리했습니다.

---

## 🔄 Event Driven Flow

SAYREN에서 제가 담당한 도메인은 단순 CRUD뿐 아니라  
**상태 변경 → 이력 기록 → 이벤트 발행 → 후속 처리** 흐름을 고려하여 구성했습니다.

```text
상태 변경 요청
      ↓
StatusChanger
      ↓
Entity 상태 변경
      ↓
DB 반영
      ↓
HistoryRecorder
      ↓
EventPublisher
      ↓
후속 도메인 처리
```

대표적으로 사용한 이벤트 흐름은 다음과 같습니다.

- `OrderPlacedEvent`
- `PaymentStatusChangedEvent`
- `DeliveryStatusChangedEvent`

이를 통해 주문·결제·배송·알림 도메인의 직접적인 결합을 줄이고  
각 도메인의 책임을 분리했습니다.

---

## 🏗 Backend Architecture

Controller → Service → Repository / Mapper 계층으로 역할을 분리했습니다.

```text
Client
  │
  ▼
Controller
  │
  ▼
Service
  │
  ├── Business Logic
  ├── StatusChanger
  ├── HistoryRecorder
  └── EventPublisher
  │
  ▼
Repository / Mapper
  │
  ▼
MariaDB
```

### Controller

- REST API 요청 / 응답 처리
- DTO 전달
- 인증 사용자 요청 처리

### Service

- 핵심 비즈니스 로직
- 주문 생성
- 장바구니 처리
- 배송 생성
- 상태 변경

### Repository

- Spring Data JPA 기반 Entity 영속성 처리

### Mapper

- Entity ↔ DTO 변환
- MapStruct 활용

### Event

- 도메인 상태 변경 이후 필요한 후속 작업 연결

---

## 🗃 Main Domain Structure

```text
Member
 │
 ├── CartItem
 │    ├── Product
 │    └── OrderPlan
 │
 ├── Order
 │    │
 │    ├── OrderItem
 │    │    ├── Product
 │    │    └── OrderPlan
 │    │
 │    ├── Address
 │    └── OrderHistory
 │
 └── Delivery
      │
      ├── Address
      │
      └── DeliveryItem
             │
             └── OrderItem
```

---

## 📊 Order Status

```text
PENDING
   ↓
PAID
```

취소 시에는 주문 상태를 `CANCELED`로 변경합니다.

| 상태 | 의미 |
| --- | --- |
| `PENDING` | 주문 생성 / 결제 대기 |
| `PAID` | 결제 완료 |
| `CANCELED` | 주문 취소 |

주문의 현재 상태만 저장하는 것이 아니라  
`OrderHistory`를 통해 상태 변경 이력을 별도로 관리했습니다.

---

## 🛠 Tech Stack

### Backend

![Java](https://img.shields.io/badge/Java_21-007396?style=for-the-badge&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot_3.5.5-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring_Security-6DB33F?style=for-the-badge&logo=springsecurity&logoColor=white)
![JPA](https://img.shields.io/badge/Spring_Data_JPA-6DB33F?style=for-the-badge)
![QueryDSL](https://img.shields.io/badge/QueryDSL-0769AD?style=for-the-badge)
![MapStruct](https://img.shields.io/badge/MapStruct-EF6C00?style=for-the-badge)

### Database / Cache

![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)

### Infrastructure

![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=for-the-badge&logo=amazonec2&logoColor=white)
![AWS RDS](https://img.shields.io/badge/AWS_RDS-527FFF?style=for-the-badge&logo=amazonrds&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS_S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)

### Frontend Integration

![Next.js](https://img.shields.io/badge/Next.js_14-000000?style=for-the-badge&logo=nextdotjs&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![TanStack Query](https://img.shields.io/badge/TanStack_Query-FF4154?style=for-the-badge&logo=reactquery&logoColor=white)
![Redux Toolkit](https://img.shields.io/badge/Redux_Toolkit-764ABC?style=for-the-badge&logo=redux&logoColor=white)

---

## 💡 Technical Challenges & Solutions

### 1. 상품 가격 변경으로 인한 과거 주문 금액 변경 문제

**Problem**

주문이 Product의 현재 가격만 참조하면  
상품 가격 수정 시 기존 주문 금액까지 영향을 받을 수 있습니다.

**Solution**

주문 생성 시점의 가격을 `OrderItem.productPriceSnapshot`으로 저장했습니다.

**Result**

상품 가격이 이후 변경되더라도  
이미 생성된 주문의 가격은 주문 당시 금액으로 유지할 수 있도록 했습니다.

---

### 2. 주문·결제·배송 간 강한 결합

**Problem**

Order, Payment, Delivery Service가 서로 직접 호출할 경우  
도메인 간 의존성이 증가하고 변경 영향 범위가 커질 수 있습니다.

**Solution**

Spring Event를 사용하여 상태 변경을 기준으로 후속 작업을 연결했습니다.

```text
Payment PAID
     ↓
PaymentStatusChangedEvent
     ↓
Delivery 생성
```

**Result**

도메인별 책임을 분리하면서  
상태 변화에 필요한 후속 기능을 연결할 수 있었습니다.

---

### 3. 배송과 회수의 서로 다른 상태 전환 규칙

**Problem**

배송과 회수는 같은 Delivery 영역에서 관리하지만  
진행 가능한 상태 흐름이 서로 다릅니다.

**Solution**

배송:

```text
READY → SHIPPING → DELIVERED
```

회수:

```text
RETURN_READY → IN_RETURNING → RETURNED
```

로 상태 흐름을 분리했습니다.

**Result**

배송과 회수의 생명주기를 명확하게 관리할 수 있도록 구성했습니다.

---

### 4. 주문 상태 변경 이력 추적

**Problem**

Order Entity에 현재 상태만 저장하면  
이전 상태와 변경 과정을 확인하기 어렵습니다.

**Solution**

`OrderHistory`를 별도 관리하여  
주문 상태 변경 이력을 저장했습니다.

**Result**

현재 상태와 상태 변경 기록을 분리하여  
주문의 변경 과정을 추적할 수 있도록 했습니다.

---

### 5. Frontend Mutation 이후 이전 데이터가 남는 문제

**Problem**

장바구니 삭제 또는 주문 처리 이후에도  
기존 Query Cache가 남아 이전 데이터가 화면에 표시되는 문제가 있었습니다.

**Solution**

TanStack Query Mutation 성공 이후 관련 Query를 invalidate하도록 처리했습니다.

```text
Mutation
   ↓
Success
   ↓
invalidateQueries
   ↓
Server Data Refetch
   ↓
UI Synchronization
```

**Result**

서버 상태와 Frontend 화면 상태를 일관되게 유지하도록 개선했습니다.

---

## 🖥 Frontend Integration

담당 Backend REST API를 Next.js Frontend와 연결하여 다음 화면 및 기능과 연동했습니다.

- 상품 상세 요금제 선택
- 장바구니
- 장바구니 삭제 / 전체 비우기
- 바로구매
- Checkout
- 기존 배송지 선택
- 신규 배송지 등록
- 주문 생성
- 주문 내역 조회
- 관리자 배송 및 회수 상태 관리

Frontend에서는 Axios와 TanStack Query를 사용하여  
REST API 요청과 서버 상태를 관리했습니다.

---

## 📸 Project Screens

프로젝트의 실제 동작 화면은 Portfolio에서 확인할 수 있습니다.

### Portfolio

👉 **[SAYREN Portfolio 바로가기](https://seongjun-portfolio.vercel.app)**

Portfolio에서 다음 화면을 확인할 수 있습니다.

- SAYREN 메인 화면
- 상품 및 렌탈 요금제 선택
- 장바구니
- 주문 / Checkout
- 주문 생성
- 배송지 선택
- 신규 배송지 등록
- 관리자 배송 관리

---

## 🔗 Related Repository

### Backend

현재 Repository: `sayren_back`

### Frontend

👉 **[SAYREN Frontend Repository](https://github.com/songseongjun/sayren_front)**

---

## 👥 Team Project

본 프로젝트는 **팀 프로젝트**이며,  
본 Repository는 팀 프로젝트 Repository를 기반으로 한 개인 Fork Repository입니다.

제가 직접 담당한 주요 영역은 다음과 같습니다.

```text
OrderPlan
Cart
Order
OrderItem
OrderHistory
Address
Delivery
DeliveryItem
Notification
Frontend REST API Integration
```

결제 `Payment`, 구독 `Subscribe`, 상품 `Product`, 회원 `Member` 등은  
팀원 담당 도메인이며 필요한 부분을 이벤트와 Entity 관계를 통해 연동했습니다.

---

## 🎯 What I Learned

SAYREN 프로젝트를 통해 단순 CRUD 구현을 넘어 다음 내용을 경험했습니다.

- 여러 도메인이 연결되는 주문 비즈니스 흐름 설계
- Entity 관계 설계 및 JPA 기반 데이터 처리
- 주문 시점 데이터 Snapshot 관리
- 상태 전환과 상태 변경 이력 관리
- Spring Event 기반 도메인 간 연동
- JWT 인증 환경의 회원별 API 처리
- Backend REST API와 Next.js Frontend 연동
- TanStack Query를 이용한 서버 상태 관리
- 팀 프로젝트에서 담당 도메인과 다른 도메인 간 인터페이스 조율

특히 기능을 단순히 동작시키는 것에서 끝내지 않고,  
**"데이터가 어떤 흐름으로 이동하고, 상태가 왜 변경되며, 다른 도메인에 어떤 영향을 주는가"**를
고려하며 구현하는 경험을 할 수 있었습니다.
