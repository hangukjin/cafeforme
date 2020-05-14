ID : 5team@gkn2019hotmail.onmicrosoft.com  
PW : 6400

![image](https://user-images.githubusercontent.com/28293389/81536861-34084480-93a7-11ea-866f-c09e24ef9412.png)

# 개인 과제 : 한국진 - 카페포미(5조)

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [카페포미](#---)
  - [조직](#조직)
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
    - [AS-IS 조직 (Horizontally-Aligned)]
    - [TO-BE 조직 (Vertically-Aligned)]
    - [Event Storming 결과-1차, 2차]
    - [2차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증]
    - [비기능적 요구사항(검증)]
    - [헥사고날 아키텍처 다이어그램 도출]
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - 장애 격리
    - [비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - 구현
  - 운영과 Retirement

# 새로운 조직 : 경영 관리
- 경영 관리 KPI : 매출액 집계 및 보고

# 추가 서비스 시나리오
- [기능적 요구사항]
1. Delivery에서 수락 되어 제작 된 상품과 가격을 수집한다.
1. 매출액 합계를 본사 시스템으로 전송할 수 있다.

- [비기능적 요구사항]
1. 트랜잭션
    - 본사 시스템(외부 API)은 Req/Res로 연결 된다.  Sync 호출 
1. 장애격리
    - 해당 서비스가 작동하지 않아도 기존 시스템에 영향을 주지 않으며 재작동시 기존 주문을 참고하여 집계 한다.  Async (event-driven), Eventual Consistency
1. 성능
    - 매출액 합계는 수시로 조회가 가능해야 한다.  CQRS

# 분석/설계

## 기존 조직
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)

## 신규 조직 추가 (경영 관리팀)
  ![image](https://user-images.githubusercontent.com/28293389/81789840-80868800-953f-11ea-9be8-223d8ea1e08c.png)


## Event Storming 결과

### 기존 모형
![image](https://user-images.githubusercontent.com/63624054/81637640-cd8c3080-9451-11ea-8446-46ec36b17108.png)

### 신규 모형
![image](https://user-images.githubusercontent.com/28293389/81842274-b438d080-9586-11ea-9150-37d2d959a1bc.png)


    - Accounting 과 Headquaters 추가
  
  
  

### 기능적/비기능적 요구사항을 커버하는지 검증

기능적 요구사항(검증)
![image](https://user-images.githubusercontent.com/28293389/81842650-51940480-9587-11ea-8f63-8f04c4a4dce6.png)

1. Delivery에서 수락 되어 제작 된 상품과 가격을 수집하고 동시에 매출액 합계를 본사 시스템으로 전송 된다.

 비기능적 요구사항(검증)
1. 트랜잭션
    - 본사 시스템(외부 API)은 Req/Res로 연결 된다.  Sync 호출 
1. 장애격리
    - 해당 서비스가 작동하지 않아도 기존 시스템에 영향을 주지 않으며 재작동시 기존 주문을 참고하여 집계 한다.  Async (event-driven), Eventual Consistency
1. 성능
    - 매출액 합계는 수시로 조회가 가능해야 한다.  CQRS


## 업무 프로세스 흐름도
![image](https://user-images.githubusercontent.com/28293389/81843985-607bb680-9589-11ea-890d-fd8ecc273edc.png)


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/28293389/81883964-917fd980-95d1-11ea-9b7e-60e2cf5fc82b.png)

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
# 8081
cd customer
mvn spring-boot:run

# 8084
cd delivery
mvn spring-boot:run 

# 8088
cd gateway
mvn spring-boot:run  

# 8085
cd account
mvn spring-boot:run  

# 8086
cd headquaters
python command-handler.py
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Delivery 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.

```
@Entity
@Table(name="Accounting_table")
public class Accounting {

    @Id
    private String yearmonth;
    private Double salesSum;
    private Double salesQty;
    private Double orderCount;

        public String getYearmonth() {
        return yearmonth;
    }

    public void setYearmonth(String yearmonth) {
        this.yearmonth = yearmonth;
    }

    public Double getSalesSum() {
        return salesSum;
    }

    .
    .
    .

    public Double getOrderCount() {
        return orderCount;
    }

    public void setOrderCount(Double orderCount) {
        this.orderCount = orderCount;
    }

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 CrudRepository 를 적용하기로 정하였으며 서비스 특성 상 Sort 된 데이터를 조회하는 일이 빈번하여 PagingAndSortingRepository 를 적용하였다.
```
package local;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface AccountingRepository extends PagingAndSortingRepository<Accounting, String>{
}
```
## 적용 후 REST API 의 테스트

### Customer 서비스의 주문 처리
```bash
http POST gateway:8080/orders product="IceAmericano" qty=1 price=2000
```
```
{
    "_links": {
        "order": {
            "href": "http://Customer:8080/orders/1"
        },
        "self": {
            "href": "http://Customer:8080/orders/1"
        }
    },
    "cannotOrderCanceled": false,
    "price": 2000,
    "product": "IceAmericano",
    "qty": 1,
    "status": null
}
```

### Delivery 서비스의 승인 처리
```bash
http PUT gateway:8080/deliveries/1 status="receive" product="IceAmericano" qty=1 price=2000
```
```
{
    "_links": {
        "delivery": {
            "href": "http://Delivery:8080/deliveries/1"
        },
        "self": {
            "href": "http://Delivery:8080/deliveries/1"
        }
    },
    "price": 2000,
    "product": "IceAmericano",
    "qty": 1,
    "status": "receive"
}
```

### Account 상태 확인 및 View
```bash
http gateway:8080/accountings
```
```
"accountings": [
        {
            "_links": {
                "accounting": {
                    "href": "http://account:8080/accountings/20200514"
                },
                "self": {
                    "href": "http://account:8080/accountings/20200514"
                }
            },
            "orderCount": 1.0,
            "salesQty": 1.0,
            "salesSum": 2000.0
        }
    ]
```
```bash
http gateway:8080/accountingStatisticses
```
```
"accountingStatisticses": [
        {
            "_links": {
                "accountingStatistics": {
                    "href": "http://account:8080/accountingStatisticses/20200514"
                },
                "self": {
                    "href": "http://account:8080/accountingStatisticses/20200514"
                }
            },
            "orderCount": 1.0,
            "salesQty": 1.0,
            "salesSum": 2000.0
        }
    ]
```

## 동기식 호출 과 Fallback 처리

- 본사 시스템(Headquarters)은 외부 시스템으로 가정하였으므로 매출액을 전송할 때 REST POST 방식으로 전송하였다.

```java
# (account) Accounting.java

@PostPersist
@PostUpdate
public void onPrePersist(){
    SalesTransferred salesTransferred = new SalesTransferred();
    BeanUtils.copyProperties(this, salesTransferred);
    salesTransferred.setCafeId(92);

    this.salesTransfer(salesTransferred);
}

@RequestMapping(method= RequestMethod.POST, path="/headquarters")
public void salesTransfer(@RequestBody SalesTransferred salesTransferred){
}
```

## 장애 격리
```
# Math.random() > 0.5 이면 결제 서비스가 정상 Response로 가정

# 주문 후 결제가 자동으로 진행
http POST customer:8080/orders product="IceAmericano" qty=1 price=2000    # Success
http POST customer:8080/orders product="IceAmericano2" qty=1 price=2000   # Fail

# 결제 결과 확인
http customer:8080/orderStatuses/1   # Status="pay_success"
http customer:8080/orderStatuses/2   # Status="pay_fail"
```


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


결제가 이루어진 후에 상점시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 상점 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 결제 승인이 되었다는 도메인 이벤트를 카프카로 송출한다 (Publish)
 
```
PaymentCompleted paymentCompleted = new PaymentCompleted();
BeanUtils.copyProperties(this, paymentCompleted);
paymentCompleted.publish();
```
- 상점 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
@StreamListener(KafkaProcessor.INPUT)
public void wheneverPaymentCompleted_OrderFromOrder(@Payload PaymentCompleted paymentCompleted){

    if(paymentCompleted.isMe()){

        Delivery delivery = new Delivery();
        delivery.setOrderId(paymentCompleted.getOrderId());
        delivery.setProduct(paymentCompleted.getProduct());
        delivery.setQty(paymentCompleted.getQty());
        delivery.setStatus("order_get");

        deliveryRepository.save(delivery);
    }
}
```
실제 구현 시, 점주는 Delivery UI를 통해 주문에 대한 수락/거절을 한다. (Delivery에 대해 PUT)
  
```
@PostUpdate
public void onPostUpdate(){
    if ("receive".equals(this.getStatus())) {
        OrderReceived orderReceived = new OrderReceived();
        BeanUtils.copyProperties(this, orderReceived);
        orderReceived.publishAfterCommit();
    }

    if ("reject".equals(this.getStatus())) {
        OrderRejected orderRejected = new OrderRejected();
        BeanUtils.copyProperties(this, orderRejected);
        orderRejected.publishAfterCommit();
    }
}
```

상점 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 상점시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# 테스트 편의성을 위해 Local에서 진행
# 상점 서비스 (store) 를 잠시 내려놓음 (ctrl+c)

#주문처리
http POST localhost:8081/orders product="IceAmericano" qty=1 price=2000 

#주문상태 확인
http localhost:8081/orderStatuses/1     # 주문 상태 : pay_success

#상점 서비스 기동
cd delivery
mvn spring-boot:run

#상점에서 주문 확인
http localhost:8084/deliveries/1        # 주문 상태 : order_get

#상점 주인의 수락/거절 진행
http PUT localhost:8084/deliveries/1 status="receive"

#주문상태 확인
http localhost:8081/orderStatuses/1     # 주문 상태 : order_received
```


# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure(Dev)를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 azure-pipelines.yml 에 포함되었다.


### 오토스케일 아웃

- Customer/Delivery 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다.  
  
    Customer : CPU 사용량이 50프로를 넘어서면 replica를 10개까지 늘려준다(최소 1)  
    Delivery : CPU 사용량이 50프로를 넘어서면 replica를 5개까지 늘려준다(최소 1)
```
kubectl autoscale deploy customer --min=1 --max=10 --cpu-percent=50
kubectl autoscale deploy delivery --min=1 --max=5 --cpu-percent=50
```
- 부하 테스트를 위해 cpu-percent를 임시 조정
```
kubectl autoscale deploy customer --min=3 --max=10 --cpu-percent=5
```

- siege 를 통해 test를 진행하였으나 충분한 부하가 걸리지 않아 hpa는 확인하지 못하였다.
```
NAME                            READY   STATUS    RESTARTS   AGE
pod/customer-85f968c94c-b254s   1/1     Running   0          15m
pod/customer-85f968c94c-k5pq4   1/1     Running   0          14m
pod/customer-85f968c94c-pgwd7   1/1     Running   0          15m
pod/delivery-78f4bb9f9b-7h42s   1/1     Running   0          68m
pod/gateway-675c9b584d-tr7cv    1/1     Running   0          17h
pod/httpie                      1/1     Running   1          18h
```
```
Transactions:                  12858 hits
Availability:                 100.00 %
Elapsed time:                 119.46 secs
Data transferred:               3.42 MB
Response time:                  0.43 secs
Transaction rate:             107.63 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                   46.27
Successful transactions:       12858
Failed transactions:               0
Longest transaction:            2.44
Shortest transaction:           0.00
```



## 무정지 재배포

* readiness 설정
```
(azure-pipelines.yaml)

readinessProbe:
httpGet:
    path: /actuator/health
    port: 8080
initialDelaySeconds: 10
timeoutSeconds: 2
periodSeconds: 5
failureThreshold: 10
```

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://customer:8080/orders POST {"product": "iceAmericano", "qty":1, "price":1000}'


```

- Pipeline을 통한 새버전으로의 배포 시작 - UI
![image](https://user-images.githubusercontent.com/28293389/81762438-94ae9300-9507-11ea-8e94-8a761529ecac.png)

```
NAME                            READY   STATUS        RESTARTS   AGE
pod/customer-5d479469b9-5wlxp   0/1     Terminating   0          55m
pod/customer-85f968c94c-b254s   1/1     Running       0          87s
pod/customer-85f968c94c-k5pq4   1/1     Running       0          27s
pod/customer-85f968c94c-pgwd7   1/1     Running       0          55s
pod/delivery-78f4bb9f9b-7h42s   1/1     Running       0          54m
pod/gateway-675c9b584d-tr7cv    1/1     Running       0          17h
pod/httpie                      1/1     Running       1          18h
```


- seige 의 화면으로 넘어가서 Availability 확인
```
defaulting to time-based testing: 120 seconds
** SIEGE 3.0.8
** Preparing 100 concurrent users for battle.
The server is now under siege...
Lifting the server siege...      done.

Transactions:                  12976 hits
Availability:                 100.00 %
Elapsed time:                 119.67 secs
Data transferred:               3.45 MB
Response time:                  0.42 secs
Transaction rate:             108.43 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                   45.44
Successful transactions:       12976
Failed transactions:               0
Longest transaction:            2.29
Shortest transaction:           0.00
```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


## 구현  

기존의 마이크로 서비스에 수정을 발생시키지 않도록 Inbund 요청을 REST 가 아닌 Event 를 Subscribe 하는 방식으로 구현. 기존 마이크로 서비스에 대하여 아키텍처나 기존 마이크로 서비스들의 데이터베이스 구조와 관계없이 추가됨. 

## 운영과 Retirement

Request/Response 방식으로 구현하지 않았기 때문에 서비스가 더이상 불필요해져도 Deployment 에서 제거되면 기존 마이크로 서비스에 어떤 영향도 주지 않음.

* [비교] 결제 (pay) 외부 서비스의 경우 API 변화나 Retire 시에 주문 (customer) 마이크로 서비스의 변경을 초래함:


## 시연용 프로세스

- 고객이 온라인으로 주문 내역을 만든다.
```
http POST http://customer:8080/orders product="IceAmericano" qty=1 price=2000
```
- 고객이 주문 내역으로 결제한다. (외부 API를 통한 동기식 진행)
```
http http://customer:8080/orderStatuses
```
- 주문 내역이 결제 되면 해당 내역을 매장에서 접수하거나 거절한다.
```
http http://delivery:8080/deliveries
```
- 매장에서 접수하면 커피를 제작하고 고객이 주문을 취소할 수 없다.
```
http PUT http://delivery:8080/deliveries/1 product="IceAmericano" qty=1 status="receive"
```
- 매장에서 거절하면 주문 내역이 취소되고 결제가 환불 된다.
```
http PUT http://delivery:8080/deliveries/1 product="IceAmericano" qty=1 status="reject"
```
- 고객이 주문 내역을 취소를 요청할 수 있다.
```
http PUT http://customer:8080/orders/1 product="IceAmericano" qty=1 status="order_cancel_request"
```

- 매장에서 주문 내역 취소 요청을 받으면 취소가 가능하다면 취소한다.
```
http http://delivery:8080/deliveries
```
- 주문 내역이 취소되면 결제가 환불 된다.
- 매장에서 커피가 제작 되면 고객이 주문 건을 픽업한다.
- 주문 진행 상태가 바뀔 때 마다 SMS로 알림을 보낸다.

## 참고 명령어
- httpie pod 접속
```
kubectl exec -it httpie bin/bash
```
- Order Status 확인
```
http http://customer:8080/orderStatuses
```
