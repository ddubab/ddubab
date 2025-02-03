# [티맥스비아이] ML Platform 프로젝트


### 소개

> 쿠버네티스 환경에서 정형 데이터를 학습시키고  학습된 모델을 실시간으로 서빙 해 주는 플랫폼
> 

### 팀원 및 기간

- 프론트 2명, 백엔드 3명, 모델러 1명
- 2024.01~2024.08

### 기술

- Backend Server – Spring Boot Server
- Meta DB, Schama Patch – MySQL, flyway
- ML Train Engine – Argo Workflow on K8s
- ML Serving – Kserve (Knative + Istio) on K8s

※ 백엔드 서버-K8s 리소스 통신 dependency – Java fabric8 kubernetes client API

![version](/version.png)

### 담당 업무

**[학습 파트 백엔드]**

1. Outbox 패턴을 이용한 이벤트 핸들링
    - K8s 클러스터와 백엔드 서버 간 통신 안정성을 위해 Outbox 테이블을 설계하고 이를 구현하여 데이터 일관성을 보장하였습니다.
2. K8s API 서버와 백엔드 서버 간 통신 구현
    - Fabric8 Kubernetes-client 패키지를 활용해 외부 서버와의 안정적인 통신을 구현하였습니다.
3. Fabric8 패키지를 활용한 백엔드 기능 개발
    - Argo Workflows의 CRD(Workflow 리소스)를 생성하는 인터페이스를 설계하여 DAG 기반 학습 생성 기능을 개발하였습니다.
    - Watcher를 통해 Workflow 리소스 상태를 실시간으로 감지하고 상태를 업데이트하는 모니터링 기능을 구현하였습니다.
    - PVC에 학습 데이터를 업로드하는 기능을 설계해 사용자 파일 처리의 효율성을 향상시켰습니다.

[Helm Chart Manifests 작성]

- Minikube, Argo, KServe, Knative, Istio 모듈 설치를 위한 Helm Chart를 작성하고 이를 통해 개발자들이 동일한 환경에서 작업할 수 있도록 표준화된 개발 환경을 구축하였습니다.


### 학습 서비스 생성 흐름

![createTrainEvent](/createTrainEvent.png)

### 서빙 서비스 생성 흐름

![createServingEvent](/createServingEvent.png)


## 트러블 슈팅

### 1. 모델 배포 업데이트

문제 상황 

- 학습 과정을 통해 모델을 새롭게 생성하는데 Kserve가 제공하는 InferenceService에는                   배포 할 모델이 사전 정의되어 있어야 함
 → 새롭게 학습한 모델을 배포하기 위해서는 iSVC를 계속 새롭게 생성해야 함

해결 

- Kserve에서 제공하는 멀티모델서빙 방식을 도입
→ TrainedModel CRD를 추가해 유동적으로 학습된 모델을 추가할 수 있도록 변경

성과 :

- iSVC 생성의 반복적인 작업을 제거함으로 배포 프로세스 간소화
- 배포 속도를 개선하여, 운영 효율성 향상

![beforeOne](/beforeOne.png)

![afterOne](/afterOne.png)



### 2. 아르고 워크플로우 구조 세분화

문제 상황  

- 모델 학습 코드를 단일 이미지로 실행하는 구조로 이로 인해 학습 상태 모니터링이 어려워 Python 코드 내부에서 백엔드 서버로 학습 상태를 전달하는 로직을 추가하여야 했음
- 모델 학습 과정을 세분화하지 않아 테스트 및 디버깅이 비효율적

해결 

- 학습 과정을 Preprocessing, Feature Engineering, Training 단계로 세분화하고, 각각을 별도의 이미지로 분리
- Argo 워크플로우의 DAG(Directed Acyclic Graph) 구조를 활용하여 각 단계별로 순서를 정의하고 순서에 맞게 분리된 이미지를 호출하도록 수정
- 학습 상태 모니터링을 Python 코드 레벨에서 수행하지 않고, Argo 워크플로우에서 생성하는 Pod 리소스 단위로 처리하도록 변경

성과 :

- 학습 상태를 Pod 단위로 모니터링할 수 있어, 상태 확인 및 관리 용이
- 학습 과정을 세분화함으로써 모델 테스트와 디버깅 효율성 향상

![beforeTwo](/beforeTwo.png)

![afterTwo](/afterTwo.png)



### 3. 서빙 생성 에러

문제 상황 

- 서빙 생성 요청 시, 다음과 같은 에러 발생 
`io.fabric8.kubernetes.client.KubernetesClientException: Failure executing: POST at: https://192.168.49.2:8443/apis/serving.kserve.io/v1beta1/namespaces/gaia-ax-mvp/inferenceservices. Message: Forbidden! User minikube doesn't have permission. admission webhook "inferenceservice.kserve-webhook-server.validator" denied the request: Exactly one of [SKLearn, XGBoost, Tensorflow, PyTorch, Triton, ONNX, PMML, LightGBM, Paddle, PodSpec] must be specified in PredictorSpec.`

해결 

- 권한 관련 혹은 요청 시 생성해주는 yaml 파일의 오류 문제일 것이라 생각
    - 사용 중인 서비스 어카운트에 권한을 부여하여 다시 생성요청해 보았지만 똑같은 오류 발생
    - 웹훅 이후 에러 메세지를 보기 위해 웹훅 삭제
    - 웹훅 삭제 후 에러 메세지 -> yaml spec에 predictor 필드가 들어가지 않는단 내용
        
        `io.fabric8.kubernetes.client.KubernetesClientException: Failure executing: POST at: https://192.168.49.2:8443/apis/serving.kserve.io/v1beta1/namespaces/gaia-ax-mvp/inferenceservices. Message: InferenceService.serving.kserve.io "serving-cd142051-aa76-49" is invalid: spec.predictor: Required value. Received status: Status(apiVersion=v1, code=422, details=StatusDetails(causes=StatusCause(field=spec.predictor, ...  additionalProperties={}).`
        
- 자바에서 yaml 파일을 작성할 때 들어가는 변수명을 "predictorSpec"에서 "predictor"로 변경
객체 필드 변수를 직접 yaml 파일 필드로 인식.
