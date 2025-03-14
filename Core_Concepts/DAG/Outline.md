# Apache Airflow DAG 개요
## 요약
* Apache Airflow의 DAG는 데이터 파이프라인을 정의하는 핵심 개념입니다.
* DAG을 통해 작업을 구성하고, 실행 순서 및 주기를 설정하며, DAG 간 의존성을 정의할 수 있습니다. 
* DAG을 효과적으로 설계하면 복잡한 데이터 워크플로우를 효율적으로 자동화할 수 있습니다.

## DAG(방향 비순환 그래프)란?

DAG(Directed Acyclic Graph)는 Airflow의 핵심 개념으로, 여러 개의 **Task(작업)**을 하나로 묶고, 실행 순서 및 의존성을 정의하는 구조입니다.

### DAG의 기본 예제

DAG는 다음과 같은 작업(Task)을 정의할 수 있습니다: `A`, `B`, `C`, `D`.  
이 DAG는 실행 순서를 결정하고, 어떤 작업이 다른 작업에 의존하는지 지정합니다. 또한 DAG 실행 빈도를 설정할 수도 있습니다.

예:
- `"내일부터 5분마다 실행"`
- `"2020년 1월 1일부터 매일 실행"`

DAG 자체는 개별 Task에서 수행되는 작업의 내용을 알지 못합니다. DAG의 역할은 Task의 실행 순서, 재시도 횟수, 타임아웃 등의 실행 관리에 집중됩니다.

---

## DAG 선언 방법

DAG를 선언하는 방법은 다음 세 가지가 있습니다.

### 1. `with` 문을 사용한 선언 (권장)
`with` 문을 사용하면 내부에 정의된 모든 Task가 해당 DAG에 자동으로 추가됩니다.

```python
import datetime
from airflow import DAG
from airflow.operators.empty import EmptyOperator

with DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
):
    EmptyOperator(task_id="task")
```

### 2. 표준 생성자 사용
DAG 객체를 생성한 후 Task에 명시적으로 연결하는 방식입니다.

```python
import datetime
from airflow import DAG
from airflow.operators.empty import EmptyOperator

my_dag = DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
)
EmptyOperator(task_id="task", dag=my_dag)
```

### 3. `@dag` 데코레이터 사용
함수를 DAG 생성기로 변환할 수 있습니다.

```python
import datetime
from airflow.decorators import dag
from airflow.operators.empty import EmptyOperator

@dag(start_date=datetime.datetime(2021, 1, 1), schedule="@daily")
def generate_dag():
    EmptyOperator(task_id="task")

generate_dag()
```

---

## Task 의존성 설정

DAG에서는 Task 간의 관계를 정의해야 합니다. Task는 **업스트림(Upstream) Task**(이전 작업)과 **다운스트림(Downstream) Task**(이후 작업)을 가질 수 있습니다.

### 1. `>>` 및 `<<` 연산자를 사용한 방법 (권장)
```python
first_task >> [second_task, third_task]
third_task << fourth_task
```

### 2. `set_upstream()` 및 `set_downstream()` 메서드 사용
```python
first_task.set_downstream([second_task, third_task])
third_task.set_upstream(fourth_task)
```

### 3. 복잡한 의존성 선언을 위한 `cross_downstream()`
여러 Task 간의 종속 관계를 쉽게 정의할 수 있습니다.

```python
from airflow.models.baseoperator import cross_downstream

cross_downstream([op1, op2], [op3, op4])
```

### 4. `chain()`을 사용한 체인 의존성
```python
from airflow.models.baseoperator import chain

chain(op1, op2, op3, op4)
```

---

## DAG 실행 방식

DAG는 두 가지 방식으로 실행됩니다.

1. **수동 실행**: UI 또는 API를 통해 직접 실행
2. **스케줄링 실행**: 사전에 정의한 일정에 따라 실행

DAG 실행 주기는 `schedule` 매개변수로 정의됩니다.

```python
with DAG("my_daily_dag", schedule="@daily"):
    ...
```

유효한 `schedule` 값:
```python
with DAG("my_daily_dag", schedule="0 0 * * *"):  # 매일 자정 실행
    ...

with DAG("my_one_time_dag", schedule="@once"):  # 한 번만 실행
    ...

with DAG("my_continuous_dag", schedule="@continuous"):  # 지속 실행
    ...
```

각 DAG 실행은 **DAG Run**이라는 인스턴스로 관리됩니다. DAG Run은 서로 독립적으로 실행될 수 있으며, 특정 기간 동안의 데이터를 처리할 수 있도록 **데이터 인터벌(data interval)**을 정의합니다.

---

## Task 실행 규칙 (Trigger Rules)

기본적으로 Airflow는 모든 업스트림 작업이 성공해야 해당 Task가 실행됩니다. 하지만 특정 조건에서 Task 실행을 조정할 수 있습니다.

| Trigger Rule | 설명 |
|-------------|------|
| `all_success` | 모든 업스트림 Task가 성공해야 실행 (기본값) |
| `all_failed` | 모든 업스트림 Task가 실패해야 실행 |
| `all_done` | 업스트림 Task가 모두 완료되면 실행 (성공 또는 실패 포함) |
| `one_failed` | 하나 이상의 업스트림 Task가 실패하면 실행 |
| `one_success` | 하나 이상의 업스트림 Task가 성공하면 실행 |
| `none_failed_min_one_success` | 모든 업스트림 Task가 실패하지 않고 최소 하나 이상 성공해야 실행 |
| `always` | 모든 업스트림 Task 상태와 관계없이 항상 실행 |

```python
task1 = EmptyOperator(task_id="task1", trigger_rule="one_failed")
```

---


