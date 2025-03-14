# **요약**
- DAG의 시각화를 위해 **Airflow UI의 Graph 뷰** 또는 **`airflow dags show` 명령어**를 사용할 수 있음.
- **TaskGroup**을 사용하여 DAG을 그룹화하고 UI에서 더 쉽게 볼 수 있음.
- **Edge Labels**을 활용하면 DAG의 분기 조건을 명확하게 표시할 수 있음.
---
# DAG Visualization
DAG(Directed Acyclic Graph)의 시각적 표현을 확인하려면 두 가지 방법이 있습니다:

1. Airflow UI를 열고, 해당 DAG으로 이동한 후 **"Graph"** 뷰를 선택하는 방법  
2. `airflow dags show` 명령어를 실행하여 DAG을 이미지 파일로 렌더링하는 방법  

일반적으로 **Graph 뷰**를 사용하는 것이 좋습니다. 이 뷰에서는 특정 DAG 실행(DAG Run) 내의 모든 태스크 인스턴스(Task Instances) 상태도 확인할 수 있습니다.

DAG가 점점 복잡해질 것이므로, Airflow에서는 DAG 뷰를 보다 쉽게 이해할 수 있도록 몇 가지 기능을 제공합니다.

---

## **TaskGroups**
TaskGroup은 **Graph 뷰에서 태스크를 계층적인 그룹으로 구성하는 기능**입니다. 반복되는 패턴을 만들거나 **시각적 복잡성을 줄이는 데** 유용합니다.

- **SubDAG과 다름**: TaskGroup은 **UI에서만 그룹화하는 개념**입니다. TaskGroup 내의 태스크들은 원래 DAG에서 실행되며, DAG의 설정 및 리소스 풀(Pool) 구성을 그대로 따릅니다.
- **의존성(Dependency) 설정 가능**: TaskGroup 내부의 모든 태스크에 대해 `>>` 및 `<<` 연산자를 사용하여 의존 관계를 설정할 수 있습니다.

예제 코드:
```python
from airflow.decorators import task_group
from airflow.operators.empty import EmptyOperator

@task_group()
def group1():
    task1 = EmptyOperator(task_id="task1")
    task2 = EmptyOperator(task_id="task2")

task3 = EmptyOperator(task_id="task3")

group1() >> task3
```

### **TaskGroup의 default_args 사용**
TaskGroup은 DAG과 마찬가지로 `default_args`를 설정할 수 있으며, DAG의 `default_args`를 덮어씁니다.

```python
import datetime
from airflow import DAG
from airflow.decorators import task_group
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator

with DAG(
    dag_id="dag1",
    start_date=datetime.datetime(2016, 1, 1),
    schedule="@daily",
    default_args={"retries": 1},  # DAG 기본 설정
):

    @task_group(default_args={"retries": 3})  # TaskGroup에 별도 설정 적용
    def group1():
        """이 docstring이 UI에서 TaskGroup의 툴팁으로 표시됩니다."""
        task1 = EmptyOperator(task_id="task1")
        task2 = BashOperator(task_id="task2", bash_command="echo Hello World!", retries=2)
        print(task1.retries)  # 3 (TaskGroup의 설정 적용)
        print(task2.retries)  # 2 (개별 태스크에서 재설정)
```

### **TaskGroup의 ID 프리픽스**
- 기본적으로, TaskGroup 내의 태스크/서브 그룹은 **부모 TaskGroup의 ID를 접두사(prefix)로 추가**하여 생성됩니다.
- 이를 방지하려면 `prefix_group_id=False` 옵션을 설정해야 합니다. 단, 이 경우 모든 태스크와 그룹의 ID가 **고유해야** 합니다.

---

## **Edge Labels (엣지 라벨)**
태스크를 그룹화하는 것 외에도 **Graph 뷰에서 태스크 간의 의존 관계(Edges)에 라벨을 추가**할 수 있습니다.  
이 기능은 DAG의 **분기(branching) 로직을 이해하기 쉽게** 만들기 위해 유용합니다.

### **엣지 라벨 추가 방법**
1. `>>` 또는 `<<` 연산자를 사용하여 직접 추가:
```python
from airflow.utils.edgemodifier import Label

my_task >> Label("When empty") >> other_task
```

2. `set_upstream` 또는 `set_downstream` 메서드에서 `Label` 객체 전달:
```python
from airflow.utils.edgemodifier import Label

my_task.set_downstream(other_task, Label("When empty"))
```

### **엣지 라벨 예제 DAG**
```python
import pendulum
from airflow import DAG
from airflow.operators.empty import EmptyOperator
from airflow.utils.edgemodifier import Label

with DAG(
    "example_branch_labels",
    schedule="@daily",
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
) as dag:
    ingest = EmptyOperator(task_id="ingest")
    analyse = EmptyOperator(task_id="analyze")
    check = EmptyOperator(task_id="check_integrity")
    describe = EmptyOperator(task_id="describe_integrity")
    error = EmptyOperator(task_id="email_error")
    save = EmptyOperator(task_id="save")
    report = EmptyOperator(task_id="report")

    ingest >> analyse >> check
    check >> Label("No errors") >> save >> report
    check >> Label("Errors found") >> describe >> error >> report
```

위 DAG에서:
- `check` 태스크가 실행된 후 **오류가 없으면** `"No errors"` 라벨이 표시된 후 `save` → `report`로 진행됩니다.
- 반면 **오류가 발생하면** `"Errors found"` 라벨이 표시되며 `describe` → `error` → `report` 경로로 진행됩니다.

