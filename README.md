# CNA Azure 1차수 (5조_든든한 아침식사)

# 서비스 시나리오

## 든든한 아침식사

기능적 요구사항

1. 고객이 회원가입을 하고 정보 수정 및 탈퇴를 한다
1. 매니져가 음식을 카탈로그에 등록하고 재고등 정보 수정 및 삭제를 한다.
1. 카탈로그에 등록된 음식정보는 별도의 카탈로그 조회 (프론트엔드) 에서 확인할수 있어야 한다. 
1. 고객이 음식을 주문하고, 주문을 생성할때는 고객정보와 음식 카탈로그 정보가 있어야 한다.
    1. Order -> Customer 동기호출
    1. Order -> FoodCatalog 동기호출
1. 고객이 결제를 한다.
1. 결제가 완료되면 주문 내역이 배송팀에게 전달된다.
1. 배송팀이 주문 확인 후 배송 처리한다.
1. 배송이 완료되면 상태가 업데이트 된다.
1. 고객이 주문을 취소할 수 있다.
1. 고객이 주문을 취소하면 배송팀 확인후 배송이 취소된다.
1. 배송이 취소되면 결제도 취소된다.
1. 고객이 주문하거나 취소를 하면 음식 Catalog 의 재고수량에 반영이 되어야 한다.

비기능적 요구사항

1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다  Sync 호출
    1. 배송이 완료된 주문은 취소할수 없다. Sync 호출
1. 장애격리
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객은 자신이 주문한 내용과 배달상태를 나의 주문정보(프론트엔드)에서 확인할 수 있어야 한다  CQRS


# Event Storming 모델
 ![image](https://user-images.githubusercontent.com/62786155/105103637-3f923a80-5af4-11eb-9107-52e919060740.PNG)

## 구현 점검

### 모든 서비스 정상 기동 
```
* Httpie Pod 접속
kubectl exec -it httpie -- bash

* API 
http http://gateway:8080/orders
http http://gateway:8080/deliveries
http http://gateway:8080/customers
http http://gateway:8080/myOrders
http http://gateway:8080/pays
http http://gateway:8080/foodCatalogs
http http://gateway:8080/foodCatalogViews
```

### Kafka 기동 및 모니터링 용 Consumer 연결
```
kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-console-consumer --bootstrap-server my-kafka:9092 --topic gmfd --from-beginning
```

### 고객 생성
```
http POST http://gateway:8080/customers name=lee phone=010 address=seoul age=29
http POST http://gateway:8080/customers name=kim phone=011 address=busan age=35
```

### 책 정보 생성
```
$ http POST http://gateway:8080/books bookName="alice in a wonderland" stock=100
HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 01:53:45 GMT
Location: http://bookinventory:8080/books/1
transfer-encoding: chunked

{
    "_links": {
        "book": {
            "href": "http://bookinventory:8080/books/1"
        }, 
        "self": {
            "href": "http://bookinventory:8080/books/1"
        }
    }, 
    "bookName": "alice in a wonderland", 
    "stock": 100
}

$ http POST http://gateway:8080/books bookName="quobadis?" stock=50
HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 01:54:38 GMT
Location: http://bookinventory:8080/books/2
transfer-encoding: chunked

{
    "_links": {
        "book": {
            "href": "http://bookinventory:8080/books/2"
        }, 
        "self": {
            "href": "http://bookinventory:8080/books/2"
        }
    }, 
    "bookName": "quobadis?", 
    "stock": 50
}

```

### 주문 생성
```
http POST http://gateway:8080/orders bookId=1 customerId=1 deliveryAddress="bundang gu" quantity=50
http POST http://gateway:8080/orders bookId=1 customerId=2 deliveryAddress="incheon si" quantity=100
```

##### Message 전송 확인 결과
```
{"eventType":"Ordered","timestamp":"20200909024119","orderId":4,"bookId":1,"customerId":2,"quantity":100,"deliveryAddress":"incheon si","orderStatus":"ORDERED","me":true}
```

##### Deliveriy 확인 결과
```
root@httpie:/# http http://gateway:8080/deliveraries
{
    "_links": {
        "delivery": {
            "href": "http://delivery:8080/deliveries/4"
        }, 
        "self": {
            "href": "http://delivery:8080/deliveries/4"
        }
    }, 
    "deliveryAddress": "incheon si", 
    "deliveryStatus": "CreateDelivery", 
    "orderId": 4
}

```

##### Deliverables 확인 결과
```
root@httpie:/# http http://gateway:8080/deliverables
           {
                "_links": {
                    "deliverable": {
                        "href": "http://bookinventory:8080/deliverables/5"
                    }, 
                    "self": {
                        "href": "http://bookinventory:8080/deliverables/5"
                    }
                }, 
                "orderId": 4, 
                "quantity": 100, 
                "status": "Stock_Lacked"
            }
```

### 주문 준비
```
root@httpie:/# http POST http://gateway:8080/stockInputs bookId=1 quantity=200
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:47:04 GMT
transfer-encoding: chunked

{
    "bookId": 1, 
    "id": 8, 
    "inCharger": null, 
    "quantity": 200
}
```
```
{"eventType":"DeliveryPrepared","timestamp":"20200909024704","id":null,"orderId":4,"status":"Delivery_Prepared","me":true}
{"eventType":"DeliveryStatusChanged","timestamp":"20200909024704","id":4,"orderId":4,"deliveryStatus":"Shipped","me":true}
```
##### 재고 수량 변경 확인 결과
```
root@httpie:/# http http://gateway:8080/books/1 
HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:53:26 GMT
transfer-encoding: chunked
{
    "_links": {
        "book": {
            "href": "http://bookinventory:8080/books/1"
        }, 
        "self": {
            "href": "http://bookinventory:8080/books/1"
        }
    }, 
    "bookName": "alice in a wonderland", 
    "stock": 150
}
```

##### 주문 상태 변경 확인 결과
```
root@httpie:/# http http://gateway:8080/orders/4
HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:55:30 GMT
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://order:8080/orders/4"
        }, 
        "self": {
            "href": "http://order:8080/orders/4"
        }
    }, 
    "bookId": 1, 
    "customerId": 2, 
    "deliveryAddress": "incheon si", 
    "orderStatus": "Shipped", 
    "quantity": 100
}

root@httpie:/# http http://gateway:8080/deliverables/5
HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:55:07 GMT
transfer-encoding: chunked

{
    "_links": {
        "deliverable": {
            "href": "http://bookinventory:8080/deliverables/5"
        }, 
        "self": {
            "href": "http://bookinventory:8080/deliverables/5"
        }
    }, 
    "orderId": 3, 
    "quantity": 100, 
    "status": "Delivery_Prepared"
}
```

### 배송 상태 변경
```
```

### 고객 Mypage 이력 확인
```
```

### 장애 격리
```
1. Customer 서비스 중지
	kubectl delete deploy customer
	
2. 주문 생성
	root@httpie:/# http POST http://gateway:8080/orders bookId=1 customerId=1 deliveryAddress="bundang gu" quantity=50

3. 주문 생성 결과 확인
HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:00:35 GMT
Location: http://order:8080/orders/1
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://order:8080/orders/1"
        }, 
        "self": {
            "href": "http://order:8080/orders/1"
        }
    }, 
    "bookId": 1, 
    "customerId": 1, 
    "deliveryAddress": "bundang gu", 
    "orderStatus": "Customer_Not_Verified", 
    "quantity": 50
}
```

## CI/CD 점검

## Circuit Breaker 점검

```
Hystrix Command
	5000ms 이상 Timeout 발생 시 CircuitBearker 발동

CircuitBeaker 발생
	http http://delivery:8080/selectDeliveryInfo?deliveryId=1
		- 잘못된 쿼리 수행 시 CircuitBeaker
		- 10000ms(10sec) Sleep 수행
		- 5000ms Timeout으로 CircuitBeaker 발동
		- 10000ms(10sec) 
```

```
실행 결과

root@httpie:/# http http://delivery:8080/selectDeliveryInfo?deliveryId=1
HTTP/1.1 200 
Content-Length: 7
Content-Type: text/plain;charset=UTF-8
Date: Wed, 09 Sep 2020 04:27:53 GMT

Shipped

root@httpie:/# http http://delivery:8080/selectDeliveryInfo?deliveryId=0
HTTP/1.1 200 
Content-Length: 17
Content-Type: text/plain;charset=UTF-8
Date: Wed, 09 Sep 2020 04:28:03 GMT

CircuitBreaker!!!

root@httpie:/# http http://delivery:8080/selectDeliveryInfo?deliveryId=1
HTTP/1.1 200 
Content-Length: 17
Content-Type: text/plain;charset=UTF-8
Date: Wed, 09 Sep 2020 04:28:06 GMT

CircuitBreaker!!!

```

```
소스 코드

@GetMapping("/selectDeliveryInfo")
  @HystrixCommand(fallbackMethod = "fallbackDelivery", commandProperties = {
          @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000"),
          @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000")
  })
  public String selectDeliveryInfo(@RequestParam long deliveryId) throws InterruptedException {

   if (deliveryId <= 0) {
    System.out.println("@@@ CircuitBreaker!!!");
    Thread.sleep(10000);
    //throw new RuntimeException("CircuitBreaker!!!");
   } else {
    Optional<Delivery> delivery = deliveryRepository.findById(deliveryId);
    return delivery.get().getDeliveryStatus();
   }

   System.out.println("$$$ SUCCESS!!!");
   return " SUCCESS!!!";
  }

 private String fallbackDelivery(long deliveryId) {
  System.out.println("### fallback!!!");
  return "CircuitBreaker!!!";
 }
```

## Autoscale 점검
### 설정 확인
```
application.yaml 파일 설정 변경
(https://k8s.io/examples/application/php-apache.yaml 파일 참고)
 resources:
  limits:
    cpu: 500m
  requests:
    cpu: 200m
```
### 점검 순서
```
1. HPA 생성 및 설정
	kubectl autoscale deploy bookinventory --min=1 --max=10 --cpu-percent=30
	kubectl get hpa bookinventory -o yaml
2. 모니터링 걸어놓고 확인
	kubectl get hpa bookinventory -w
	watch kubectl get deploy,po
3. Siege 실행
  siege -c10 -t60S -v http://gateway:8080/books/
```
### 점검 결과
![Alt text](images/HPA_test.PNG?raw=true "Optional Title")

## Readiness Probe 점검
### 설정 확인
```
readinessProbe:
  httpGet:
    path: '/orders'
    port: 8080
  initialDelaySeconds: 12
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
```
### 점검 순서
#### 1. Siege 실행
```
siege -c2 -t120S  -v 'http://gateway:8080/orders
```
#### 2. Readiness 설정 제거 후 배포
#### 3. Siege 결과 Availability 확인(100% 미만)
```
Lifting the server siege...      done.

Transactions:                    330 hits
Availability:                  70.82 %
Elapsed time:                 119.92 secs
Data transferred:               0.13 MB
Response time:                  0.02 secs
Transaction rate:               2.75 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    0.07
Successful transactions:         330
Failed transactions:             136
Longest transaction:            1.75
Shortest transaction:           0.00
```
#### 4. Readiness 설정 추가 후 배포

#### 6. Siege 결과 Availability 확인(100%)
```
Lifting the server siege...      done.

Transactions:                    443 hits
Availability:                 100.00 %
Elapsed time:                 119.60 secs
Data transferred:               0.51 MB
Response time:                  0.01 secs
Transaction rate:               3.70 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    0.04
Successful transactions:         443
Failed transactions:               0
Longest transaction:            0.18
Shortest transaction:           0.00
 
FILE: /var/log/siege.log
```

## Liveness Probe 점검
### 설정 확인
```
livenessProbe:
  httpGet:
    path: '/isHealthy'
    port: 8080
  initialDelaySeconds: 120
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 5
```
### 점검 순서 및 결과
#### 1. 기동 확인
```
http http://gateway:8080/orders
```
#### 2. 상태 확인
```
oot@httpie:/# http http://order:8080/isHealthy
HTTP/1.1 200 
Content-Length: 0
Date: Wed, 09 Sep 2020 02:14:22 GMT
```

#### 3. 상태 변경
```
root@httpie:/# http http://order:8080/makeZombie
HTTP/1.1 200 
Content-Length: 0
Date: Wed, 09 Sep 2020 02:14:24 GMT
```
#### 4. 상태 확인
```
root@httpie:/# http http://order:8080/isHealthy
HTTP/1.1 500 
Connection: close
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:14:28 GMT
Transfer-Encoding: chunked

{
    "error": "Internal Server Error", 
    "message": "zombie.....", 
    "path": "/isHealthy", 
    "status": 500, 
    "timestamp": "2020-09-09T02:14:28.338+0000"
}
```
#### 5. Pod 재기동 확인
```
root@httpie:/# http http://order:8080/isHealthy
http: error: ConnectionError: HTTPConnectionPool(host='order', port=8080): Max retries exceeded with url: /makeZombie (Caused by NewConnectionError('<requests.packages.urllib3.connection.HTTPConnection object at 0x7f5196111c50>: Failed to establish a new connection: [Errno 111] Connection refused',))

root@httpie:/# http http://order:8080/isHealthy
HTTP/1.1 200 
Content-Length: 0
Date: Wed, 09 Sep 2020 02:36:00 GMT
```
