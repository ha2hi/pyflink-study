## 개요
Flink에서 SQL API를 사용하여 쿼리를 사용하여 이벤트 처리를 할 수 있으나 아래와 같은 제한이 있습니다.
- Table API에서 SQL을 사용 시 Java, Scala 또는 Python에 대한 이해가 필요함.
- Java 및 Scala 프로그램을 실행하려면 Maven, Gradle등 빌드 도구를 사용하여 JAR파일 생성 후 Flink에 업로드하고 실행함으로 복잡함.  
  
이러한 불편함으로 해소하기 위해 Flink Client는 Java, Scala, Python과 같은 코드 사용 없이 쿼리를 실행하여 바로 디버깅할 수 있도록 돕습니다.
  
## 실행
Flink Client를 실행하기 위한 2가지 방법이 있다.
- embedded standalone
- SQL Gateway
SQL Client의 기본 모드는 embedded입니다.  
(주의: Cluster가 실행되어 있어야 합니다.)  
  
### 1. embedded standalone
- embedded모드 실행
```
./bin/sql-client.sh embedded
```
- 쿼리 작성
```
SET 'sql-client.execution.result-mode' = 'tableau';
SET 'execution.runtime-mode' = 'batch';

SELECT
  name,
  COUNT(*) AS cnt
FROM
  (VALUES ('Bob'), ('Alice'), ('Greg'), ('Bob')) AS NameTable(name)
GROUP BY name;
```
  
### 2. SQL Gateway
아래와 같은 순서로 진행됩니다.
1. SQL Gateway 실행
2. 세션 생성
3. SQL 실행
4. 쿼리 결과 조회
  
1. SQL Gateway 실행
- SQL Gateway 실행
```
./bin/sql-gateway.sh start -Dsql-gateway.endpoint.rest.address=localhost
```
  
- 확인
실행한 Gateway에 REST Endpoint를 사용할 수 있는지 확인
```
curl http://localhost:8083/v1/info
```
  
2. 세션 생성
사용자 식별을 위한 세션 생성
```
curl --request POST http://localhost:8083/v1/sessions
```  
반환된 세션 값을 재사용해야 됩니다.
```
export sessionHandle="YOUR_SESSION_HANDLE"
```  
  
3. SQL 실행
```
curl --request POST http://localhost:8083/v1/sessions/${sessionHandle}/statements/ --data '{"statement": "SELECT 1"}'
```  
반환 되는 operationHandle를 저장해야 됩니다.
```
export operationHandle="YOUR_OPERATION_HANDLE"
```
  
4. 쿼리 결과 조회
```
curl --request GET http://localhost:8083/v1/sessions/${sessionHandle}/operations/${operationHandle}/result/0
```

## ETC
- SQL Clint Configuration
```
SET 'key' = 'value';
```
  
- SQL Client result modes
총 3가지의 result mode가 있습니다.
1. table
2. changelog
3. tableau
`tableau`가 가시성은 제일 종습니다.  
```
SET 'sql-client.execution.result-mode' = 'tableau';
```
  
- SQL statements
단일 SQL문이 아닌 일련의 SQL문을 사용하기 위해 STATEMENT SET 구문을 지원합니다.
STATEMENT SET구문은 하나 이상의 INSERT INTO문을 포함합니다
```
EXECUTE STATEMENT SET 
BEGIN
  -- one or more INSERT INTO statements
  { INSERT INTO|OVERWRITE <select_statement>; }+
END;
```

- DML 동기 및 비동기 실행
SQL Client는 DML(SELECT, INSERT, UPDATE, DELETE)를 비동기 방식으로 실행합니다.  
따라서 SQL Client는 DML을 동시에 여러 작업을 호출할 수 있습니다.  
  
그러나 동기적으로 실행이 필요한 경우 아래와 같이 사용하면 됩니다.
```
SET 'table.dml-sync' = 'true';
```
  
- Savepoint에서 실행
Flink는 원하는 Savepoint부터 작업을 실행할 수 있습니다.
```
SET 'execution.state-recovery.path' = '/tmp/flink-savepoints/savepoint-cca7bc-bb1e257f0dab';
```  
  
- 모니터링
작업 제출 및 결과를 모니터링할 수 있습니다.
```
SHOW JOBS;
```
  
- 작업 종료
```
STOP JOB '<YOUR_JOB_ID>' WITH SAVEPOINT;
```  

  
## 정리
- Flink에서 SQL을 사용하여 이벤트 처리할 수 있는 SQL Client를 제공한다. 
- SQL Client 실행은 embedded와 sql gateway 방식이 있다.
