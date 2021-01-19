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
kubectl exec -it siege -- /bin/bash

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

### FoodCatalog 정보 생성
```
# http POST http://gateway:8080/foodCatalogs name=pizza stock=100 price=1000             

HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Tue, 19 Jan 2021 10:51:18 GMT
Location: http://foodCatalog:8080/foodCatalogs/1
transfer-encoding: chunked

{
    "_links": {
        "foodCatalog": {
            "href": "http://foodCatalog:8080/foodCatalogs/1"
        },
        "self": {
            "href": "http://foodCatalog:8080/foodCatalogs/1"
        }
    },
    "name": "pizza",
    "price": 1000.0,
    "stock": 100
} 


$ http POST http://gateway:8080/foodCatalogs name=meat stock=100 price=2000

HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Tue, 19 Jan 2021 10:53:52 GMT
Location: http://foodCatalog:8080/foodCatalogs/2
transfer-encoding: chunked

{
    "_links": {
        "foodCatalog": {
            "href": "http://foodCatalog:8080/foodCatalogs/2"
        },
        "self": {
            "href": "http://foodCatalog:8080/foodCatalogs/2"
        }
    },
    "name": "meat",
    "price": 2000.0,
    "stock": 100
}      

```

### 주문 생성
```
http POST http://gateway:8080/orders qty=20 foodcaltalogid=1 customerid=1
```

##### Message 전송 확인 결과
```
{"eventType":"Ordered","timestamp":"20210119130159","id":3,"qty":20,"status":null,"foodcaltalogid":1,"customerid":1,"me":true}
```

##### Delivery 확인 결과
```
# http http://gateway:8080/deliveries

HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 19 Jan 2021 13:04:21 GMT
transfer-encoding: chunked

{
    "_embedded": {
        "deliveries": [
         {
                "_links": {
                    "delivery": {
                        "href": "http://delivery:8080/deliveries/4"
                    },
                    "self": {
                        "href": "http://delivery:8080/deliveries/4"
                    }
                },
                "orderId": 3,
                "status": "Ordered"
            }
        ]
    } 
}
```

##### MyOrder 조회 (CQRS)
```
# http http://gateway:8080/myOrders

Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 19 Jan 2021 12:48:28 GMT
transfer-encoding: chunked

{
    "_embedded": {
        "myOrders": [
 {
                "_links": {
                    "myOrder": {
                        "href": "http://customer:8080/myOrders/5"
                    },
                    "self": {
                        "href": "http://customer:8080/myOrders/5"
                    }
                },
                "customerid": 1,
                "deliveryid": 3,
                "foodcatlogid": 1,
                "orderid": 3,
                "qty": 20,
                "status": “Delivery Start”
            }
        ]
    }
}
```

### 주문 취소
```
# http DELETE http://gateway:8080/orders/46
```

##### Delivery 취소, 주문취소, Payment 취소 메시지 전송
```
{"eventType":"DeliveryCanceled","timestamp":"20210119203304","id":1171,"status":"Cancelled-delivery","orderId":46,"me":true}
{"eventType":"OrderCancelled","timestamp":"20210119203304","id":46,"qty":50,"status":＂Order Canceled","foodcaltalogid":1,"customerid":2,"me":true}
{"eventType":"Cancelled","timestamp":"20210119203304","id":46,"amout":50,"status":"Order Canceled-payment","orderid":75,"me":true}
```


### 장애 격리
```
1. Delivery 서비스 중지
	kubectl delete deploy delivery
	
2. 주문 생성
	# http POST http://gateway:8080/orders qty=35 foodcaltalogid=2 customerid=2

3. 주문 생성 결과 확인

HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Tue, 19 Jan 2021 20:53:43 GMT
Location: http://order:8080/orders/1375
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://order:8080/orders/1375"
        },
        "self": {
            "href": "http://order:8080/orders/1375"
        }
    },
    "customerid": 2,
    "foodcaltalogid": 2,
    "qty": 35,
    "status": null
}

4. Delivery 서비스 재성후 Shipped 메시지 정상 전송확인
	
	/gmfd/delivery/kubernetes$ kubectl apply -f deployment.yml

{"eventType":"Shipped","timestamp":"20210119205555","id":1,"status":"Delivery Start","orderId":1375,"me":true}
```

## CI/CD 점검

## Circuit Breaker 점검

### Readiness 와 Liveness 제외후 Order 서비스 재생성

```
kubectl delete deploy order
kubectl apply -f deployment2.yml 
```
### 부하발생

```
Kubectl exec –it siege -- /bin/bash
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://gateway:8080/orders POST {"qty": 50, "foodcaltalogid":1 , "customerid":2 }'

```


### 실행결과

![image](https://user-images.githubusercontent.com/62786155/105106062-b29db000-5af8-11eb-80ae-90583bd49e08.png)



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
