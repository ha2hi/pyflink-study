### Flink Quick Start
Flink Kubernetes Operator를 사용하여 Custom Python 코드를 Flink Operator에 작업 제출하는 방법이 담겼습니다.
- Flink Operator v.10
  
### 1. 인증 관리 설치
cert-manager는 Webhook 구성 요소를 추가할 수 있도록 돕습니다.  
```
kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```  
  
### 2. Flink Kubernetes Operator 설치
Flink Operator Repo를 추가합니다. 이 때 `flink-kubernetes-operator-1.10.0`미만인 버전으로 설치 시 v1_20을 사용할 수 없습니다.  
```
helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.10.0/

helm install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator
```  
  
### 3. Docker image build & push
```
docker build . -t hiha2/flink-python-example:1.20

docker push -t hiha2/flink-python-example:1.20
```

### 4. 실행 및 확인
- 작업 실행
```
kubectl apply -f python-example.yaml
```
- 작업 확인
```
kubectl logs -f deploy/python-example

kubectl get flinkdeployment
```


### Ref:
- https://github.com/apache/flink-kubernetes-operator/tree/main/examples/flink-python-example 
