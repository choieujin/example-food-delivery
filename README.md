
# 자동차 정비소 수리공간 배치
# Table of contents

- [자동차 정비소 수리공간 배치](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)

# 서비스 시나리오


기능적 요구사항
1. 주문을 접수한다.
2. 수리요청이 온다.
3. 수리공이 공간배치를 신청한다.
4. 공간을 배정해준다. (40퍼의 확률로 공간배치가 실패한다.)
5. 수리공에게 배정된 공간을 전달해준다. 
1. 수리주문건이 취소될 수 있다.
2. 수리주문건이 취소되면 공간배정이 취소된다
3. 고객이 주문상태를 중간중간 조회한다
4. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다

비기능적 요구사항
1. 트랜잭션
    1. 취소가 된 수리건은 공간배치가 취소되어야 한다. Sync 호출
1. 장애격리
    1. 공간배치 기능이 수행되지 않더라도 주문은 받을 수 있어야 한다.
1. 성능
    1. 고객이 수리상태를 Display 에서 확인할 수 있어야 한다  CQRS


# 분석/설계
## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/9D9vu5mgxNhZwInvqXjioqXtoTF3/mine/94f63ceddc78dbc8e3edd3675d480881/-MLNyEWMJzNsbEf6qodl


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/20468807/98316984-ddf71f80-201e-11eb-9d9a-4c48c8047574.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd receipt
mvn spring-boot:run
cd repair
mvn spring-boot:run
cd payment
mvn spring-boot:run
cd place
mvn spring-boot:run
cd display
mvn spring-boot:run

```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```
package automechanicsmall;
import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
@Entity
@Table(name="Place")
public class Place {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Integer placeNumber;
    private Long repairId;
    private String state;

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Integer getPlaceNumber() {
        return placeNumber;
    }

    public void setPlaceNumber(Integer placeNumber) {
        this.placeNumber = placeNumber;
    }
    public Long getRepairId() {
        return repairId;
    }

    public void setRepairId(Long repairId) {
        this.repairId = repairId;
    }

}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
public interface PlaceRepository extends PagingAndSortingRepository<Place, Long>{

    List<Place> findByRepairId(Long repairId);

}
```
- 적용 후 REST API 의 테스트
```
# 접수생성 & 수리요청 
curl -X POST  http://localhost:8088/receipts -d '{"vehiNo":"12가1234", "stat":"REQUESTREPAIR","receiptId":"1"}' -H 'Content-Type':'application/json'

# 공간 배정 요청
curl -X PATCH http://localhost:8088/repairs/1  -d '{"stat":"PLACEREQUEST"}' -H 'Content-Type':'application/json'

# 주문 상태 확인
curl -X GET http://localhost:8088/repairs/1

```


## 폴리글랏 퍼시스턴스

앱프런트 (app) 는 서비스 특성상 많은 사용자의 유입과 상품 정보의 다양한 콘텐츠를 저장해야 하는 특징으로 인해 RDB 보다는 Document DB / NoSQL 계열의 데이터베이스인 Mongo DB 를 사용하기로 하였다. 이를 위해 order 의 선언에는 @Entity 가 아닌 @Document 로 마킹되었으며, 별다른 작업없이 기존의 Entity Pattern 과 Repository Pattern 적용과 데이터베이스 제품의 설정 (application.yml) 만으로 MongoDB 에 부착시켰다

```
# application.yml

---
spring:
  profiles: nosql
  data:
    mongodb:
      uri: mongodb://${mongodb.url}:27017/place
...
---
spring:
  profiles: docker
...
  datasource:
    url: jdbc:mariadb://${datasource.url}:3306/place?useSSL=true
    username: ${datasource.user}
    password: ${datasource.password}
    driverClassName: org.mariadb.jdbc.Driver

```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(app)->결제(pay) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- Place서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
```
# (repair) PlaceService.java
@FeignClient(name="Place", url="${api.place.url}")
public interface PlaceService {

    @GetMapping("/places/cancel")
    public void placeCancel(@RequestParam("repairId") String repairId);

}
```

```
# Repair.java

   @PreUpdate
    public void onPreUpdate(){
        // 수리 취소
        if(this.getStat().equals("REPAIRECANCELLED")) {
            // place cancel 요청
            String repairId = this.id.toString();
            RepairApplication.applicationContext.getBean(automechanicsmall.external.PlaceService.class)
                    .placeCancel(repairId);
        }
    }

```

```
# PlaceController.java
  public void cancel(@RequestParam("repairId") String repairId) {
   List<Place> placeList = placeRepository.findByRepairId(Long.parseLong(repairId));
   Place place = placeList.get(0);
   place.setState("CANCELED");
   placeRepository.save(place);
  }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# Place 서비스를 잠시 내려놓음 (ctrl+c)

# 수리주문 취소 처리
curl -X PATCH http://localhost:8082/repairs/1  -d '{"vehiNo":"0000", "stat":"REPAIRECANCELLED"}' -H 'Content-Type':'application/json' # 실패
# {"timestamp":"2020-11-06T02:23:37.942+0000","status":500,"error":"Internal Server Error","message":"Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction","path":"/repairs/1"}

# Place 서비스 재기동

# 수리주문 취소 처리
curl -X PATCH http://localhost:8082/repairs/1  -d '{"vehiNo":"0000", "stat":"REPAIRECANCELLED"}' -H 'Content-Type':'application/json' # 성공
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

 
- 이를 위하여 공간배치 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
@Entity
@Table(name="Place")
public class Place {

 ...
    @PostPersist
    public void onPostPersist(){
        if(this.state.equals("COMPLETE")){
            AssignApproved assignApproved = new AssignApproved();
            assignApproved.setId(this.id);
            assignApproved.setPlaceNumber(this.placeNumber);
            assignApproved.setRepairId(this.repairId);
            BeanUtils.copyProperties(this, assignApproved);
            assignApproved.publishAfterCommit();
        }
        if(this.state.equals("FAILED")){
            AssignApproved assignApproved = new AssignApproved();
            assignApproved.setId(this.id);
            assignApproved.setPlaceNumber(this.placeNumber);
            assignApproved.setRepairId(this.repairId);
            BeanUtils.copyProperties(this, assignApproved);
            assignApproved.publishAfterCommit();
        }
    }

}
```
- 상점 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package fooddelivery;

...

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverAssignApproved_RequestRepair(@Payload AssignApproved assignApproved){

        if(assignApproved.isMe()){
            if(assignApproved.getRepairId()==null){
                System.out.println("ID is null.");
                return;
            }
            Optional<Repair> entity = repairRepository.findById(assignApproved.getRepairId());
            if(!entity.isPresent()) { // null
                System.out.println("There is no such Repair Log");
            } else{
                Repair repair = entity.get();
                if(assignApproved.getPlaceNumber().equals(0) ){ // place assign 실패
                    System.out.println("PLACE ASSIGNE 실패");
                    repair.setStat("ASSIGNEFAILED");
                } else{ // place assing
                    System.out.println("PLACE ASSIGNE 성공");
                    repair.setStat("PLACEASSIGNED");
                }
                repairRepository.save(repair);
            }
        }
    }

}

```
실제 구현을 하자면, 카톡 등으로 점주는 노티를 받고, 요리를 마친후, 주문 상태를 UI에 입력할테니, 우선 주문정보를 DB에 받아놓은 후, 이후 처리는 해당 Aggregate 내에서 하면 되겠다.:
  
```
  @Autowired 주문관리Repository 주문관리Repository;
  
  @StreamListener(KafkaProcessor.INPUT)
  public void whenever결제승인됨_주문정보받음(@Payload 결제승인됨 결제승인됨){

      if(결제승인됨.isMe()){
          카톡전송(" 주문이 왔어요! : " + 결제승인됨.toString(), 주문.getStoreId());

          주문관리 주문 = new 주문관리();
          주문.setId(결제승인됨.getOrderId());
          주문관리Repository.save(주문);
      }
  }

```

상점 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 상점시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# Place 를 잠시 내려놓음 (ctrl+c)

#주문 접수 
curl -X PATCH http://localhost:8088/repairs/1  -d '{"vehiNo":"0000", "stat":"PLACEREQUEST"}' -H 'Content-Type':'application/json' # 성공

#주문
curl -X GET http://localhost:8088/repairs     # 상태 안바뀜 확인

# Place 서비스 기동

#공간배정 상태 확인
curl -X GET http://localhost:8088/places     # 상태가 "PLACEASSIGNED"으로 확인
```


# 운영

## CI/CD 설정

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Istio Destination rule MSA 의 각 서비스들의 에러가 전파되는 것을 막기 위해서 특정 서비스에서 에러가 발생할 경우 해당 서비스로의 연결을 차단하도록 구성하였습니다.

Place 서비스에 부하가 차거나 에러가 발생할 경우 연결 차단

- Destination rule 를 설정: http connection pool 이 1개가 차면 연결을 끊는다. 5xx Error 가 5번 연속적으로 발생 시 해당 서비스로의 연결을 5분 동안 끊는다.
```
# ❯ kubectl -n car get destinationrule place-dr -o yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: place-dr
  namespace: car
spec:
  host: place
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      baseEjectionTime: 5m
      consecutive5xxErrors: 1
      interval: 5s
      maxEjectionPercent: 100

```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:

```
# destinationrule 적용 전
$ siege -c20 -t2S -v --content-type "application/json" 'http://gateway:8080/places/1 PATCH {"vehiNo":"111111111111","stat":"RECEIPTED","receiptId":"1"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 200     0.11 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.19 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.18 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.20 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.19 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.20 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.22 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.20 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.21 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.22 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.11 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.11 secs:     217 bytes ==> PATCH http://gateway:8080/places/1

Lifting the server siege...
Transactions:		         163 hits
Availability:		      100.00 %
Elapsed time:		        1.62 secs
Data transferred:	        0.03 MB
Response time:		        0.19 secs
Transaction rate:	      100.62 trans/sec
Throughput:		        0.02 MB/sec
Concurrency:		       18.98
Successful transactions:         163
Failed transactions:	           0
Longest transaction:	        0.31
Shortest transaction:	        0.08

# 모두 성공하였다.

# destinationrule 적용
❯ k -n car get destinationrule
NAME       HOST    AGE
place-dr   place   6s

$ siege -c20 -t2S -v --content-type "application/json" 'http://gateway:8080/places/1 PATCH {"vehiNo":"111111111111","stat":"RECEIPTED","receiptId":"1"}'

HTTP/1.1 200     0.14 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.11 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 503     0.06 secs:      81 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.18 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.17 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.19 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.08 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.11 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.11 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.09 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.11 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.09 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.09 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.09 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.09 secs:     217 bytes ==> PATCH http://gateway:8080/places/1
HTTP/1.1 200     0.12 secs:     217 bytes ==> PATCH http://gateway:8080/places/1

Lifting the server siege...
Transactions:		         137 hits
Availability:		       92.57 %
Elapsed time:		        1.11 secs
Data transferred:	        0.03 MB
Response time:		        0.15 secs
Transaction rate:	      123.42 trans/sec
Throughput:		        0.03 MB/sec
Concurrency:		       18.73
Successful transactions:         137
Failed transactions:	          11
Longest transaction:	        0.35
Shortest transaction:	        0.03

circuit breaker 가 동작하여 중간에 연결에 차단되 발생된 에러가 생겼다. 

```

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy place --min=1 --max=10 --cpu-percent=20 -n car

❯ k -n car get hpa
NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
place   Deployment/place   1%/20%    1         10        1          9h
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
# 부하가 발생하도록 place 서비스의 cpu request 값을 낮춰서 재생성한다. 
# 부하가 발생하도록 place 서비스의 circuit breaker 에서 connection pool 에 따른 제약조건을 없앤다.

siege -c250 -t50S -v --content-type "application/json" 'http://gateway:8080/places/1 PATCH {"vehiNo":"111111111111","stat":"RECEIPTED","receiptId":"1"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:

- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
❯ k -n car get pod
NAME                       READY   STATUS    RESTARTS   AGE
gateway-5f98595bc8-x6629   2/2     Running   0          3h37m
place-9d5578bbf-2cmr9      1/2     Running   0          68s
place-9d5578bbf-bqmsm      2/2     Running   0          68m
place-9d5578bbf-dsb2j      1/2     Running   0          68s
place-9d5578bbf-wfsqn      1/2     Running   0          68s
repair-694f75b799-vfzr6    2/2     Running   0          3h37m
siege-5c7c46b788-2x4q6     2/2     Running   0          98m
:
```


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함
```
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /places
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
```
- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
$ siege -c2 -t2S -v --content-type "application/json" 'http://gateway:8080/places'
...

Lifting the server siege...
Transactions:		          36 hits
Availability:		      100.00 %
Elapsed time:		        1.21 secs
Data transferred:	        0.03 MB
Response time:		        0.06 secs
Transaction rate:	       29.75 trans/sec
Throughput:		        0.03 MB/sec
Concurrency:		        1.93
Successful transactions:          36
Failed transactions:	           0
Longest transaction:	        0.30
Shortest transaction:	        0.01

```

- 새로 배포 시작
```
kubectl edit deploy place
```

```
$ siege -c2 -t2S -v --content-type "application/json" 'http://gateway:8080/places'

...
HTTP/1.1 200     0.03 secs:     950 bytes ==> GET  /places
HTTP/1.1 200     0.27 secs:     950 bytes ==> GET  /places

Lifting the server siege...
Transactions:		          29 hits
Availability:		      100.00 %
Elapsed time:		        1.70 secs
Data transferred:	        0.03 MB
Response time:		        0.10 secs
Transaction rate:	       17.06 trans/sec
Throughput:		        0.02 MB/sec
Concurrency:		        1.72
Successful transactions:          29
Failed transactions:	           0
Longest transaction:	        0.28
Shortest transaction:	        0.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


## Liveness
```
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - echo Hello Kubernetes! > /tmp/healthy && cat /tmp/healthy
          failureThreshold: 3
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
```
liveness 체크를 위해 변경한뒤 Pod 이 재시작 되는 것을 체크한다. 
```
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - cat /tmp/healthy
          failureThreshold: 3
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
```

```
place-59b8755b55-28wmr     1/2     Running   1          47s
```
