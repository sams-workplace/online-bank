http://www.msaez.io/#/courses/assessment/running@cloud-final-aws-1-5


송민영


# correlation - 삭제 / 요청 데이터는 요청 실패시 삭제한다.

# 평가항목 

  Saga  
  CQRS  
  Correlation  
  Req/Resp  
  Gateway  
  Deploy / Pipeline   
  Circuit Breaker  
  Autoscale (HPA)  
  Zero-downtime deploy (Readiness Probe)   
  Config Map / Persistence Volume   
  Polyglot  
  Self-healing (Liveness Probe)
=================================================================================
# 배포 시나리오
=================================================================================

# JDK, Docker, Kubernetes 설치여부 확인

java -version
docker --version
kubectl version --short

# IAM > 사용자 > 액세스 키 만들기

# AWS CLI V2 설치

sudo ./aws/install
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo apt-get install unzip( unzip 미설치시 )
sudo ./aws/install

# IAM 보안 자격증명 설정

aws configure
Access Key : 
Secret Access Key : 
Region Name : ap-northeast-2
Default Out Format : json

# eksctl 설치

eksctl version
curl --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# 쿠버네티스 클러스터 생성

eksctl create cluster --name onlineBankCluster --version 1.17 --nodegroup-name standard-workers --node-type t3.medium --nodes 4 --nodes-min 1 --nodes-max 4

# 클러스터 current-context 설정

aws eks --region ap-northeast-2 update-kubeconfig --name onlineBankCluster
kubectl get nodes
kubectl config current-context

# 도커 클라이언트 인증

ECR > 리파지토리 생성 > 리파지토리 선택 > 푸시 명령 보기

docker login --username AWS -p $(aws ecr get-login-password --region (Region-Code)) (Account-Id).dkr.ecr.(Region-Code).amazonaws.com/

docker login --username AWS -p $(aws ecr get-login-password --region ap-northeast-2) 879772956301.dkr.ecr.ap-northeast-2.amazonaws.com/

# Metric-Server 설치

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
kubectl top node

# helm 설치 

curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
kubectl --namespace kube-system create sa tiller 
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm repo add incubator https://charts.helm.sh/incubator
helm repo update 
helm init --kube-context arn:aws:eks:ap-northeast-2:283210891307:cluster/My-Cluster --service-account tiller 
( Kube-context : kubectl config current-context 명령을 통해 알 수 있음)

# Kafka 설치

kubectl create ns kafka
helm install --name my-kafka --namespace kafka incubator/kafka
kubectl get svc my-kafka -n kafka

# Kafka topic 생성

kubectl apply -f kafka.yaml
kubectl -n kafka exec onlinebank-client -- ./bin/kafka-topics.sh --zookeeper my-kafka-zookeeper:2181 --topic onlinebank --create --partitions 1 --replication-factor 1

# siege 설치

kubectl apply -f siege.yaml
kubectl get all
kubectl exec -it siege -- /bin/bash

# git 소스 저장

git clone https://github.com/sams-workplace/online-bank.git
cd online-bank

# ECR 리파지토리 생성

aws ecr create-repository --repository-name request --region ap-northeast-2
aws ecr create-repository --repository-name auth --region ap-northeast-2
aws ecr create-repository --repository-name account --region ap-northeast-2
aws ecr create-repository --repository-name history --region ap-northeast-2
aws ecr create-repository --repository-name mypage --region ap-northeast-2
aws ecr create-repository --repository-name gateway --region ap-northeast-2
export ECR=879772956301.dkr.ecr.ap-northeast-2.amazonaws.com

# Dockerfile 파일 수정

FROM ghcr.io/gkedu/openjdk:8u212-jdk
COPY target/*SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-Xmx400M","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar","--spring.profiles.active=docker"]

# 소스 배포

cd BankRequest
mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/request:latest .
docker push ${ECR}/request:latest

cd ../BankAuthentication
mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/auth:latest .
docker push ${ECR}/auth:latest

cd ../BankAccount
mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/account:latest .
docker push ${ECR}/account:latest

cd ../BankHistory
mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/history:latest .
docker push ${ECR}/history:latest

cd ../MyPage
mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/mypage:latest .
docker push ${ECR}/mypage:latest

cd ../gateway
mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/gateway:latest .
docker push ${ECR}/gateway:latest

(참조)

kubectl create deploy request --image=${ECR}/request:latest 
kubectl expose deploy request --type="ClusterIP" --port=8080

kubectl create deploy auth --image=${ECR}/auth:latest -n mybank
kubectl expose deploy auth --type="ClusterIP" --port=8080 -n mybank

kubectl create deploy account --image=${ECR}/account:latest -n mybank
kubectl expose deploy account --type="ClusterIP" --port=8080 -n mybank

kubectl create deploy history --image=${ECR}/history:latest -n mybank
kubectl expose deploy history --type="ClusterIP" --port=8080 -n mybank

kubectl create deploy mypage --image=${ECR}/mypage:latest -n mybank
kubectl expose deploy mypage --type="ClusterIP" --port=8080 -n mybank

kubectl create deploy gateway --image=${ECR}/gateway:latest -n mybank
kubectl expose deploy gateway --type="LoadBalancer" --port=8080 -n mybank

(참조)

# BankRequest

export ECR=879772956301.dkr.ecr.ap-northeast-2.amazonaws.com
mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/request:latest .
aws ecr create-repository --repository-name request --region ap-northeast-2
docker push ${ECR}/request:latest
kubectl create deploy request --image=${ECR}/request:latest 
kubectl expose deploy request --type="LoadBalancer" --port=8080

# BankAuthentication

mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/auth:latest .
docker push ${ECR}/auth:latest
kubectl create deploy auth --image=${ECR}/auth:latest -n mybank
kubectl expose deploy auth --type="LoadBalancer" --port=8080 -n mybank

# BankAccount

mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/account:latest .
docker push ${ECR}/account:latest
kubectl create deploy account --image=${ECR}/account:latest -n mybank
kubectl expose deploy account --type="LoadBalancer" --port=8080 -n mybank

# BankHistory

mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/history:latest .
docker push ${ECR}/history:latest
kubectl create deploy history --image=${ECR}/history:latest -n mybank
kubectl expose deploy history --type="LoadBalancer" --port=8080 -n mybank

# MyPage

mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/mypage:latest .
docker push ${ECR}/mypage:latest
kubectl create deploy mypage --image=${ECR}/mypage:latest -n mybank
kubectl expose deploy mypage --type="LoadBalancer" --port=8080 -n mybank

# gateway

mvn package -Dmaven.test.skip=true
docker build -t ${ECR}/gateway:latest .
docker push ${ECR}/gateway:latest
kubectl create deploy gateway --image=${ECR}/gateway:latest -n mybank
kubectl expose deploy gateway --type="LoadBalancer" --port=8080 -n mybank

# yaml 파일을 통한 서비스 배포

kubectl apply -f request-deploy.yaml
kubectl apply -f auth-deploy.yaml
kubectl apply -f account-deploy.yaml
kubectl apply -f history-deploy.yaml
kubectl apply -f mypage-deploy.yaml
kubectl apply -f gateway-deploy.yaml

# 여기까지 정상적으로 수행한 경우 기본 프로그램 설치 및 서비스 배포가 완료됩니다.

# 테스트

http request:8080/requests accountNo="1111" requestId="01" requestName="Deposit" amountOfMoney=10000 userId="1@sk.com" userName="sam" userPassword="1234"
http gateway:8080/requests accountNo="1111" requestId="01" requestName="Deposit" amountOfMoney=10000 userId="1@sk.com" userName="sam" userPassword="1234"

###############################################################################################
# 구현
###############################################################################################

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (서비스 포트는 8081, 8082, 8083, 8084, 8084, 8088 이다)

cd BankRequest
mvn spring-boot:run

cd BankAuthentication
mvn spring-boot:run

cd BankAccount
mvn spring-boot:run

cd BankHistory
mvn spring-boot:run

cd Mypage
mvn spring-boot:run

cd gateway
mvn spring-boot:run

###############################################################################################
# DDD의 적용
###############################################################################################

1. 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. 
(예시는 request 마이크로 서비스 )

# Request.java

package onlinebank;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Request_table")
public class Request {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String accountNo;
    private String requestId;
    private String requestName;
    private Date requestDate;
    private Integer amountOfMoney;
    private String userId;
    private String userName;
    private String userPassword;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getAccountNo() {
        return accountNo;
    }

    public void setAccountNo(String accountNo) {
        this.accountNo = accountNo;
    }
    
    ...

2. Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 
데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 
자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

# RequestRepository.java

package onlinebank;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="requests", path="requests")
public interface RequestRepository extends PagingAndSortingRepository<Request, Long>{

}

3. 적용 후 REST API 의 테스트

# request 서비스의 요청처리

http request:8080/requests accountNo="1111" requestId="01" requestName="Deposit" amountOfMoney=10000 userId="1@sk.com" userName="sam" userPassword="1234"  

- 모든 요청은 request 에서 처리하는 관계로 다른 마이크로시스템에 접속하지 않는다. 

# 요청상태 확인

http http://request:8080/requests/1

###############################################################################################
# 폴리글랏 퍼시스턴스(진행중)
###############################################################################################

###############################################################################################
# 폴리글랏 프로그래밍(진행중)
###############################################################################################

###############################################################################################
# 동기식 호출 (구현)
###############################################################################################

분석단계에서의 조건 중 하나로 요청(request)->인증(auth) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 

처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 

이용하여 호출하도록 한다. 

1. 결제서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 구현

# AuthService.java

@FeignClient(name="auths", url="http://auth:8080")
public interface AuthService {
    @RequestMapping(method= RequestMethod.GET, path="/auths")
    public void requestAuth(@RequestBody Auth auth);

}

2. 요청을 받은 직후(@PostPersist) 인증을 요청하도록 처리

# Request.java

@PostPersist
    public void onPostPersist(){

        // requestId
        // "01" : Deposit
        // "02" : Withdraw
        // "03" : Balance
        // "04" : Transaction  
        // "05" : addAccount
        // "06" : deleteAccount
        if( "01".equals( getRequestId() ) || // Deposit
            "02".equals( getRequestId() ) || // Withdraw
            "05".equals( getRequestId() ) || // addAccount
            "06".equals( getRequestId() ) ){ // deleteAccount 
            onlinebank.external.Auth auth = new onlinebank.external.Auth();
            BeanUtils.copyProperties(this, auth);
            auth.setBankRequestId( getId() );
            BankRequestApplication.applicationContext.getBean(onlinebank.external.AuthService.class).requestAuth(auth);
        }
        
        // Balance
        if( "03".equals( getRequestId() ) ){ 
            BalanceRequested balanceRequested = new BalanceRequested();
            BeanUtils.copyProperties(this, balanceRequested);
            balanceRequested.setBankRequestId( getId() );
            balanceRequested.publish();
        }

        // Transaction
        if( "04".equals( getRequestId() ) ){ 
            TransactionRequested transactionRequested = new TransactionRequested();
            BeanUtils.copyProperties(this, transactionRequested);
            transactionRequested.publishAfterCommit();
        }
    }

###############################################################################################
# 비동기식 호출 
###############################################################################################

인증성공 후 계좌 시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 

계좌 처리를 위하여 인증 기능이 블로킹 되지 않아도록 처리한다.

1. 이를 위하여 인증이력에 기록을 남긴 후에 곧바로 인증완료 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)

# Auth.java

package onlinebank;

@Entity
@Table(name="Auth_table")
public class Auth {

    @PrePersist
    public void onPrePersist(){

        String userId = "1@sk.com";
        String userName = "sam";
        String userPassword = "1234";
        boolean authResult = false ;

        if( userId.equals( getUserId() ) && userName.equals( getUserName() ) && userPassword.equals( getUserPassword() ) ){
            authResult = true ;
        }

        if( authResult == false ){
            AuthCancelled authCancelled = new AuthCancelled();
            BeanUtils.copyProperties(this, authCancelled);
	    // 실패 이벤트 카프카 송출
            authCancelled.publish();

        }else{
            AuthCertified authCertified = new AuthCertified();
            BeanUtils.copyProperties(this, authCertified);

            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
                @Override
                public void beforeCommit(boolean readOnly) {
		    // 성공 이벤트 카프카 송출
                    authCertified.publish();
                }
            });
        }
    }

2. 계좌 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다

# PolicyHandler.java

package onlinebank;

@Service
public class PolicyHandler{
    @Autowired AccountRepository accountRepository;

    // 인증 성공시 정책 수신
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverAuthCertified_RecieveAccountRequest(@Payload AuthCertified authCertified){

        if(!authCertified.validate()) return;
        
        // 입금처리
        if( "01".equals( authCertified.getRequestId() ) ){

           // 입금계좌가 없을 경우 
           if( accountRepository.findByAccountNo( authCertified.getAccountNo() ) == null ){
 
               AccountRequestCancelled accountRequestCancelled = new AccountRequestCancelled();              
               accountRequestCancelled.setAccountNo(authCertified.getAccountNo());
               accountRequestCancelled.setBankRequestId(authCertified.getBankRequestId());
               accountRequestCancelled.setMessage("Account Not Exist");
               accountRequestCancelled.setRequestId(authCertified.getRequestId());
               accountRequestCancelled.setRequestName(authCertified.getRequestName());
               accountRequestCancelled.setRequestMoney(authCertified.getAmountOfMoney());
               accountRequestCancelled.setRequestDate(new Date());
               
               accountRequestCancelled.publish();
           }
           // 입금계좌가 있을 경우
           else{

               Account account = accountRepository.findByAccountNo( authCertified.getAccountNo() );
               account.setAccountNo(authCertified.getAccountNo());
               account.setRequestId(authCertified.getRequestId());
               account.setRequestName(authCertified.getRequestName());
               account.setRequestMoney(authCertified.getAmountOfMoney());
               account.setAmountOfMoney(account.getAmountOfMoney() + authCertified.getAmountOfMoney() );
               account.setRequestDate(new Date());
               accountRepository.save(account);
           }
        }

        ...

    }
}

###############################################################################################
# 운영
###############################################################################################

###############################################################################################
# 서킷 브레이킹
###############################################################################################

1. Spring FeignClient + Hystrix 옵션을 사용하여 구현

2. 요청-인증시 Request/Response 로 연동하여 구현이 되어있으며 요청이 과도할 경우 CB를 통하여 장애격리 

3. Hystrix 를 설정: 요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 

CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정

# application.yml

feign:
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

4. 인증 서비스의 임의 부하 처리 

# Auth.java (Entity)

    @PrePersist
    public void onPrePersist(){  

        ...
        
        try {
	    // 인증 데이터 저장 전 처리 시간을 400ms ~ 620ms 강제 지연
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

5. 부하테스터 seige 툴을 통한 서킷 브레이커 동작 확인

root@siege:/# siege -v -c100 -t90S -r10 --content-type "application/json" 'http://request:8080/requests POST {"accountNo":"1111","requestId":"01","requestName":"Deposit","amountOfMoney":10000,"userId":"1@sk.com","userName":"sam","userPassword":"1234"}'
( 동시사용자 100명, 90초간 진행 )

HTTP/1.1 500     4.46 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     3.88 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     3.69 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     3.88 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     3.85 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     3.60 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     3.76 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     4.09 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     3.62 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     4.14 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     3.59 secs:     250 bytes ==> POST http://request:8080/requests

HTTP/1.1 201     4.40 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.33 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.45 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.35 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.38 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.45 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.51 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.57 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.02 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.63 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.05 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.03 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.01 secs:     370 bytes ==> POST http://request:8080/requests

HTTP/1.1 500     4.31 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     4.14 secs:     250 bytes ==> POST http://request:8080/requests

HTTP/1.1 201     4.09 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.15 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.14 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.10 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.15 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.24 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.20 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.24 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.26 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.16 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.30 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.20 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.24 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.29 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 500     4.32 secs:     250 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     4.18 secs:     370 bytes ==> POST http://request:8080/requests

Lifting the server siege...
Transactions:                   1545 hits
Availability:                  71.40 %
Elapsed time:                  89.88 secs
Data transferred:               0.69 MB
Response time:                  5.66 secs
Transaction rate:              17.19 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   97.34
Successful transactions:        1545
Failed transactions:             619
Longest transaction:           11.60
Shortest transaction:           0.01
 
운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 
동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

###############################################################################################
# 오토스케일 아웃
###############################################################################################

# 앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 

이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.

결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 

설정은 CPU 사용량이 50프로를 넘어서면 replica 를 10개까지 늘려준다

root@labs-579721623:/home/project/online-bank/BankAuthentication# kubectl autoscale deployment request --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/request autoscaled

# 부하 테스트 진행

root@siege:/# siege -v -c100 -t90S -r10 --content-type "application/json" 'http://request:8080/requests POST {"accountNo":"1111","requestId":"01","requestName":"Deposit","amountOfMoney":10000,"userId":"1@sk.com","userName":"sam","userPassword":"1234"}'
( 동시사용자 100명, 90초간 진행 )
                                                                                          
HTTP/1.1 201     1.63 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.98 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.80 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.09 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.90 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.89 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.18 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.19 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     1.71 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.10 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.80 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.81 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.82 secs:     370 bytes ==> POST http://request:8080/requests

Lifting the server siege...
Transactions:                   8988 hits
Availability:                 100.00 %
Elapsed time:                  89.51 secs
Data transferred:               3.17 MB
Response time:                  0.99 secs
Transaction rate:             100.41 trans/sec
Throughput:                     0.04 MB/sec
Concurrency:                   99.42
Successful transactions:        8989
Failed transactions:               0
Longest transaction:            8.59
Shortest transaction:           0.01

# Terminal 을 추가하여 오토스케일링 현황을 모니터링 한다. ( watch kubectl get pod )

# 부하 테스트 진행전 

Every 2.0s: kubectl get pod       labs-579721623: Thu Aug 19 08:44:47 2021

NAME                              READY   STATUS    RESTARTS   AGE
account-6b844c4f44-gdsvd          1/1     Running   0          145m
auth-7c55b8b7b9-9r6bb             1/1     Running   0          145m
efs-provisioner-fbcc88cb8-zrlzx   1/1     Running   0          79m
gateway-55bd75dfb9-cwlvg          1/1     Running   0          142m
history-77cc54b895-v5nqm          1/1     Running   0          144m
mypage-7bc648bd4d-5psgz           1/1     Running   0          143m
request-675f455d5c-7txbc          1/1     Running   0          28m

# 부하 테스트 진행 후 

Every 2.0s: kubectl get pod       labs-579721623: Thu Aug 19 08:46:34 2021

NAME                              READY   STATUS    RESTARTS   AGE
account-6b844c4f44-gdsvd          1/1     Running   0          147m
auth-7c55b8b7b9-9r6bb             1/1     Running   0          147m
efs-provisioner-fbcc88cb8-zrlzx   1/1     Running   0          81m
gateway-55bd75dfb9-cwlvg          1/1     Running   0          144m
history-77cc54b895-v5nqm          1/1     Running   0          146m
mypage-7bc648bd4d-5psgz           1/1     Running   0          144m
request-675f455d5c-256tz          0/1     Running   0          31s
request-675f455d5c-6s2nz          0/1     Running   0          46s
request-675f455d5c-7txbc          1/1     Running   0          30m
request-675f455d5c-bz4nq          0/1     Running   0          46s
request-675f455d5c-mdbbl          0/1     Running   0          46s
siege                             1/1     Running   0          3h19m

# 부하테스트 결과 Availability 는 100% 를 보이며 성공하였고, 늘어난 pod 개수를 통하여

오토 스케일링이 정상적으로 수행되었음을 확인할 수 있다. 

###############################################################################################
# 무정지 재배포
###############################################################################################

# 무정지 재배포 여부를 확인을 위해서 Autoscaler 와 CB 설정을 제거한다.

root@siege:/# siege -v -c100 -t90S -r10 --content-type "application/json" 'http://request:8080/requests POST {"accountNo":"1111","requestId":"01","requestName":"Deposit","amountOfMoney":10000,"userId":"1@sk.com","userName":"sam","userPassword":"1234"}'
( 동시사용자 100명, 90초간 진행 )

# 부하테스트중 추가 생성한 Terminal 에서 readiness 설정되지 않은 버젼으로 재배포 한다.

root@labs-579721623:/home/project/online-bank/yaml# kubectl apply -f request-redeploy.yaml
deployment.apps/request configured
service/request configured

[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
HTTP/1.1 201     0.83 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.86 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.84 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     1.50 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.94 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.91 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.94 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     1.64 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.81 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     1.43 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.88 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.93 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     1.40 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.89 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     1.42 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.94 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.85 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.85 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.90 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.93 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.80 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.92 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.94 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.94 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.97 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.97 secs:     370 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.91 secs:     370 bytes ==> POST http://request:8080/requests
siege aborted due to excessive socket failure; you
can change the failure threshold in $HOME/.siegerc

Transactions:                   1154 hits
Availability:                  51.38 %
Elapsed time:                  12.57 secs
Data transferred:               0.41 MB
Response time:                  1.06 secs
Transaction rate:              91.81 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                   97.58
Successful transactions:        1154
Failed transactions:            1092
Longest transaction:            4.69
Shortest transaction:           0.02

# 부하테스트중 새로 배포된 서비스를 READY 상태로 인지하여 서비스 중단됨을 확인함.

# 부하테스트 진행

root@siege:/# siege -v -c100 -t30S -r10 --content-type "application/json" 'http://request:8080/requests POST {"accountNo":"1111","requestId":"01","requestName":"Deposit","amountOfMoney":10000,"userId":"1@sk.com","userName":"sam","userPassword":"1234"}'
( 동시사용자 100명, 90초간 진행 )

# 부하테스트중 추가 생성한 Terminal 에서 readiness 설정 되어있는 버젼으로 재배포 한다.

root@labs-579721623:/home/project/online-bank/yaml# kubectl apply -f request-deploy.yaml
deployment.apps/request configured
service/request unchanged

HTTP/1.1 201     0.37 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.56 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.38 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.38 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.36 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.35 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.36 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.34 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.36 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.37 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.35 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.05 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.33 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.05 secs:     372 bytes ==> POST http://request:8080/requests
HTTP/1.1 201     0.08 secs:     372 bytes ==> POST http://request:8080/requests

Lifting the server siege...
Transactions:                  24451 hits
Availability:                 100.00 %
Elapsed time:                  89.32 secs
Data transferred:               8.65 MB
Response time:                  0.36 secs
Transaction rate:             273.75 trans/sec
Throughput:                     0.10 MB/sec
Concurrency:                   99.31
Successful transactions:       24451
Failed transactions:               0
Longest transaction:            2.72
Shortest transaction:           0.00

# 배포중 Availability 100%를 보이며 무정지 재배포가 정상적으로 성공하였다.

###############################################################################################
# Gateway / Corelation
###############################################################################################

# Gateway 기능이 정상적으로 수행되는지 확인하기 위하여 Gateway를 통하여 요청서비스를 호출한다.  

root@siege:/# http gateway:8080/requests accountNo="1111" requestId="01" requestName="Deposit" amountOfMoney=10000 userId="1@sk.com" userName="sam" userPassword="1234"

HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Thu, 19 Aug 2021 06:54:01 GMT
Location: http://request:8080/requests/2
transfer-encoding: chunked

{
    "_links": {
        "request": {
            "href": "http://request:8080/requests/2"
        },
        "self": {
            "href": "http://request:8080/requests/2"
        }
    },
    "accountNo": "1111",
    "amountOfMoney": 10000,
    "requestDate": null,
    "requestId": "01",
    "requestName": "Deposit",
    "userId": "1@sk.com",
    "userName": "sam",
    "userPassword": "1234"
}

# 요청 처리결과를 통하여 Gateway 기능이 정상적으로 수행되었음을 확인할 수 있다. 

# 요청이 정상적으로 처리되지 않는 경우( 예를 들어서 입금 요청을 했으나 계좌가 존재하지 않는 등 )

요청시 파라미터로 전송된 id 값을 기준으로 기 저장된 요청 데이터를 삭제한다. 

Gateway 테스트시 존재하지 않는 계좌에 입금을 시도하였으며 요청이 정상적으로 처리되지 못한 관계로

기 저장된 데이터가 삭제 처리 된다. 

root@siege:/# http http://request:8080/requests

HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Thu, 19 Aug 2021 06:54:56 GMT
Transfer-Encoding: chunked

{
    "_embedded": {
        "requests": []
    },
    "_links": {
        "profile": {
            "href": "http://request:8080/profile/requests"
        },
        "self": {
            "href": "http://request:8080/requests{?page,size,sort}",
            "templated": true
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 0,
        "totalPages": 0
    }
}

# request 데이터가 정상적으로 삭제되었음을 확인할 수 있다. 

###############################################################################################
# 동기식 호출 (운영)
###############################################################################################

# 동기식 호출인 관계로 인증시스템 장애시 서비스를 처리할 수 없다. 

1) 인증 서비스 임시로 삭제한다. 

root@labs-579721623:/home/project/online-bank/yaml# kubectl delete service auth
service "auth" deleted

2) 요청 처리결과를 확인한다.

root@siege:/# http request:8080/requests accountNo="1111" requestId="01" requestName="Deposit" amountOfMoney=10000 userId="1@sk.com" userName="sam" userPassword="1234"
HTTP/1.1 500 
Connection: close
Content-Type: application/json;charset=UTF-8
Date: Thu, 19 Aug 2021 06:59:08 GMT
Transfer-Encoding: chunked

{
    "error": "Internal Server Error",
    "message": "Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction",
    "path": "/requests",
    "status": 500,
    "timestamp": "2021-08-19T06:59:08.624+0000"
}

3) 인증서비스 재기동 한다. 

root@labs-579721623:/home/project/online-bank/yaml# kubectl expose deploy auth --type="LoadBalancer" --port=8080
service/auth exposed

4) 요청처리 결과를 확인한다. 

root@siege:/# http http://request:8080/requests accountNo="1111" requestId="01" requestName="Deposit" amountOfMoney=10000 userId="1@sk.com" userName="sam" userPassword="1234"

HTTP/1.1 201 
Content-Type: application/json;charset=UTF-8
Date: Thu, 19 Aug 2021 07:02:31 GMT
Location: http://request:8080/requests/4
Transfer-Encoding: chunked

{
    "_links": {
        "request": {
            "href": "http://request:8080/requests/4"
        },
        "self": {
            "href": "http://request:8080/requests/4"
        }
    },
    "accountNo": "1111",
    "amountOfMoney": 10000,
    "requestDate": null,
    "requestId": "01",
    "requestName": "Deposit",
    "userId": "1@sk.com",
    "userName": "sam",
    "userPassword": "1234"
}

# 테스트를 통하여 인증 서비스가 기동되지 않은 상태에서는 업무 요청이 실패함을 확인 할 수 있음.

###############################################################################################
# Persistence Volume
###############################################################################################

# Persistence Volume 을 생성한다. 

root@labs-579721623:/home/project/online-bank/yaml# kubectl get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS   REASON   AGE
pvc-60c0deaa-241e-443d-a770-2c4890b0d9db   1Gi        RWO            Delete           Bound    kafka/datadir-my-kafka-2   gp2                     174m
pvc-ce2fe4aa-be29-4c82-8637-7d247b243456   1Gi        RWO            Delete           Bound    kafka/datadir-my-kafka-1   gp2                     175m
pvc-f0331c5b-0127-475f-93db-58999bb38980   1Gi        RWO            Delete           Bound    kafka/datadir-my-kafka-0   gp2                     177m
task-pv-volume                             100Mi      RWO            Retain           Bound    labs-579721623/aws-efs     aws-efs                 4m4s

# Persistence Volume Claim 을 생성한다. 

root@labs-579721623:/home/project/online-bank/yaml# kubectl get pvc

NAME      STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
aws-efs   Bound    task-pv-volume   100Mi      RWO            aws-efs        101s

# Pod 로 접속하여 파일시스템 정보를 확인한다. 

root@labs-579721623:/home/project/online-bank/yaml# kubectl get pod

NAME                              READY   STATUS             RESTARTS   AGE
account-6b844c4f44-gdsvd          1/1     Running            0          76m
auth-7c55b8b7b9-9r6bb             1/1     Running            0          76m
efs-provisioner-fbcc88cb8-zrlzx   1/1     Running            0          10m
gateway-55bd75dfb9-cwlvg          1/1     Running            0          73m
history-77cc54b895-v5nqm          1/1     Running            0          75m
mypage-7bc648bd4d-5psgz           1/1     Running            0          73m
request-5cdc6474bf-p76tr          0/1     ImagePullBackOff   0          25m
request-646c4cc7c6-xmk59          1/1     Running            0          28m
siege                             1/1     Running            0          128m

root@labs-579721623:/home/project/online-bank/yaml# kubectl exec -it request-646c4cc7c6-xmk59 -- /bin/bash

# 생성된 Persistence Volume 은 Mount 되지 않은 상태임을 확인한다. 

root@request-646c4cc7c6-xmk59:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          80G  4.2G   76G   6% /
tmpfs            64M     0   64M   0% /dev
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/nvme0n1p1   80G  4.2G   76G   6% /etc/hosts
shm              64M     0   64M   0% /dev/shm
tmpfs           1.9G   12K  1.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           1.9G     0  1.9G   0% /proc/acpi
tmpfs           1.9G     0  1.9G   0% /sys/firmware

# Persistenct Volume 이 Mount 되도록 yaml 설정파일을 변경한다. 

# request-deploy-vol.yaml

    spec:
      containers:
        - name: request
          image: 879772956301.dkr.ecr.ap-northeast-2.amazonaws.com/request
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
...
          volumeMounts:
          - mountPath: "/mnt/aws"
            name: volume
...
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: aws-efs

# 변경된 yaml 파일로 서비스 재배포 한다. 
root@labs-579721623:/home/project/online-bank/yaml# kubectl apply -f request-deploy-vol.yaml
deployment.apps/request created

# Pod 로 접속하여 파일시스템 정보를 확인한다. 

root@request-675f455d5c-t8lzd:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          80G  4.1G   76G   6% /
tmpfs            64M     0   64M   0% /dev
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/nvme0n1p1   80G  4.1G   76G   6% /mnt/aws
shm              64M     0   64M   0% /dev/shm
tmpfs           1.9G   12K  1.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           1.9G     0  1.9G   0% /proc/acpi
tmpfs           1.9G     0  1.9G   0% /sys/firmware

# 생성된 Persistence Volume 이 pod 내 정상 mount 되었음을 확인할 수 있다. 

###############################################################################################
# Liveness Prove
###############################################################################################

# request 서비스 배포시 yaml 파일내 Liveness Prove 설정을 추가한다. 

# request-deploy.yaml

spec:
  replicas: 1
  selector:
    matchLabels:
      app: request
  template:
    metadata:
      labels:
        app: request
    spec:
      containers:
        - name: request
          image: 879772956301.dkr.ecr.ap-northeast-2.amazonaws.com/request
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
...
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5

# Liveness Prove 설정 정상 적용여부를 확인하기 위해서 기존에 생성된 request pod 삭제시

정상적으로 재생성 되는지 여부를 확인한다. 

root@labs-579721623:/home/project/online-bank/yaml# kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
account-6b844c4f44-gdsvd          1/1     Running   0          84m
auth-7c55b8b7b9-9r6bb             1/1     Running   0          85m
efs-provisioner-fbcc88cb8-zrlzx   1/1     Running   0          19m
gateway-55bd75dfb9-cwlvg          1/1     Running   0          82m
history-77cc54b895-v5nqm          1/1     Running   0          84m
mypage-7bc648bd4d-5psgz           1/1     Running   0          82m
request-675f455d5c-t8lzd          1/1     Running   0          2m10s
siege                             1/1     Running   0          137m

# request pod 를 삭제한다. 

root@labs-579721623:/home/project/online-bank/yaml# kubectl delete pod request-675f455d5c-t8lzd
pod "request-675f455d5c-t8lzd" deleted

# request pod 삭제 후 pod 정보를 재조회 한다. 

root@labs-579721623:/home/project/online-bank/yaml# kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
account-6b844c4f44-gdsvd          1/1     Running   0          85m
auth-7c55b8b7b9-9r6bb             1/1     Running   0          85m
efs-provisioner-fbcc88cb8-zrlzx   1/1     Running   0          19m
gateway-55bd75dfb9-cwlvg          1/1     Running   0          83m
history-77cc54b895-v5nqm          1/1     Running   0          85m
mypage-7bc648bd4d-5psgz           1/1     Running   0          83m
request-675f455d5c-zqhwq          0/1     Running   0          13s
siege                             1/1     Running   0          138m

# request pod Liveness Prove 설정이 적용되어 삭제 후 다른 이름으로 재생성 되었음을 확인할 수 있다. 
