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
     resources:
            requests:
              cpu: 1000m
              memory: 256Mi
            limits:
              cpu: 2500m
              memory: 512Mi
```
### 점검 순서
```
1. HPA 생성 및 설정
	kubectl autoscale deployment order --cpu-percent=10 --min=1 --max=10
2. 모니터링 걸어놓고 확인
	kubectl get hpa order -w
	watch kubectl get deploy,po
3. Siege 실행
       siege -c100 -t60S -v 'http://order:8080/orders'
```

### 점검 결과

![image](https://user-images.githubusercontent.com/62786155/105106704-b8e05c00-5af9-11eb-95c2-cdeb314c4404.png)
![image](https://user-images.githubusercontent.com/62786155/105106715-bc73e300-5af9-11eb-9d16-88f486ecab7e.png)

## Readiness Probe 점검
### 설정 확인
```
   readinessProbe:
      	    httpGet:
              path: '/orders'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10

```
### 점검 순서
#### 1. Siege 실행
```
siege -c100 -t60S -v 'http://order:8080/orders'
```
#### 2. Order 이미지 교체

```
  kubectl set image deployment order order=final05crg.azurecr.io/order:latest
```

#### 3. Pod 모니터링
![image](https://user-images.githubusercontent.com/62786155/105107327-3fe20400-5afb-11eb-9b16-7fefba20c270.png)


#### 4. Siege 결과 Availability 확인(100%)

![image](https://user-images.githubusercontent.com/62786155/105107324-3e184080-5afb-11eb-935f-a35d85be1624.png)

