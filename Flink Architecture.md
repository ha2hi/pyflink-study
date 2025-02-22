### Flink 아키텍처
Flink는 리소스 관리 매니저인 Hadoop Yarn 혹은 K8S을 통해 실행할 수 있습니다.  
또는 standalone방식 혹은 library를 통해 1개의 노드에서 실행할 수 있습니다.  
  
### 실행 구조
Flink는 각각 1개 이상의 JobManager와 TaskManager로 구성됩니다.  
![processes](./img/processes.svg)  
  
### JobManager
JobManager는 작업을 관리하고 체크포인트를 조정하고, 실패 시 복구하는 등의 작업을 합니다.  
JobManager는 3가지 구성요소로 구성됩니다.  
- ResourceManager
ResourceManager에서는 클러스터에서 리소스 할당을 해제 및 프로비저닝을 담당합니다. 즉, 슬롯을 관리합니다.  
Yarn, K8S 및 Standalone과 같은 다양한 환경에서 ResourcManager를 실행할 수 있습니다.  
Standalone 방식에서는 사용 가능한 TaskManager의 슬롯만 사용 가능하며 TaskManager를 추가적으로 생성할 수 없습니다.  
- Dispatcher
Flink 애플리케이션을 제출하기 위한 REST 인터페이스를 제공하고, 각작업에 대해 JobMaster를 시작합니다.  
또 WebUI를 실행하여 작업 실행에 대한 정보를 제공합니다.  
- JobMaster
JobGraph의 실행을 관리하며 실행한 작업은 각각 JobMaster를 가집니다.  

### JobManager Mode
Spark에서 작업을 제출 할 때 Local, Client, Cluster 모드가 있는 것 처럼 Flink도 마찬가지로 작업을 배포할 때 모드를 지정할 수 있습니다.  
![deployment_modes](./img/deployment_modes.svg)  
- Application Mode
각 작업마다 별도의 클러스터가 생성됩니다. 여러 개의 작업을 실행할 수 있지만 작업마다 클러스터가 구성됩니다.  
리소스 격리가 가능하나 작업을 배포할 때 마다 Cluster를 구성해야 되기 때문에 시간이 오래걸립니다.  
  
- Session Mode
실행중인 Flink 클러스터를 공유하여 여러 작업을 실행하는 방식입니다.  
모든 작업들이 동일한 클러스터의 자원을 공유하며 실행됩니다. 따라서 Job을 실행할 때 마다 클러스터를 새로 생성할 필요가 없습니다.  
동일한 클러스터에서 실행되기 때문에 여러 Job이 한정된 자원 안에서 경쟁한다는 단점이 있습니다.  
  
- Per-Job Mode(deprecated)
Flink 1.15버전 이후에는 더이상 지원하지 않는 배포 모드입니다.  

### TaskManager
TaskManager는 워커 노드로 작업을 실행하는 노드로 작업을 실행하고 데이터 스트림 버퍼를 교환합니다.  
모든 작업은 최소 1개이상의 TaskManager가 있어야 하며 Slot이라는 가장 작은 단위로 나눠 작업을 실행합니다.  
Slot은 동시 처리할 수 있는 작업 수입니다.  

### Slot and Resources
- TaskManager는 1개의 JVM 프로세스인데 Slot이라는 최소 단위로 나눠 작업을 동시에 처리합니다.  
- Slot은 Threads이기 때문에 동일한 작업인 경우 데이터를 서로 주고 받을 수 있습니다.  
- TaskManger의 메모리는 Slot별로 분배됩니다. 즉 3개의 Slot이 있다면 각 슬롯은 1/3의 메모리가 분배됩니다. 메모리와 달리 CPU는 공유하여 사용합니다.  
- TaskManager당 1개의 Slot으로 실행하는 경우 별도의 JVM에서 실행되기 때문에 격리성이 높아짐
- TaskManager당 2개 이상의 Slot으로 실행하는 경우 하나의 JVM에서 여러 Task가 실행되고 자원을 서로 공유할 수 있습니다.  
![slot_sharing](./img/slot_sharing.svg)  


