### State란?
Flink는 State 저장 작업을 지원하는 스트림 프로세싱 프레임워크입니다.  
여기서 State는 이전에 들어온 데이터로 상태 정보 데이터를 의미합니다.  
  
Window를 통해 2개 이상의 데이터를 처리할 수 있으나 그 이상의 데이터 값을 처리해야되는 경우가 있습니다.  
예를 들어, 사기거래탐지를 개발한다고 했을 때 이전 데이터와 현재 데이터를 비교해야될 것 입니다.
이때 이전 데이터를 State로 관리하여 현재 들어 온 데이터와 비교하여 분석할 수 있습니다.  
  
이렇듯 State는 스트림 프로세싱에서 중요한 기능이기 때문에 자세히 알아보도록 하겠습니다.  
  
### State Backend Options
Flink에서 2개의 State Backend를 지원합니다.
현재 내 컴퓨팅 자원 사용 가능량, State 크기, 빠른 State Read/Write 상황에 맞춰 어떤 Backend를 사용할지 고민해야 됩니다.  
어떤 State Backend 가 있고 언제 사용하면 좋은지 알아보도록 하겠습니다.
          
1. HashMapStateBackend
- State를 JVM(Java Heap)의 객체로 보유합니다.
- 큰 State, Windows, Key/Value State를 저장할 수 있습니다.
- HashMapStateBackend 혹은 Statless 작업을 하는 경우 Manged Memory(taskmanager.memory.managed.size)를 0으로 설정하여 최대 Heap 메모리를 사용합니다.

2. EmbeddedRocksDBStateBackend
- State를 RocksDB에 저장합니다.  
- HashMapStateBackend과 달리 EmbeddedRocksDBStateBackend는 State를 저장할 때 직렬화(serialized)하여 저장하고 읽을 때는 역직렬화(de-serialization)작업을 하여 상대적으로 느립니다.
- 엄청 큰 State, Windows, Key/Value State를 저장할 수 있습니다.
  
큰 State를 사용하는 경우 EmbeddedRocksDBStateBackend를 사용하는 것이 좋으나 메모리 설정 작업과 같은 튜닝 작업이 필요하고 디스크에 State 정보를 저장할 때 직렬화/역직렬화 작업을 하므로 속도가 상대적으로 느립니다.  
상대적으로 작은 State를 사용하는 경우 HashMapStateBackend를 사용하는 것이 관리 혹은 속도 측면에서 좋습니다.  
  
상황에 맞게 알맞는 State Backend를 사용해야 됩니다.  

### Keyed State
Keyed State는 Key별로 병렬로 실행할 수 있는 State 방식입니다.  
- 종류
  - ValueState<T> : 단일 값 저장
  - ListState<T> : Key별 값 목록 저장
  - MapState<K, V> : Key-Value 맵 저장
  - ReducingState<T> : 값 집계
  - AggregatingState<IN, OUT> : ReducingState 확장  

이제 Keyed State를 코드로 어떻게 사용하는 알아보도록 하겠습니다.  
- key_by
`key_by`를 통해 keyed State를 사용할 수 있습니다.  
```
words = # type: DataStream[Row]
keyed = words.key_by(lambda row: row[0])
```

- Descriptor
State를 사용하기 위해서 Descriptor를 생성해야 됩니다.  
Descriptor에서는 어떤 State를 사용할지, TTL 설정등에 관한 State 정보에 대한 내용이 입력됩니다.  
```
def open(self, runtime_context: RuntimeContext):
        descriptor = ValueStateDescriptor(
            "average",  # the state name
            Types.PICKLED_BYTE_ARRAY()  # type information
        )
        self.sum = runtime_context.get_state(descriptor)
```

- Function
Flink의 내장 Function(ex. KeyedProcessFunction, FlatMapFunction)을 상속 받아 State작업 처리를 위한 Function을 개발합니다.  
아래 코드는 FlatMapFunction을 입력 받아 Custom한 `CountWindowAverage`라는 Function을 개발한 코드입니다.  
```
class CountWindowAverage(FlatMapFunction):

    def __init__(self):
        self.sum = None

    def open(self, runtime_context: RuntimeContext):
        descriptor = ValueStateDescriptor(
            "average",  # the state name
            Types.PICKLED_BYTE_ARRAY()  # type information
        )
        self.sum = runtime_context.get_state(descriptor)

    def flat_map(self, value):
        # access the state value
        current_sum = self.sum.value()
        if current_sum is None:
            current_sum = (0, 0)

        # update the count
        current_sum = (current_sum[0] + 1, current_sum[1] + value[1])

        # update the state
        self.sum.update(current_sum)

        # if the count reaches 2, emit the average and clear the state
        if current_sum[0] >= 2:
            self.sum.clear()
            yield value[0], int(current_sum[1] / current_sum[0])
```  
  
Flnik에서 제공하는 함수는 아래 링크에서 확인할 수 있습니다.  
https://nightlies.apache.org/flink/flink-docs-release-1.17/api/python/reference/pyflink.datastream/functions.html
  

이제 제가 작성한 코드로 State를 사용한 작업을 보도록 하겠습니다.  
아래 코드는 입력 데이터로 (User명, 결제 금액) 받아와서 이전 결제 금액과 현재 결재 금액이 10,000$ 이상 차이가 나면 Alert을 출력하도록 작성한 코드입니다.  

```
from pyflink.common import Types
from pyflink.datastream import StreamExecutionEnvironment, RuntimeContext
from pyflink.datastream.functions import KeyedProcessFunction
from pyflink.datastream.state import ValueStateDescriptor

class MyKeyedProcessFunction(KeyedProcessFunction):
    def __init__(self):
        self.state = None

    def open(self, runtime_context: RuntimeContext):
        state_descriptor = ValueStateDescriptor("YOUR_STATE_NAME", Types.LONG())
        self.state = runtime_context.get_state(state_descriptor)

    def process_element(self, value, ctx):
        id, now_money = value
        prev_money = self.state.value()

        if prev_money is not None and now_money - prev_money >= 10000:
            print(f"Fraud Alert User : {id}, Prev : {prev_money}, Now : {now_money}, Diff :", now_money-prev_money)
        self.state.update(now_money)

env = StreamExecutionEnvironment.get_execution_environment()
data = [
    ("user1", 5000),
    ("user2", 2000),
    ("user1", 7000),
    ("user2", 8000),
    ("user1", 110000),
    ("user2", 3000),
    ("user2", 120000)
]

ds = env.from_collection(data)
ds.key_by(lambda row: row[0]).process(MyKeyedProcessFunction()).print()

env.execute()
```
`MyKeyedProcessFunction`은 `KeyedProcessFunction`을 상속 받아 `open` 함수에서 Descriptor를 정의하였고 `process_element` 함수에서 실질적으로 처리 되는 내용을 작성했습니다.  
  
KeyedProcessFunction의 자세한 코드는 아래 링크에서 확인할 수 있습니다.  
https://nightlies.apache.org/flink/flink-docs-release-1.17/api/python/_modules/pyflink/datastream/functions.html#KeyedProcessFunction

### State TTL(Time-To-Live)
State를 계속 누적하여 저장하면 좋겠지만 그렇다면 저장공간과 속도에 대한 문제가 발생할 수 있습니다.  
이 때문에 TTL을 설정하여 조건을 통해 State를 초기화하는 작업이 필요합니다.  
  
TTL을 사용하기 위해서는 `StateTtlConfig` 객체를 빌드해야 됩니다.  
```
from pyflink.common.time import Time
from pyflink.common.typeinfo import Types
from pyflink.datastream.state import ValueStateDescriptor, StateTtlConfig

ttl_config = StateTtlConfig \
  .new_builder(Duration.ofSeconds(1)) \
  .set_update_type(StateTtlConfig.UpdateType.OnCreateAndWrite) \
  .set_state_visibility(StateTtlConfig.StateVisibility.NeverReturnExpired) \
  .build()

state_descriptor = ValueStateDescriptor("text state", Types.STRING())
state_descriptor.enable_time_to_live(ttl_config)
```
- new_builder(Duration.ofSeconds(1))
  - TTL을 1초로 설정합니다.
  - 1초가 지나면 상태 데이터가 자동으로 삭제 됩니다.
- set_update_type
  - State TTL의 갱신 정책입니다.  
  - TTL 타이머가 언제 리셋되는지 결정
  - OnCreateAndWrite
    - State가 생성될 때와 값이 업데이트될 때 마다 TTL 타이머 리셋
    - 예를 들어 State 5분간 활동이 없는 경우 세션 종료
  - OnReadAndWrite
    - State를 읽거나 쓸 대 마다 타이머 리셋
    - 자주 조회되는 설정 정보를 계속 유지해야 할 때 
  - 기본 값은 OnCreateAndWrite입니다.  
- set_state_visibility
  - 상태 반환 여부를 설정합니다.
  - StateTtlConfig.StateVisibility.NeverReturnExpired
    - 만료된 State를 반환하지 않습니다.
    - TTL 기간을 초과하면 논리적으로 삭제 처리
    - state.value()시 null 반환
  - StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp
    - 물리적으로 State가 삭제 되기 전까지는 만료된 상태를 반환할 수 있습니다.  
  
### 문제
1. User 별로 누적 결제 금액을 출력 할 수 있는 코드를 작성해보세요