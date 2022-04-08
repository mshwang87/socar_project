![image](https://user-images.githubusercontent.com/12591322/162278748-e2e84bb4-b2e0-4e48-abc2-5225bbbd61e7.png)

# 쏘카 커버하기 

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# 평가항목
  * 분석/설계(이벤트스토밍)
  * SAGA
  * CQRS
  * Correlation / Compensation
  * Req / Resp
  * Gateway
  * Deploy / Pipeline
  * Circuit Breaker
  * Autoscale(HPA)
  * Self-healing(Liveness Probe)
  * Zero-downtime deploy(Readiness Probe)
  * Config Map / Persustemce Volume
  * Polyglot
   
----

# 분석/설계(이벤트스토밍)

## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/12591322/162216120-ac106969-f41a-4b74-9c7e-c07a229476cc.png)

## TO-BE 조직 (Vertically-Aligned)  
  ![image](https://user-images.githubusercontent.com/12591322/162217480-50928b2c-1bcb-44c3-9f4f-b7c02ed4f143.png)


# 서비스 시나리오

기능적 요구사항

1. Host는 고객에게 제공할 차량을 등록/수정/삭제 한다.
2. Customer는 차량을 선택하여 예약한다.
3. 예약과 동시에 결제가 진행된다.
4. 예약이 되면 예약 내역(Message)이 전달된다.
5. 고객이 예약을 취소할 수 있다.
6. 예약 사항이 취소될 경우 취소 내역(Message)이 전달된다.
7. 전체적인 차량 예약에 대한 정보 및 상태 등을 한 화면에서 확인 할 수 있다.(viewpage)

비기능적 요구사항

1. 트랜잭션
    1. 결제가 되지 않은 예약 건은 성립되지 않아야 한다.  (Sync 호출)
2. 장애격리
    1. 차량 신규 등록 및 메시지 전송 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다.  Async (event-driven), Eventual Consistency
    2. 예약 시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시 후에 하도록 유도한다  Circuit breaker, fallback
3. 성능
    1. 모든 차량에 대한 정보 및 예약 상태 등을 한번에 확인할 수 있어야 한다  (CQRS)
    2. 예약의 상태가 바뀔 때마다 메시지로 알림을 줄 수 있어야 한다  (Event driven)



## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  https://labs.msaez.io/#/storming/wUpqnKyxMya9cRHPexnY8ITeWDs1/2a05aba524e2b1919fb1083aa310c989

* 쏘카 서비스의 전체적인 구조 및 흐름을 파악하였으며, 각 Bounded Context 간의 pub/sub, req/res 관계를 확인하여 연결하였습니다.
![1](https://user-images.githubusercontent.com/12591322/162213249-6a5cd9da-fef3-4c27-b5db-7934ff3fbada.png)



# SAGA

분석/설계 단계에서 도출된 결과에 따라 마이크로 서비스들을 스프링부트로 구현하였습니다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같습니다

```
    cd car
    mvn spring-boot:run
```
```
    cd message
    mvn spring-boot:run 
```
```
    cd payment
    mvn spring-boot:run  
```
```
    cd reservation
    mvn spring-boot:run  
```
```
    cd viewpage
    mvn spring-boot:run  
```


+ DDD적용<p>
    5개의 도메인으로 관리되고 있으며 `차량관리(Car)`, `메시지관리(Message)`, `결제(Payment)`, `예약(Reservation)`, `뷰페이지(CQRS)(viewpage)`으로 구성됩니다.


```diff
@Entity
@Table(name="Car_table")
public class Car  {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long carId;
    private String status;
    private String carName;
    private Long amount;
    private String carType;

    @PostPersist
    public void onPostPersist(){
        // 차량 등록 
        // 초기값 세팅 
        status = "available";       // 최초 등록시 항상 이용가능

        CarRegistered carRegistered = new CarRegistered();
        BeanUtils.copyProperties(this, carRegistered);
        carRegistered.publishAfterCommit();

    }

    @PostUpdate
    public void onPostUpdate(){
        // 차량 정보 수정 
        CarModified carModified = new CarModified();
        BeanUtils.copyProperties(this, carModified);
        carModified.publishAfterCommit();

        CarReserved carReserved = new CarReserved();
        BeanUtils.copyProperties(this, carReserved);
        carReserved.publishAfterCommit();

        CarCancelled carCancelled = new CarCancelled();
        BeanUtils.copyProperties(this, carCancelled);
        carCancelled.publishAfterCommit();

    }
```


+ 서비스 호출 흐름(Sync)<p>
* `예약(Reservation)` -> `결제(Payment)`간 호출은 동기식으로 일관성을 유지하는 트랜젝션으로 처리
* Customer는 차량을 확인하고 예약 및 결제 수행
* 결제서비스를 호출하기위해 FeinClient를 이용하여 인터페이스(Proxy)를 구현 
* 예약이 가능하면(`@PostPersist`) 결제를 요청하도록 처리한다.
	
	
	
```
// Reservation/src/main/java/socar/external/PaymentService.java

@FeignClient(name="payment", url="http://user06-payment:8080")
public interface PaymentService {
    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void approvePayment(@RequestBody Payment payment);

}


// Reservation/src/main/java/socar/external/CarService.java
@FeignClient(name="car", url="http://user06-car:8080")
public interface CarService {
    @RequestMapping(method= RequestMethod.GET, path="/cars")
    public void chkAndReqReserve(@RequestBody Car car);

}
```	
	
	
	
	
	
	
## CQRS

차량의 사용가능 여부 확인, 차량 예약 및 결제 등 각각의 Status 에 대하여 고객(Customer)이 조회 할 수 있도록 CQRS 로 구현하였습니다.
- car, reservation, payment 개별 Aggregate Status 를 통합 조회하여 성능 Issue 를 사전에 예방할 수 있습니다.
- 비동기식으로 처리되어 발행된 이벤트 기반 Kafka 를 통해 수신/처리 되어 별도 Table 에 관리합니다
- Table 모델링 (carView)
![image](https://user-images.githubusercontent.com/12591322/162228220-c7ed2828-5476-4f6f-aeab-316491c5d048.png)


- viewpage MSA ViewHandler 를 통해 구현 ("CarRegistered" 이벤트 발생 시, Pub/Sub 기반으로 별도 Carview 테이블에 저장)
- 실제로 view 페이지를 조회해 보면 차량정보, 예약 및 결제 등을 확인 할 수 있습니다. 

	
```	
	@Service
	public class CarviewViewHandler {


	    @Autowired
	    private CarviewRepository carviewRepository;

+   	    // 차량이 등록되었을 때 insert -> viewpage table 
	    @StreamListener(KafkaProcessor.INPUT)
+	    public void whenCarRegistered_then_CREATE_1 (@Payload CarRegistered carRegistered) {
		try {

		    if (!carRegistered.validate()) return;

		    // view 객체 생성
		    Carview carview = new Carview();
		    // view 객체에 이벤트의 Value 를 set 함
		    carview.setCarId(carRegistered.getcarId());
		    carview.setCarStatus(carRegistered.getstatus());
		    carview.setCarName(carRegistered.getcarName());
		    carview.setCarType(carRegistered.getcarType());
		    // view 레파지 토리에 save
		    carviewRepository.save(carview);

		}catch (Exception e){
		    e.printStackTrace();
		}
	    }


+	    // 차량이 수정되었을 때 update -> viewpage table 
	    @StreamListener(KafkaProcessor.INPUT)
+	    public void whenCarModified_then_UPDATE_1(@Payload CarModified carModified) {
		try {
		    if (!carModified.validate()) return;
			// view 객체 조회
		    Optional<Carview> carviewOptional = carviewRepository.findById(carModified.getcarId());

		    if( carviewOptional.isPresent()) {
			 Carview carview = carviewOptional.get();
		    // view 객체에 이벤트의 eventDirectValue 를 set 함
			 carview.setCarStatus(carModified.getstatus());
			 carview.setCarName(carModified.getcarName());
			 carview.setCarType(carModified.getcarType());
			// view 레파지 토리에 save
			 carviewRepository.save(carview);
			}


		}catch (Exception e){
		    e.printStackTrace();
		}
	    }

+	    // 예약이 확정되었을 때 update -> viewpage table 
	    @StreamListener(KafkaProcessor.INPUT)
+	    public void whenReservationConfirmed_then_UPDATE_2(@Payload ReservationConfirmed reservationConfirmed) {
		try {
		    if (!reservationConfirmed.validate()) return;
			// view 객체 조회
		    Optional<Carview> carviewOptional = carviewRepository.findById(reservationConfirmed.getcarId());

		    if( carviewOptional.isPresent()) {
			 Carview carview = carviewOptional.get();
		    // view 객체에 이벤트의 eventDirectValue 를 set 함
			 carview.setRsvId(reservationConfirmed.getrsvId());
			 carview.setRsvStatus(reservationConfirmed.getstatus());
			// view 레파지 토리에 save
			 carviewRepository.save(carview);
			}


		}catch (Exception e){
		    e.printStackTrace();
		}
	    }

+	    // 결제가 완료 되었을 때 update -> viewpage table 
	    @StreamListener(KafkaProcessor.INPUT)
+	    public void whenPaymentApproved_then_UPDATE_3(@Payload PaymentApproved paymentApproved) {
		try {
		    if (!paymentApproved.validate()) return;
			// view 객체 조회

			    List<Carview> carviewList = carviewRepository.findByRsvId(paymentApproved.getrsvId());
			    for(Carview carview : carviewList){
			    // view 객체에 이벤트의 eventDirectValue 를 set 함
			    carview.setPayId(paymentApproved.getpayId());
			    carview.setPayStatus(paymentApproved.getstatus());
			// view 레파지 토리에 save
			carviewRepository.save(carview);
			}

		}catch (Exception e){
		    e.printStackTrace();
		}
	    }

+	    // 예약이 취소 되었을 때 update -> viewpage table
	    @StreamListener(KafkaProcessor.INPUT)
+	    public void whenReservationCancelled_then_UPDATE_4(@Payload ReservationCancelled reservationCancelled) {
		try {
		    if (!reservationCancelled.validate()) return;
			// view 객체 조회

			    List<Carview> carviewList = carviewRepository.findByRsvId(reservationCancelled.getrsvId());
			    for(Carview carview : carviewList){
			    // view 객체에 이벤트의 eventDirectValue 를 set 함
			    carview.setRsvStatus(reservationCancelled.getstatus());
			// view 레파지 토리에 save
			carviewRepository.save(carview);
			}

		}catch (Exception e){
		    e.printStackTrace();
		}
	    }

+	    // 결제가 취소 되었을 때 update -> viewpage table
	    @StreamListener(KafkaProcessor.INPUT)
+	    public void whenPaymentCancelled_then_UPDATE_5(@Payload PaymentCancelled paymentCancelled) {
		try {
		    if (!paymentCancelled.validate()) return;
			// view 객체 조회

			    List<Carview> carviewList = carviewRepository.findByPayId(paymentCancelled.getpayId());
			    for(Carview carview : carviewList){
			    // view 객체에 이벤트의 eventDirectValue 를 set 함
			    carview.setPayStatus(paymentCancelled.getstatus());
			// view 레파지 토리에 save
			carviewRepository.save(carview);
			}

		}catch (Exception e){
		    e.printStackTrace();
		}
	    }

+	    // 차량정보를 삭제 하였을 때  delete -> viewpage table
	    @StreamListener(KafkaProcessor.INPUT)
+	    public void whenCarDeleted_then_DELETE_1(@Payload CarDeleted carDeleted) {
		try {
		    if (!carDeleted.validate()) return;
		    // view 레파지 토리에 삭제 쿼리
		    carviewRepository.deleteById(carDeleted.getcarId());
		}catch (Exception e){
		    e.printStackTrace();
		}
	    }
	}

```	

## API 게이트웨이
      1. gateway 스프링부트 App을 추가 후 application.yaml내에 각 마이크로 서비스의 routes 를 추가하고 gateway 서버의 포트를 8080 으로 설
       
          - application.yaml 예시
            ```
            spring:
		  profiles: docker
		  sleuth:
		    propagation-keys: x-request-id,x-ot-span-context
		  zipkin:
		    base-url: http://jaeger-collector.default.svc.cluster.local:9411
		  cloud:
		    gateway:
		      routes:
			- id: payment
			  uri: http://user06-payment:8080
			  predicates:
			    - Path=/payments/** 
			- id: car
			  uri: http://user06-car:8080
			  predicates:
			    - Path=/cars/** 
			- id: reservation
			  uri: http://user06-reservation:8080
			  predicates:
			    - Path=/reservations/** 
			- id: message
			  uri: http://user06-message:8080
			  predicates:
			    - Path=/messages/** 
			- id: viewpage
			  uri: http://user06-viewpage:8080
			  predicates:
			    - Path= /roomviews/**
			- id: frontend
			  uri: http://user06-frontend:8080
			  predicates:
			    - Path=/**
		      globalcors:
			corsConfigurations:
			  '[/**]':
			    allowedOrigins:
			      - "*"
			    allowedMethods:
			      - "*"
			    allowedHeaders:
			      - "*"
			    allowCredentials: true

		server:
		  port: 8080      
            ```

         
      2. Kubernetes용 Deployment.yaml 을 작성하고 Kubernetes에 Deploy를 생성함
          - Deployment.yaml 예시
          

            ```
		apiVersion: apps/v1
		kind: Deployment
		metadata:
		  name: gateway
		  labels:
		    app: gateway
		spec:
		  replicas: 1
		  selector:
		    matchLabels:
		      app: gateway
		  template:
		    metadata:
		      labels:
			app: gateway
		    spec:
		      containers:
			- name: gateway
			  #image: 979050235289.dkr.ecr.ap-southeast-2.amazonaws.com/user06-gateway:1.0
          		  image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
          
			  ports:
			    - containerPort: 8080
            ```               
            

            ```
            Deploy 생성
            kubectl apply -f deployment.yaml
            ```     
          - Kubernetes에 생성된 Deploy. 확인
            
![image](https://user-images.githubusercontent.com/12591322/162340420-02685eee-e1ac-47e8-84d2-d273532c31b3.png)

	    
            
      3. Kubernetes용 Service.yaml을 작성하고 Kubernetes에 Service/LoadBalancer을 생성하여 Gateway 엔드포인트를 확인함. 
          - Service.yaml 예시
          
            ```
            apiVersion: v1
              kind: Service
              metadata:
                name: gateway
                namespace: airbnb
                labels:
                  app: gateway
              spec:
                ports:
                  - port: 8080
                    targetPort: 8080
                selector:
                  app: gateway
                type:
                  LoadBalancer           
            ```             

           
            ```
            Service 생성
            kubectl apply -f service.yaml            
            ```             
            
            
          - API Gateay 엔드포인트 확인
           
            ```
            Service  및 엔드포인트 확인 
            kubectl get svc -n airbnb           
            ```                 
![image](https://user-images.githubusercontent.com/12591322/162277731-27dbe1a3-7202-4340-be94-bcaa75551aeb.png)

	
# Correlation

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였습니다. (예시는 Car 마이크로 서비스). 
  가급적(유비쿼터스 랭귀지)를 그대로 사용하려고 노력했습니다. 

```
package socar;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;


@Entity
@Table(name="Car_table")
public class Car  {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long carId;
    private String status;
    private String carName;
    private Long amount;
    private String carType;

    public Long getCarId() {
        return carId;
    }

    public void setCarId(Long carId) {
        this.carId = carId;
    }
    
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    
    public String getCarName() {
        return carName;
    }

    public void setCarName(String carName) {
        this.carName = carName;
    }
    
    public Long getAmount() {
        return amount;
    }

    public void setAmount(Long amount) {
        this.amount = amount;
    }
    
    public String getCarType() {
        return carType;
    }

    public void setCarType(String carType) {
        this.carType = carType;
    }
}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 다양한 데이터소스에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였습니다
```
package socar;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="cars", path="cars")
public interface CarRepository extends PagingAndSortingRepository<Car, Long>{
}

```
- 적용 후 REST API 의 테스트
```
# car 서비스의 car 등록
http POST http://localhost:8088/cars carName="Mercedes-Benz"  

# reservation 서비스의 예약 요청
http POST http://localhost:8088/reservations carId=1 status=reqReserve

# reservation 서비스의 예약 상태 확인
http GET http://localhost:8088/reservations

```

## 동기식 호출(Sync) 과 Fallback 처리

분석 단계에서의 조건 중 하나로 예약 시 차량(Car) 간의 예약 가능 상태 확인 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하도록 했습니다.
그리고, 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 하였습니다.
또한 예약(reservation) -> 결제(payment) 서비스도 동기식으로 처리하기로 하였습니다 

- 차량, 결제 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# CarService.java

package socar.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="car", url="http://user06-car:8080")
public interface CarService {
    @RequestMapping(method= RequestMethod.GET, path="/cars")
    public void chkAndReqReserve(@RequestBody Car car);

}



# PaymentService.java

package socar.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="payment", url="http://user06-payment:8080")
public interface PaymentService {
    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void approvePayment(@RequestBody Payment payment);

}

```

- 예약 요청을 받은 직후(@PostPersist) 가능상태 확인 및 결제를 동기(Sync)로 요청하도록 처리
```
# Reservation.java (Entity)

    @PostPersist
    public void onPostPersist(){
        // 예약 요청이 들어왔을 경우 사용가능한지 확인
        socar.external.Car car = new socar.external.Car();
        // mappings goes here
        boolean result = ReservationApplication.applicationContext.getBean(socar.external.CarService.class)
            .chkAndReqReserve(car);
        System.out.println("사용가능 여부 : " + result);

        if(result) { 

            // 예약 가능한 상태인 경우(Available)
            // PAYMENT 결제모듈 호출 (POST방식) - SYNC 호출
            socar.external.Payment payment = new socar.external.Payment();
            payment.setRsvId(this.getRsvId());
            payment.setCarId(this.getCarId());
            payment.setStatus("paid");
            ReservationApplication.applicationContext.getBean(socar.external.PaymentService.class)
                .approvePayment(payment);

            // 이벤트시작 --> ReservationCreated
            ReservationCreated reservationCreated = new ReservationCreated();
            BeanUtils.copyProperties(this, reservationCreated);
            reservationCreated.publishAfterCommit();
        }
    }
```


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


결제가 이루어진 후에 차량의 상태가 업데이트 되고, 예약 시스템의 상태가 업데이트 되며, 예약 및 취소 메시지가 전송되는 시스템과의 통신 행위는 비동기식으로 처리합니다
 
- 이를 위하여 결제가 승인되면 결제가 승인 되었다는 이벤트를 카프카로 송출합니다. (Publish)
 
```
# Payment/src/.../Payment.java

package socar;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Payment_table")
public class Payment  {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long payId;
    private Long rsvId;
    private Long carId;
    private String status;


    @PostPersist
    public void onPostPersist(){
        // 결재 승인나면 paymentApproved 시작 
        PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();

    }
    ....
}
```

- 예약 시스템에서는 결제 승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현합니다

```
# Reservation.java

package socar;

    @PostUpdate
    public void onPostUpdate(){

        // 예약 취소 요청일 경우 
        if(this.getStatus().equals("reservationCancelRequested")) {
            ReservationCancelRequested reservationCancelRequested = new ReservationCancelRequested();
            BeanUtils.copyProperties(this, reservationCancelRequested);
            reservationCancelRequested.publishAfterCommit();
        }

        // 예약 확정일 경우 
        if(this.getStatus().equals("reservationConfirmed")) {
            ReservationConfirmed reservationConfirmed = new ReservationConfirmed();
            BeanUtils.copyProperties(this, reservationConfirmed);
            reservationConfirmed.publishAfterCommit();
        }

        // 예약 취소일 경우 
        if(this.getStatus().equals("reservationCancelled")) {
            ReservationCancelled reservationCancelled = new ReservationCancelled();
            BeanUtils.copyProperties(this, reservationCancelled);
            reservationCancelled.publishAfterCommit();
        }

    }

```

그 외 메시지 서비스는 예약/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 메시지 서비스가 유지보수로 인해 잠시 내려간 상태 라도 예약을 받는데 문제가 없도록 개발하였습니다

	

# 운영


## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD는 buildspec.yml을 이용한 AWS codebuild를 사용하였습니다.

- CodeBuild 프로젝트를 생성하고 AWS_ACCOUNT_ID, KUBE_URL, KUBE_TOKEN 환경 변수 세팅을 합니다
	
+ Service Account 생성
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
EOF
```
+ ClusterRoleBinding 생성
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
EOF
```
+ EKS 접속토큰 가져오기
```
kubectl -n kube-system describe secret eks-admin
```
![image](https://user-images.githubusercontent.com/12591322/162277440-bc89bb64-c748-455e-b7bf-42169bf202fd.png)

	
```
buildspec-kubectl.yml 파일 
마이크로 서비스 car의 yml 파일 이용하도록 세팅
```

```
version: 0.2

env:
  variables:
    _PROJECT_NAME: "user06-car"

phases:
  install:
    runtime-versions:
      java: corretto8
      docker: 18
    commands:
      - echo install kubectl
      - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin/kubectl
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - echo $_PROJECT_NAME
      - echo $AWS_ACCOUNT_ID
      - echo $AWS_DEFAULT_REGION
      - echo $CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo start command
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION  .
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo connect kubectl
      - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
      - kubectl config set-credentials admin --token="$KUBE_TOKEN"
      - kubectl config set-context default --cluster=k8s --user=admin
      - kubectl config use-context default
      - |
          cat <<EOF | kubectl apply -f -
          apiVersion: v1
          kind: Service
          metadata:
            name: $_PROJECT_NAME
            labels:
              app: $_PROJECT_NAME
          spec:
            ports:
              - port: 8080
                targetPort: 8080
            selector:
              app: $_PROJECT_NAME
          EOF
      - |
          cat  <<EOF | kubectl apply -f -
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: $_PROJECT_NAME
            labels:
              app: $_PROJECT_NAME
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: $_PROJECT_NAME
            template:
              metadata:
                labels:
                  app: $_PROJECT_NAME
              spec:
                containers:
                  - name: $_PROJECT_NAME
                    image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                    ports:
                      - containerPort: 8080
                    readinessProbe:
                      httpGet:
                        path: /actuator/health
                        port: 8080
                      initialDelaySeconds: 10
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 10
                    livenessProbe:
                      httpGet:
                        path: /actuator/health
                        port: 8080
                      initialDelaySeconds: 120
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 5
          EOF

cache:
  paths:
    - '/root/.m2/**/*'

```

- codebuild 실행
```
codebuild 프로젝트 및 빌드 이력
```
![codebuild(프로젝트)](https://user-images.githubusercontent.com/38099203/119283851-315a5380-bc79-11eb-9b2a-b4522d22d009.PNG)
![codebuild(로그)](https://user-images.githubusercontent.com/38099203/119283850-30c1bd00-bc79-11eb-9547-1ff1f62e48a4.PNG)

- codebuild 빌드 내역 (Car 서비스 세부)

![image](https://user-images.githubusercontent.com/12591322/162275263-5db2d141-6098-4dd6-ae4a-6280b9fbd830.png)
	
- codebuild 빌드 내역 (전체 이력 조회)

![image](https://user-images.githubusercontent.com/12591322/162275202-c377b96f-170a-43ca-a7d2-e6d2c2778712.png)



## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: istio 사용하여 구현함

시나리오는 예약(reservation)--> 룸(room) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 예약 요청이 과도할 경우 CB 를 통하여 장애격리.

- DestinationRule 를 생성하여 circuit break 가 발생할 수 있도록 설정
최소 connection pool 설정
```
# destination-rule.yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-room
  namespace: airbnb
spec:
  host: room
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
#    outlierDetection:
#      interval: 1s
#      consecutiveErrors: 1
#      baseEjectionTime: 10s
#      maxEjectionPercent: 100
```

* istio-injection 활성화 및 room pod container 확인

```
kubectl get ns -L istio-injection
kubectl label namespace airbnb istio-injection=enabled 
```

![Circuit Breaker(istio-enjection)](https://user-images.githubusercontent.com/38099203/119295450-d6812600-bc91-11eb-8aad-46eeac968a41.PNG)

![Circuit Breaker(pod)](https://user-images.githubusercontent.com/38099203/119295568-0cbea580-bc92-11eb-9d2b-8580f3576b47.PNG)


* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:

siege 실행

```
kubectl run siege --image=apexacme/siege-nginx -n airbnb
kubectl exec -it siege -c siege -n airbnb -- /bin/bash
```


- 동시사용자 1로 부하 생성 시 모두 정상
```
siege -c1 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

** SIEGE 4.0.4
** Preparing 1 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.49 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.05 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     256 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     256 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     256 bytes ==> POST http://room:8080/rooms
```

- 동시사용자 2로 부하 생성 시 503 에러 168개 발생
```
siege -c2 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

** SIEGE 4.0.4
** Preparing 2 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.10 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.04 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.05 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.22 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.08 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.07 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.01 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.01 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.00 secs:      81 bytes ==> POST http://room:8080/rooms

Lifting the server siege...
Transactions:                   1904 hits
Availability:                  91.89 %
Elapsed time:                   9.89 secs
Data transferred:               0.48 MB
Response time:                  0.01 secs
Transaction rate:             192.52 trans/sec
Throughput:                     0.05 MB/sec
Concurrency:                    1.98
Successful transactions:        1904
Failed transactions:             168
Longest transaction:            0.03
Shortest transaction:           0.00
```

- kiali 화면에 서킷 브레이크 확인

![Circuit Breaker(kiali)](https://user-images.githubusercontent.com/38099203/119298194-7f7e4f80-bc97-11eb-8447-678eece29e5c.PNG)


- 다시 최소 Connection pool로 부하 다시 정상 확인

```
** SIEGE 4.0.4
** Preparing 1 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms

:
:

Lifting the server siege...
Transactions:                   1139 hits
Availability:                 100.00 %
Elapsed time:                   9.19 secs
Data transferred:               0.28 MB
Response time:                  0.01 secs
Transaction rate:             123.94 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                    0.98
Successful transactions:        1139
Failed transactions:               0
Longest transaction:            0.04
Shortest transaction:           0.00

```

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌.
  virtualhost 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- room deployment.yml 파일에 resources 설정을 추가한다
![Autoscale (HPA)](https://user-images.githubusercontent.com/38099203/119283787-0a038680-bc79-11eb-8d9b-d8aed8847fef.PNG)

- room 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 50프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deployment room -n airbnb --cpu-percent=50 --min=1 --max=10
```
![Autoscale (HPA)(kubectl autoscale 명령어)](https://user-images.githubusercontent.com/38099203/119299474-ec92e480-bc99-11eb-9bc3-8c5246b02783.PNG)

- 부하를 동시사용자 100명, 1분 동안 걸어준다.
```
siege -c100 -t60S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다
```
kubectl get deploy room -w -n airbnb 
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
![Autoscale (HPA)(모니터링)](https://user-images.githubusercontent.com/38099203/119299704-6a56f000-bc9a-11eb-9ba8-55e5978f3739.PNG)

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Lifting the server siege...
Transactions:                  15615 hits
Availability:                 100.00 %
Elapsed time:                  59.44 secs
Data transferred:               3.90 MB
Response time:                  0.32 secs
Transaction rate:             262.70 trans/sec
Throughput:                     0.07 MB/sec
Concurrency:                   85.04
Successful transactions:       15675
Failed transactions:               0
Longest transaction:            2.55
Shortest transaction:           0.01
```

## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

```
kubectl delete destinationrules dr-room -n airbnb
kubectl label namespace airbnb istio-injection-
kubectl delete hpa room -n airbnb
```

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

** SIEGE 4.0.4
** Preparing 1 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'


Transactions:                   7732 hits
Availability:                  87.32 %
Elapsed time:                  17.12 secs
Data transferred:               1.93 MB
Response time:                  0.18 secs
Transaction rate:             451.64 trans/sec
Throughput:                     0.11 MB/sec
Concurrency:                   81.21
Successful transactions:        7732
Failed transactions:            1123
Longest transaction:            0.94
Shortest transaction:           0.00

```
- 배포기간중 Availability 가 평소 100%에서 87% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함

```
# deployment.yaml 의 readiness probe 의 설정:
```

![probe설정](https://user-images.githubusercontent.com/38099203/119301424-71333200-bc9d-11eb-9f75-f8c98fce70a3.PNG)

```
kubectl apply -f kubernetes/deployment.yml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Lifting the server siege...
Transactions:                  27657 hits
Availability:                 100.00 %
Elapsed time:                  59.41 secs
Data transferred:               6.91 MB
Response time:                  0.21 secs
Transaction rate:             465.53 trans/sec
Throughput:                     0.12 MB/sec
Concurrency:                   99.60
Successful transactions:       27657
Failed transactions:               0
Longest transaction:            1.20
Shortest transaction:           0.00

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


# Self-healing (Liveness Probe)
- room deployment.yml 파일 수정 
```
콘테이너 실행 후 /tmp/healthy 파일을 만들고 
90초 후 삭제
livenessProbe에 'cat /tmp/healthy'으로 검증하도록 함
```
![deployment yml tmp healthy](https://user-images.githubusercontent.com/38099203/119318677-8ff0f300-bcb4-11eb-950a-e3c15feed325.PNG)

- kubectl describe pod room -n airbnb 실행으로 확인
```
컨테이너 실행 후 90초 동인은 정상이나 이후 /tmp/healthy 파일이 삭제되어 livenessProbe에서 실패를 리턴하게 됨
pod 정상 상태 일때 pod 진입하여 /tmp/healthy 파일 생성해주면 정상 상태 유지됨
```

![get pod tmp healthy](https://user-images.githubusercontent.com/38099203/119318781-a9923a80-bcb4-11eb-9783-65051ec0d6e8.PNG)
![touch tmp healthy](https://user-images.githubusercontent.com/38099203/119319050-f118c680-bcb4-11eb-8bca-aa135c1e067e.PNG)

# Config Map/ Persistence Volume
- Persistence Volume

1: EFS 생성
```
EFS 생성 시 클러스터의 VPC를 선택해야함
```
![클러스터의 VPC를 선택해야함](https://user-images.githubusercontent.com/38099203/119364089-85048580-bce9-11eb-8001-1c20a93b8e36.PNG)

![EFS생성](https://user-images.githubusercontent.com/38099203/119343415-60041880-bcd1-11eb-9c25-1695c858f6aa.PNG)

2. EFS 계정 생성 및 ROLE 바인딩
```
kubectl apply -f efs-sa.yml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: efs-provisioner
  namespace: airbnb


kubectl get ServiceAccount efs-provisioner -n airbnb
NAME              SECRETS   AGE
efs-provisioner   1         9m1s  
  
  
  
kubectl apply -f efs-rbac.yaml

namespace를 반듯이 수정해야함

  
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: efs-provisioner-runner
  namespace: airbnb
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-efs-provisioner
  namespace: airbnb
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
     # replace with namespace where provisioner is deployed
    namespace: airbnb
roleRef:
  kind: ClusterRole
  name: efs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
  namespace: airbnb
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
  namespace: airbnb
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
    # replace with namespace where provisioner is deployed
    namespace: airbnb
roleRef:
  kind: Role
  name: leader-locking-efs-provisioner
  apiGroup: rbac.authorization.k8s.io


```

3. EFS Provisioner 배포
```
kubectl apply -f efs-provisioner-deploy.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-provisioner
  namespace: airbnb
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: efs-provisioner
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
      serviceAccount: efs-provisioner
      containers:
        - name: efs-provisioner
          image: quay.io/external_storage/efs-provisioner:latest
          env:
            - name: FILE_SYSTEM_ID
              value: fs-562f9c36
            - name: AWS_REGION
              value: ap-northeast-2
            - name: PROVISIONER_NAME
              value: my-aws.com/aws-efs
          volumeMounts:
            - name: pv-volume
              mountPath: /persistentvolumes
      volumes:
        - name: pv-volume
          nfs:
            server: fs-562f9c36.efs.ap-northeast-2.amazonaws.com
            path: /


kubectl get Deployment efs-provisioner -n airbnb
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
efs-provisioner   1/1     1            1           11m

```

4. 설치한 Provisioner를 storageclass에 등록
```
kubectl apply -f efs-storageclass.yml


kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: aws-efs
  namespace: airbnb
provisioner: my-aws.com/aws-efs


kubectl get sc aws-efs -n airbnb
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
aws-efs         my-aws.com/aws-efs      Delete          Immediate              false                  4s
```

5. PVC(PersistentVolumeClaim) 생성
```
kubectl apply -f volume-pvc.yml


apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: aws-efs
  namespace: airbnb
  labels:
    app: test-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 6Ki
  storageClassName: aws-efs
  
  
kubectl get pvc aws-efs -n airbnb
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
aws-efs   Bound    pvc-43f6fe12-b9f3-400c-ba20-b357c1639f00   6Ki        RWX            aws-efs        4m44s
```

6. room pod 적용
```
kubectl apply -f deployment.yml
```
![pod with pvc](https://user-images.githubusercontent.com/38099203/119349966-bd9c6300-bcd9-11eb-9f6d-08e4a3ec82f0.PNG)


7. A pod에서 마운트된 경로에 파일을 생성하고 B pod에서 파일을 확인함
```
NAME                              READY   STATUS    RESTARTS   AGE
efs-provisioner-f4f7b5d64-lt7rz   1/1     Running   0          14m
room-5df66d6674-n6b7n             1/1     Running   0          109s
room-5df66d6674-pl25l             1/1     Running   0          109s
siege                             1/1     Running   0          2d1h


kubectl exec -it pod/room-5df66d6674-n6b7n room -n airbnb -- /bin/sh
/ # cd /mnt/aws
/mnt/aws # touch intensive_course_work
```
![a pod에서 파일생성](https://user-images.githubusercontent.com/38099203/119372712-9736f180-bcf2-11eb-8e57-1d6e3f4273a5.PNG)

```
kubectl exec -it pod/room-5df66d6674-pl25l room -n airbnb -- /bin/sh
/ # cd /mnt/aws
/mnt/aws # ls -al
total 8
drwxrws--x    2 root     2000          6144 May 24 15:44 .
drwxr-xr-x    1 root     root            17 May 24 15:42 ..
-rw-r--r--    1 root     2000             0 May 24 15:44 intensive_course_work
```
![b pod에서 파일생성 확인](https://user-images.githubusercontent.com/38099203/119373196-204e2880-bcf3-11eb-88f0-a1e91a89088a.PNG)


- Config Map

1: cofingmap.yml 파일 생성
```
kubectl apply -f cofingmap.yml


apiVersion: v1
kind: ConfigMap
metadata:
  name: airbnb-config
  namespace: airbnb
data:
  # 단일 key-value
  max_reservation_per_person: "10"
  ui_properties_file_name: "user-interface.properties"
```

2. deployment.yml에 적용하기

```
kubectl apply -f deployment.yml


.......
          env:
			# cofingmap에 있는 단일 key-value
            - name: MAX_RESERVATION_PER_PERSION
              valueFrom:
                configMapKeyRef:
                  name: airbnb-config
                  key: max_reservation_per_person
           - name: UI_PROPERTIES_FILE_NAME
              valueFrom:
                configMapKeyRef:
                  name: airbnb-config
                  key: ui_properties_file_name
          volumeMounts:
          - mountPath: "/mnt/aws"
            name: volume
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: aws-efs
```

