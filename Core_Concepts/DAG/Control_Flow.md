# Control Flow
기본적으로 DAG는 모든 의존하는 태스크(Task)들이 성공해야만 특정 태스크를 실행합니다. 하지만 이를 수정하는 몇 가지 방법이 있습니다:  

- **Branching (분기 처리)** - 조건에 따라 다음 실행할 태스크를 선택  
- **Trigger Rules (트리거 규칙)** - 특정 태스크를 실행할 조건 설정  
- **Setup and Teardown (설정 및 정리)** - 설정 및 정리 관계 정의  
- **Latest Only (최신 실행만 수행)** - 현재 실행 중인 DAG에서만 실행하는 특별한 분기 형태  
- **Depends On Past (이전 실행 의존성)** - 이전 실행의 동일 태스크 결과에 따라 실행 여부 결정 
---
## **요약**
- `Branching`: 특정 조건에 따라 실행할 태스크를 선택 (`@task.branch` 사용).
- `Latest Only`: 최신 DAG 실행에서만 태스크 실행 (`LatestOnlyOperator` 사용).
- `Depends On Past`: 이전 DAG 실행의 동일 태스크 성공 여부를 기준으로 실행 (`depends_on_past=True` 설정). 

---

## **Branching (분기 처리)**  
Branching을 사용하면 DAG가 모든 후속 태스크를 실행하는 대신 특정 경로만 선택하여 실행할 수 있습니다. 이를 위해 `@task.branch` 데코레이터를 사용합니다.  

### **@task.branch 데코레이터**  
`@task.branch` 데코레이터는 `@task`와 비슷하지만, 특정 태스크의 ID(또는 ID 리스트)를 반환해야 합니다. 반환된 태스크만 실행되며, 나머지는 모두 건너뛰어집니다.  
- `None`을 반환하면 모든 후속 태스크가 건너뛰어집니다.  
- 반환하는 태스크 ID는 `@task.branch` 데코레이터가 적용된 태스크의 바로 다음 태스크여야 합니다.  

> **참고:**  
> 브랜칭 태스크와 선택된 태스크의 공통 후속 태스크가 있을 경우, 해당 태스크는 건너뛰지 않고 실행됩니다.

#### **XCom을 이용한 동적 브랜칭 예제**
```python
@task.branch(task_id="branch_task")
def branch_func(ti=None):
    xcom_value = int(ti.xcom_pull(task_ids="start_task"))
    if xcom_value >= 5:
        return "continue_task"
    elif xcom_value >= 3:
        return "stop_task"
    else:
        return None

start_op = BashOperator(
    task_id="start_task",
    bash_command="echo 5",
    do_xcom_push=True,
    dag=dag,
)

branch_op = branch_func()

continue_op = EmptyOperator(task_id="continue_task", dag=dag)
stop_op = EmptyOperator(task_id="stop_task", dag=dag)

start_op >> branch_op >> [continue_op, stop_op]
```
위 예제에서는 `start_task`의 실행 결과를 확인하여 `continue_task` 또는 `stop_task`를 선택적으로 실행합니다.

#### **BaseBranchOperator 상속을 통한 커스텀 분기 처리**  
직접 브랜칭 연산자를 만들고 싶다면 `BaseBranchOperator`를 상속받아 `choose_branch` 메서드를 구현하면 됩니다.
```python
class MyBranchOperator(BaseBranchOperator):
    def choose_branch(self, context):
        """
        매월 1일에는 추가 작업 실행
        """
        if context['data_interval_start'].day == 1:
            return ['daily_task_id', 'monthly_task_id']
        elif context['data_interval_start'].day == 2:
            return 'daily_task_id'
        else:
            return None
```

> **주의:**  
> `BranchPythonOperator`를 직접 사용하기보다는 `@task.branch` 데코레이터를 사용하는 것이 권장됩니다.

---

## **Latest Only (최신 실행만 수행)**  
Airflow DAG 실행은 일반적으로 현재 날짜가 아닌 과거 날짜를 기준으로 실행될 수 있습니다. 예를 들어, 지난 한 달간의 데이터를 백필(Backfill)하기 위해 여러 DAG 실행을 수행할 수 있습니다.  

하지만 특정 태스크는 최신 실행에서만 실행되도록 해야 할 때가 있습니다. 이때 `LatestOnlyOperator`를 사용하면, "최신 DAG 실행"이 아닌 경우 후속 태스크들을 건너뛸 수 있습니다.  

### **예제: LatestOnlyOperator 적용 DAG**
```python
import datetime
import pendulum
from airflow.models.dag import DAG
from airflow.operators.empty import EmptyOperator
from airflow.operators.latest_only import LatestOnlyOperator
from airflow.utils.trigger_rule import TriggerRule

with DAG(
    dag_id="latest_only_with_trigger",
    schedule=datetime.timedelta(hours=4),
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example3"],
) as dag:
    latest_only = LatestOnlyOperator(task_id="latest_only")
    task1 = EmptyOperator(task_id="task1")
    task2 = EmptyOperator(task_id="task2")
    task3 = EmptyOperator(task_id="task3")
    task4 = EmptyOperator(task_id="task4", trigger_rule=TriggerRule.ALL_DONE)

    latest_only >> task1 >> [task3, task4]
    task2 >> [task3, task4]
```

#### **DAG 실행 흐름**
- `task1` → `latest_only` 태스크의 영향을 받으며, 최신 실행이 아닌 경우 건너뛴다.
- `task2` → `latest_only`와 무관하여 모든 실행에서 실행된다.
- `task3` → 기본 트리거 규칙이 `all_success`이므로, `task1`이 건너뛰면 `task3`도 건너뛴다.
- `task4` → 트리거 규칙이 `all_done`이므로 `task1`이 건너뛰더라도 실행된다.

---

### **Depends On Past (이전 실행 의존성)**  
특정 태스크가 실행되려면 이전 DAG 실행에서 동일한 태스크가 성공해야 하는 경우가 있습니다. 이를 위해 `depends_on_past=True` 옵션을 사용할 수 있습니다.

```python
task = PythonOperator(
    task_id="dependent_task",
    python_callable=my_function,
    depends_on_past=True,
    dag=dag,
)
```

#### **주의 사항**
- DAG의 첫 번째 실행에서는 과거 실행이 없기 때문에 해당 태스크는 실행됩니다.  
- DAG이 여러 번 실행될 경우, 이전 실행에서 성공해야 다음 실행에서도 태스크가 수행됩니다.

---

