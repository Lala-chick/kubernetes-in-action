# 디플로이먼트: 선언적 애플리케이션 업데이트

쿠버네티스 클러스터에서 실행되는 애플리케이션을 업데이트하는 방법과 k8s가 어떻게 무중단 업데이트 프로세스로 전환하는데 도움을 주는 지 알아보는 장

## 9.1 파드에서 실행 중인 애플리케이션 업데이트
새로운 애플리케이션 개발해 v2로 태그가 지정된 새 이미지를 만든 후 이미지 저장소에 푸시한 후, pod를 새버전으로 변경하고자 할 때, 2가지 방법으로 업데이트 가능
- 기존 pod을 모두 삭제한 후, 새 pod을 시작
  - 짧은 시간동안 애플리케이션 사용불가
- 새 pod을 시작하고, 기동하면 기존파드를 삭제
  - 동시에 2가지 버전을 실행해야함.
  - 애플리케이션이 데이터 저장소에 데이터를 저장하는 경우 새 버전이 이전 버전을 손상시킬수 있는 데이터 스키마나 데이터의 수정을 해서는 안됨

### 9.1.1 기존 pod을 삭제하고 새 pod 시작
<img width="934" alt="image" src="https://media.oss.navercorp.com/user/26312/files/55831137-6ae6-48cd-a7ed-0938ba62dc6d">

1. replicaset 생성  
`kubectl create -f kubia-replicaset.yaml`
2. template 수정 
- image: khosungpil/kubia - >reg.navercorp.com/k8s_study/kubia:study  
`kubectl edit rs kubia`
3. pod삭제 후 재생성  
`kubectl delete pod kubia-`

### 9.1.2 새 pod 기동과 이전 pod 삭제

새 버전을 실행하는 포드를 가져오는 동안 초기 버전의 포드만 서비스와 연결 후 새 포드가 모두 올라오면 아래 그림과 같이 서비스의 라벨 셀렉터를 변경해 서비스를 새 포드로 전환 할 수 있다.

이를 BlueGreen Deployment 라고 함.  
- 한번에 전환  
<img width="933" alt="image" src="https://media.oss.navercorp.com/user/26312/files/c18d265b-6b97-4ae9-abef-567313e63330">
- 롤링 업데이트  
<img width="942" alt="image" src="https://media.oss.navercorp.com/user/26312/files/e91f93ef-a0c7-42af-86b0-a76c6d7ef734">

## 9.2 레플리케이션컨트롤러로 자동 롤링 업데이트 수행
1. 서비스와 레플리케이션컨트롤러 실행  
`kubectl create -f kubia-rc-and-service-v1.yaml`  
2. 서비스 외부 ip 확인  
`kubectl get svc kubia`   
3. curl을 반복해 서비스 호출  
`while true; do curl http://10.105.117.77:8080; done`  
4. kubectl을 이용해 rc의 롤링 업데이트를 시작  
`kubectl rolling-update kubia-v1 kubia-v2 --image=reg.navercorp.com/k8s_study/kubia:v2 --v 6`

### 롤링업데이트가 수행되는 과정
1. 신규 레플리케이션컨트롤러 생성  
1-1. 기존 레플리케이션컨트롤러 아래 파드 레이블을 변경  
1-2. 기존 레플리케이션컨트롤러의 레이블 셀렉터를 변경  
1-3. 기존 레플리케이션컨트롤러의 파드 탬플릿에서 이미지 부분과 레이블 셀렉터 등을 변경해 신규 rc 생성  
`watch -n 1 'kubectl get po --show-labels'`  
<img width="683" alt="image" src="https://media.oss.navercorp.com/user/26312/files/4651ed97-5ee9-4309-828c-b80f7323b411">  
2. 신규 컨트롤러를 스케일업 하고 기존 것은 스케일다운해 파드를 하나씩 교체한다.  
2-1. 신규 컨트롤러가 첫 파드를 생성한다.  
2-2. 이전 레플리케이션 컨트롤러를 스케일 다운한다.  
-> 이 과정을 전체 파드가 교체될 때까지 반복한다.  
<img width="591" alt="image" src="https://media.oss.navercorp.com/user/26312/files/a16702e9-f006-481c-8e7d-c3841c8addcc">
### 롤링업데이트가 deprecated 된 이유
- 쿠버네티스가 임의로 파드와 레플리케이션 컨트롤러의 레이블 및 레이블 셀렉터를 수정
- 모든 단계를 kubectl client가 수행. 
  - 서버가 아닌 클라이언트가 업데이트 프로세스를 수행한다는 것은, 중도에 네트워크 연결이 중단될 경우 업데이트 프로세스도 중단되어 업데이트 상태와 기존 상태의 중간에 해당하는 애매한 지점에서 머무르게 된다. 

## 9.3 애플리케이션을 선언적으로 업데이트 하기 위한 디플로이먼트 사용하기
디플로이먼트란?  
  - low-level개념으로 간주되는 레플리케이션컨트롤러 또는 레플리카셋을 통해 수행하는 대신 애플리케이션을 배포하고 선언적으로 업데이트하기 위한 high-level의 리소스  

디플로이먼트를 왜 쓸까?
   - 애플리케이션을 업데이트할 때 추가적인 레플리케이션컨트롤러를 도입하고 두 컨트롤러가 잘 조화되도록 조정해야하기 때문

### 9.3.1 디플로이먼트 생성
#### 디플로이먼트 리소스 생성
`kubectl create -f kubia-deployment-v1.yaml --record`
#### 디플로이먼트 롤아웃 상태 출력
디플로이먼트 상태를 확인하는 명령어
`kubectl rollout status deployment kubia`
#### 디플로이먼트가 레플리카셋을 생성하는 방법과 레플리카셋이 파드를 생성하는 방식
- 레플리케이션컨트롤러를 이용해 파드를 생성할 때 컨트롤러의 이름과 임의로 생성된 문자열로 구성되어 있었음(ex: kubia-v1-m33mv)
- 디플로이먼트에서 생성한 파드 중간에는 이름 중간에 값이 추가로 포함
  - 디플로이먼트와 파드 템플릿의 해시값을 의미

### 9.3.2 디플로이먼트 업데이트
이전 방법은 rolluing-update 를 사용해 작업이 끝나기를 CLI 앞에서 기다려야 했고 rc 를 대체하는 새 rc 의 이름을 지정하는 작업을 해야했다(kubia-v1 → kubia-v2)  
이제는 deployment template 만 수정하면 k8s 가 실제 시스템 상태를 resource 에 정의된 상태로 만드는 데 필요한 모든 단계를 수행한다.  
deployment pod 템플릿에서 새 이미지 태그를 참조해 시스템이 원하는 상태가 되도록 k8s 에 맡긴다.  
#### 사용가능 디플로이먼트 전략
- rolling update
  - 이전 pod를 하나씩 제거하면 새로 생성
  - 두 버전이 동시에 동작 할 수 있을 때 사용
- recreate
  - 이전 pod를 모두 삭제
  - 여러 버전을 벙렬로 실행하는 것을 지원하지 않고 새 버전을 시작하기 전에 이전 버전을 완전히 중지하는 경우
#### 롤링업데이트
요청 진행 상황 확인  
`while true; do curl http://10.105.117.77:8080; done`   
디플로이먼트 수정  
`kubectl set image deployment kubia nodejs=reg.navercorp.com/k8s_study/kubia:v2`
<img width="672" alt="image" src="https://media.oss.navercorp.com/user/26312/files/962ecb6f-fb6d-43a2-976f-9741e8827f73">  
레플리카셋 조회 시 기존 레플리카셋과 새 레플리카셋을 모두 볼 수 있음  
`kubectl get rs`

- 디플로이먼트 리소스에 정의된 pod 템플릿을 수정하면 자동적으로 변경작업 수행
- 리소스 수정방식
  - kubectl patch : 속성 조절을 위해서는 kubectl patch 명령어를 통해 리소스 속성 일부를 수정할 수 있다. 오브젝트 개별 속성을 수정하는 데에 사용한다.
  - kubectl edit : 기본 편집기로 오브젝트 매니페스트를 오픈해 변경하면 오브젝트가 업데이트된다.
  - kubectl apply : 전체 yaml, json 파일 속성 값을 적용해 수정한다. 리소스 전체 정의를 포함한다. 오브젝트 없으면 신규 생성한다.
  - kubectl replace : yaml, json 파일로 오브젝트 전체를 새 것으로 교체한다. apply와 달리 오브젝트가 없으면 오류가 발생한다.
  - kubectl set image : 파드, 레플리케이션 컨트롤러의 정의 상 이미지를 변경한다.

### 9.3.3 디플로이먼트 롤백
쿠버네티스에 디플로이먼트의 마지막 롤아웃을 취소하도록 지시해서 이전에 배포된 버전으로 롤백 가능  
`kubectl rollout undo deployment kubia`  

롤아웃 이력도 표시 가능  
`kubectl rollout history deployment kubia`

특정 디플로이먼트 개정으로 롤백  
`kubectl rollout undo deployment kubia --to-revision=1`  

### 9.3.4 롤아웃 속도 제어
#### maxSurge와 maxUnavailable 속성
- maxSurge : 디플로이먼트가 의도하는 레플리카 수보다 얼마나 많은 초과치를 허용할 것인가. (기본값=25%, 반올림. 다만 비율 외 절댓값 지정도 가능)
- maxUnavailable : 업데이트 과정에서 의도하는 레플리카 수를 기준으로 사용할 수 없는 수를 설정. 해당 수치만큼의 파드만 사용 불가능 상태여야 한다는 것. (기본값=25%, 25%의 수는 사용할 수 없고 최소 75%가 사용 가능한 파드 인스턴스 수가 되도록 유지되어야. 비율 외 절댓값 지정도 가능)
- replicas=3, maxSurge=1, maxUnavailable=0
![image](https://media.oss.navercorp.com/user/26312/files/3a15a6e6-2e71-4e42-815f-a7a2d15ded25)
- replicas=3, maxSurge=1, maxUnavailable=1
![image](https://media.oss.navercorp.com/user/26312/files/fd5742e4-975b-450d-990f-a92ab4859e0d)  

### 9.3.5 롤아웃 프로세스 일시 중지
- 버그가 있는지 확인을 위해 롤아웃 프로세스 도중 일시중지 가능  
`kubectl rollout pause deployment kubia`

일시정지를 할 경우 새로운 pod가 생성되었지만 원본 pod도 계속 실행중이어야 한다. 새파드가 가동되면 서비스에 관한 요청의 일부가 새 pod로 전달된다. 이 경우 카나리 릴리스를 효과적으로 실행할 수 있다. 새 버전이 제대로 작동하는지 확인한 후 롤아웃을 계속 진행하거나 이전버전으로 롤백한다.

- 롤아웃 재개  
`kubectl rollout resume deployment kubia`

### 9.3.6 잘못된 버전의 롤아웃 방지
minReadySecond 속성을 통해 롤아웃 속도를 조절 가능  
minReadySecond는 파드를 사용가능한 것으로 취급하기 전에 새로 만든 파드를 준비할 시간을 지정한다. 
-> pod가 사용 가능할 때까지 롤아웃 프로세스가 계속되지 않는다.  
모든 파드가 readiness probe 가 성공하면 파드로 준비되고
readiness probe 에 문제가 생기면 새 파드가 제대로 작동하지 않고 새 버전의 rollout 을 차단한다.
#### v3가 완전히 롤아웃 되는것을 방지하기 위한 레디니스 프로브 정의
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  selector:
    matchLabels:
      app: kubia
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - name: nodejs
        image: reg.navercorp.com/k8s_study/kubia:v3
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        readinessProbe:
          periodSeconds: 1
          httpGet:
            path: /
            port: 8080
        ports:
        - containerPort: 8080
          protocol: TCP
```
#### kubectl apply를 통한 디플로이먼트 업데이트
이미지와 레디니스프로브를 한번에 추가하기 위해 `kubectl apply`를 사용  
`kubectl apply -f kubia-deployment-v3-with-readinesscheck.yaml`  

apply를 통해 업데이트가 시작되고 rollout status 명령어를 실행  
`kubectl rollout status deployment kubia`  

파드가 생성되었다고 표시가 되어있으므로 해당 서비스를 호출해 본다.  
`while true; do curl http://10.105.117.77:8080; done`  

v3 pod에 접근이 안되는 것을 확인 할 수 있다. `kubectl get pod`를 통해 pod 조회시 pod이 생성이 안되는 것을 확인 가능하다.  
v3은 5번째 요청 까지만 200을 반환하고, 그 이후 부터는 500을 반환하여 readiness probe 가 실패하게 된다. 결과적으로 pod 는 서비스에서 제외된다. client 가 제대로 작동하지 않는 pod 에 접속하지 않도록 한다.  
<img width="710" alt="image" src="https://media.oss.navercorp.com/user/26312/files/8771bd8f-ebe4-43ac-851d-8ae4644c0f74">

그러나 process roll-out 은 status 명령어로 봤을때 멈춰있음을 확인할 수 있다.  
사용 가능한 것으로 간주되려면 10초 이상 준비돼 있어야 한다
사용가능할 때까지 roll-out process 는 새 파드를 만들지 않으며 maxUnavalialbe 속성을 0으로 설정했기 때문에 원래 파드도 제거하지 않는다.
사실상 배포 중단된 상태

#### 롤아웃 데드라인 설정
기본적으로 롤아웃이 10분동안 진행되지 않으면 실패한것으로 간주. 아래의 명령어를 통해 ProgressDeadlineExceeded조건 표시가능
`kubectl describe deploy kubia`

#### 잘못된 롤아웃 중지
`kubectl rollout undo deployment kubia`

- extensions/v1beta1에서는 progressDeadlineSeconds에 지정된 시간이 지나면 중지요청을 보냈지만 이후 버전에서는 자동으로 중단
- extensions/v1부터 spec.progressDeadlineSeconds를 통해 데드라인 설정 가능(default: 600)