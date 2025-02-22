### 개요
S3에 저장된 Pyflink 소스 파일을 읽은 후 Flink Kubernertes Operator에 작업 제출  
  
작업 순서
- Docker Image 생성
- 서비스 계정에 IAM 역할 할당
- YAML 파일 수정
- 작업 실행 및 확인

### 1. Docker Image 생성
Flink에서 특정 기능을 확장하거나 외부 시스템과 통합할 때 플러그인을 사용합니다.  
- File System Plugin : HDFS, S3, GCS등과 같은 외부 스토리지 지원
- 사용자 정의 함수(UDF) 추가
S3에 파일을 읽기 위해서는 plugins 폴더에 다음과 같이 `flink-s3-fs-hadoop.jar`을 저장해야 합니다.
```
mkdir ./plugins/s3-fs-hadoop
cp ./opt/flink-s3-fs-hadoop-1.20.0.jar ./plugins/s3-fs-hadoop/
```  
  
따라서 저는 Dockerfile 하단에 다음 코드를 추가하였습니다.
```
RUN mkdir /opt/flink/plugins/s3-fs-hadoop && \
    cp /opt/flink/opt/flink-s3-fs-hadoop-1.20.0.jar /opt/flink/plugins/s3-fs-hadoop
```  
  
그리고 docker image를 생성하고 push합니다.
```
docker image build . -t hiha2/flink-s3:latest

docker push hiha2/flink-s3:latest
```
  
### 2. 서비스 계정에 IAM 할당
Flink에서 S3에 접근하기 위해서는 인증이 필요합니다. 이때 방법은 2가지가 있습니다.  
1. Acceess Key
2. IAM Role
방법 1(Acceess Key)은 k8s secret을 통해 다음과 같이 flinkConfiguration을 추가하여 인증할 수 있습니다.
```
s3.access-key: your-access-key
s3.secret-key: your-secret-key
```
하지만 키관리는 번거롭고 추적이 힘들기 때문에 Serviceaccount에 AWS IAM Role(방법2)를 통해 인증을하려고 합니다.  
  
flink kubernetes operator을 설치하면 Default로 `flink` Serviceaccount가 생성되고 해당 계정으로 Application을 제출합니다.  
따라서 저는 `flink` 계정에 S3 접근 권한을 주려고 합니다.  
- IAM 정책(Policy) 생성
`<YOUR_BUCKET_NAME>`에 버킷 명을 입력합니다.  
```
cat >my-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>"
        }
    ]
}
EOF
```  
  
Policy 이름은 `my-policy`으로 생성하겠습니다.  
```
aws iam create-policy --policy-name my-policy --policy-document file://my-policy.json
```  
  
- 역할(Role) 생성 및 정책(Policy) 연결
`my-role`이라는 역할을 생성하고 방금 생성한 `my-policy`을 연결합니다.  
<YOUR_CLUSTER_NAME>에 EKS 클러스터 명을 입력하고, <YOUT_ACCOUNT_ID>에 AWS 계정 ID를 입력합니다.  

```
eksctl create iamserviceaccount --name flink --namespace default --cluster <YOUR_CLUSTER_NAME> --role-name my-role \
    --attach-policy-arn arn:aws:iam::<YOUT_ACCOUNT_ID>:policy/my-policy --approve
```  
  
- 신뢰 관계 추가
신뢰 관계 생성을 위한 환경 변수를 추가합니다.  
```
account_id=$(aws sts get-caller-identity --query "Account" --output text)

oidc_provider=$(aws eks describe-cluster --name my-cluster --region $AWS_REGION --query "cluster.identity.oidc.issuer" --output text | sed -e "s/^https:\/\///")

export namespace=default
export service_account=flink
```  
  
신뢰 관계를 추가합니다.
```
cat >trust-relationship.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$account_id:oidc-provider/$oidc_provider"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$oidc_provider:aud": "sts.amazonaws.com",
          "$oidc_provider:sub": "system:serviceaccount:$namespace:$service_account"
        }
      }
    }
  ]
}
EOF
```  
```
aws iam create-role --role-name my-role --assume-role-policy-document file://trust-relationship.json --description "my-role-description"
```
```
aws iam attach-role-policy --role-name my-role --policy-arn=arn:aws:iam::$account_id:policy/my-policy
```  
  
- Serviceaccount에 Role 부여
Serviceaccount에 Role을 부여하기 위해서 Serviceacccount의 Annotations을 추가해야 합니다.  
```
kubectl annotate serviceaccount -n default flink eks.amazonaws.com/role-arn=arn:aws:iam::$account_id:role/my-role
```  
  
다음 과정이 정상으로 진행 되었다면 다음과 같이 확인이 되어야 합니다.  
- 역할 신뢰 관계(정책) 확인 가능
```
aws iam get-role --role-name my-role --query Role.AssumeRolePolicyDocument
```  
  
- 역할(Role)에 정상적으로 정책(Policy)가 연결되었는지 확인
```
aws iam list-attached-role-policies --role-name my-role --query AttachedPolicies[].PolicyArn --output text
```  
  
- `flink` Serviceaccount에 정상적으로 Annotations이 추가되었는지 확인
```
kubectl describe serviceaccount flink -n default
```  
  
### 3. YAML 파일 수정
S3에 있는 소스 코드를 저장하기 위해 Flink Main 컨테이너를 생성하고 볼륨을 마운트합니다.  
```
  serviceAccount: flink
  podTemplate:
    spec:
      containers:
        - name: flink-main-container
          volumeMounts:
            - mountPath: /opt/flink/usrlib
              name: flink-logs
      volumes:
        - name: flink-logs
          emptyDir: { }
```  
  
그리고 S3에 저장된 소스 코드를 가져오기 위해 `jobManager`에 init Container를 추가하여 복사하는 스크립트를 개발합니다.  
```
  jobManager:
    resource:
      memory: "2048m"
      cpu: 1
    podTemplate:
      spec:
        initContainers:
          - name: fetch-python-script
            image: amazon/aws-cli
            command: ["aws", "s3", "cp", "s3://flink-s3-test/test/python_demo_s3.py", "/opt/flink/usrlib/python_demo_s3.py"]
            volumeMounts:
              - name: flink-logs
                mountPath: /opt/flink/usrlib
```  
  
정리를 해보면 Main 컨테이너에서 볼륨을 마운트했고, initcontainer를 통해 Application이 실행되기 전에 S3에 저장된 파일을 마운트한 볼륨에 저장합니다.  
  

### 4. 작업 실행 및 확인
작업을 제출하여 Application을 실행합니다.
- 작업 제출  
```
kubectl apply -f python-example-s3.py
```  
- 작업 확인
```
kubectl get pods 

kubectl logs deploy/python-example-s3
```  
  
Task Manger가 생성되고 로그를 통해 output이 정상적으로 나오면 성공한 것 입니다.  
  
### Ref:
- https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/filesystems/plugins/
- https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/associate-service-account-role.html
