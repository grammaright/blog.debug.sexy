---
layout: post
title:  "In-Memory Columnar Storage와 SQL Query Engine: Apache Arrow와 DataFusion의 기술 분석"
date:   2025-04-26 18:00:00 +0900
categories: dbms
---

*이 글은 제가 2025년 4월 26일에 진행한 "Databend 세미나: Apache Arrow와 DataFusion의 기술적인 특징과 데이터베이스 시스템과의 관계" 세미나 내용을 기반으로 작성되었습니다.*

---

이 글에서는 Apache Arrow와 Apache DataFusion의 특징을 살펴보고, 이를 데이터베이스 관리 시스템(DBMS)과 비교하여 논의합니다.

## 데이터베이스 관리 시스템

먼저 데이터베이스 관리 시스템(Database Management System)이 무엇이고, 여기에 무슨 컴포넌트가 있는지 간략하게 짚고 넘어가겠습니다. 데이터베이스 관리 시스템은 데이터베이스 시스템 또는 간단히 데이터베이스라고도 불립니다. 여기서는 DBMS라고 줄여서 부르도록 하겠습니다.

![DBMS architecture](/assets/images/2025-04-26-apache-arrow-and-datafusion/dbms-arch.png){: width="300" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*DBMS 구성도 개요 ([출처](https://www.oreilly.com/library/view/database-internals/9781492040330/ch01.html))* 
{: refdef}

위 그림은 DBMS의 일반적인 아키텍쳐에 대한 이미지입니다. 가장 위의 Transport layer는 클라이언트 프로그램(예: `psql`, `mysql` 커맨드 프로그램 등)이나 다른 DBMS와의 통신을 담당합니다. Transport layer, Query Processor, Execution Engine, Storage Engine을 포함하여 하나의 DBMS라고 혹은 하나의 인스턴스라고 통상 부릅니다.

사용자가 클라이언트 프로그램을 통해 쿼리(query 또는 질의)를 입력하면, 해당 쿼리는 DBMS의 Query Processor로 전달됩니다. 관계형 데이터베이스 관리 시스템(Relational Database Management System, 줄여서 RDBMS)의 경우, 통상 Structured Query Language(SQL)을 통해 쿼리를 표현하고, 사용자는 이를 이용하여 쿼리를 요청합니다. 이는 사람이 읽을 수 있는 형태의 언어(human readable language)이기 때문에, Query Parser는 이 SQL을 컴퓨터가 이해하기 위한 형태로 변환합니다.

**Query Parser**는 SQL string을 입력받아 컴퓨터가 이해할 수 있는 **logical plan**(논리 계획)을 생성하게 됩니다. 이는 통상 tree(트리)로 나타내며, relational algebra(관계형 대수)를 모방한 내용을 담고 있습니다. Project(`SELECT` 절에 해당), Select(`WHERE` 절에 해당), Join, Sort 등의 논리적인 연산자가 포함되어 있습니다. 대략 자료구조 시간 때 산술식을 tree로 저장하는 것과 비슷하게 SQL을 tree로 저장하는 역할을 한다고 생각하시면 됩니다.

**Query Optimizer**는 이 logical plan을 입력으로 받아 **최적화된 physical plan**(물리 계획; execution plan 혹은 실행 계획이라고도 불림)을 생성합니다. 이 과정에서 이름에 걸맞게 logical plan 혹은 physical plan을 더 좋은 성능을 가지도록 최적화합니다. Physical plan은 logical plan을 어떻게 실행할지에 대한 정보를 포함하고 있습니다. 예를들어 위의 logical plan에는 "Join을 한다"라는 정보를 포함하고 있다면, physical plan에는 "Join을 Hash Join을 통해 처리한다"라는 정보를 포함하고 있습니다.

Physical plan은 (이미지의) Execution Engine에게 전달 및 처리되게 됩니다. **Execution Engine은 Storage Engine이 제공하는 데이터 형태에 따라 처리를 합니다.** 예를들어 column oriented layout을 이용하여 데이터를 관리하는 storage engine을 사용한다면, 내부 데이터 형태 또한 column oriented data structure로 관리될 것입니다(e.g., 예를들어 `int` 타입의 컬럼이라면, `vector<int> columnChunk` 와 같은 형태로 데이터를 관리합니다).

이 섹션에서 Query Processor, Execution Engine, Storage Engine의 역할을 간략하게 알아보았습니다. Apache Arrow는 여기에서 Storage Engine의 데이터 관리 형태, Apache DataFusion은 Query Processor와 Execution Engine에 해당한다고 볼 수 있습니다. 자세한 내용을 앞으로 살펴보겠습니다.

## Apache Arrow

Apache Arrow는 **빠른 데이터 교환** 및 **인메모리 분석**을 위한 **columnar format** 및 **multi-language toolbox** 입니다. 많은 사용자들의 이를 여러 프로세스나 언어간에 연결하는 장치로 사용하거나, 데이터를 빠르게 분석할 수 있는 툴로써 사용합니다.

이 섹션에서는 Apache Arrow의 **1) multi-language를 지원하며 빠르게 데이터를 교환할 수 있고**, **2) columnar in-memory format을 이용하여 데이터를 빠르게 처리할 수 있는** 점에 대해 중점적으로 논의해보도록 하겠습니다.

### Multi-language 지원 및 빠른 데이터 교환

Apache Arrow는 **Zero-copy 데이터 교환**을 통해 데이터를 굉장히 빠르게 옮길 수 있습니다. Zero-copy 데이터 교환이란 데이터를 교환할 때 데이터의 복사를 전혀 하지 않는다는 의미입니다. 우리가 데이터를 특정 프로세스에서 다른 프로세스로 옮기고자 할 때 사용할 수 있는 간편한 방법이 몇가지 있습니다. 데이터를 파일에 쓴 뒤 다른 프로세스에서 읽거나, 통신채널을 열어 데이터를 옮기는 방법이 있습니다. 하지만 이들은 하나의 프로세스가 사용하는 메모리에서 데이터를 disk 혹은 네트워크로 옮기고, 이를 다시 다른 프로세스의 메모리에서 읽어들어야 하는 과정을 수반합니다. 즉, 프로세스의 메모리에서 다른 프로세스의 메모리로 데이터가 copy(복사)되는 것이지요. 데이터의 사이즈가 작을 때에는 이것이 문제되지 않지만, 대규모 데이터 분석과 같이 데이터의 사이즈가 커짐에 따라 데이터 복사에서 발생하는 시간이나 메모리 사용은 궁극적으로 성능 하락으로 이루어질 수 있습니다.

![Shared Memory](/assets/images/2025-04-26-apache-arrow-and-datafusion/shared-mem.webp){: width="500" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*Shared Memory ([출처](https://medium.com/@rohitkumar_55648/linux-shared-memory-a01c6a8121e))* 
{: refdef}

다행히도 이를 해결하는 방법은 우리가 운영체제 시간때 배웠던 **공유메모리(Shared Memory)**를 통해서 해결할 수 있습니다. 위의 그림과 같이 shared memory는 여러 프로세스가 하나의 메모리 공간에 동시에 접근할 수 있도록 합니다. 하나의 프로세스가 메모리 공간에 데이터를 쓰고 다른 프로세스는 그 메모리 공간에 접근하여 데이터를 읽을 수 있게 되는 것이지요. 즉, 데이터 복사 없이 정보를 교환할 수 있게 되는 것입니다.

Apache Arrow는 이러한 공유메모리 기능을 통해 zero-copy 데이터 교환을 실현합니다. Apache Arrow는 공유 메모리를 할당받고 이곳에 자신들의 자료구조나 상태를 설정하며, 이후 모든 데이터는 이 공유 메모리 내에 관리를 하게 됩니다. 추후에 다른 프로세스와 데이터 교환이 필요하다면, 그 프로세스가 공유메모리에 접근할 수 있도록 세팅해주면 됩니다. 물론 서로 다른 프로세스(다른 언어 혹은 다른 시스템)에서 접근하기 위해 공유메모리서 관리되는 자료구조나 상태를 해석할 수 있는 기능이 필요합니다. Apache Arrow는 이것 또한 제공을 하며, 이를 언급하기 위해 **multi-language toolbox**라는 용어를 사용한 것으로 보입니다.

서로 다른 컴퓨터간 데이터 교환하는 경우는 데이터를 다른 컴퓨터로 전송해야 하며, 이는 필연적으로 데이터 복사가 발생하게 됩니다. 이를 위해 Apache Arrow Flight가 사용됩니다.

그럼 Apache Arrow는 어떻게 메모리 상에서 유저의 데이터를 관리할까요? 다음 섹션에서 알아보도록 하겠습니다.

### Columnar in-memory format

Apache Arrow는 데이터 교환뿐 아니라 빠른 인메모리 분석 도구로도 유용하게 사용됩니다. 예를들어, Pandas나 Polars와 같은 라이브러리에서 데이터 분석 작업 속도를 향상시키는 데 자주 사용됩니다. 그렇다면 어떻게 Apache Arrow는 이런 성능을 낼 수 있을까요? 여기에는 columnar format이 핵심적인 역할을 하고 있습니다.

Columnar format은 테이블 데이터를 저장하는 방법 중 하나입니다. 전통적인 RDBMS는 테이블 데이터를 row oriented(행 기반) format으로 관리하였습니다. 예를들어 `Student(sid INTEGER, name STRING, age INTEGER)`라는 테이블이 있다고 가정해 보겠습니다. Row oriented format의 경우 물리적으로 데이터를 `1,구교승,29$2,배정모,29$3,서정범,30$4,배현모,31`과 같은 식으로 각 행에 대한 정보를 연속된 공간에 저장합니다(`,`는 열의 구분을, `$`은 행의 구분을 위해 사용되었습니다). 반면 column oriented format의 경우 물리적으로 데이터를 `1,2,3,4$구교승,배정모,서정범,배현모$29,29,30,31`과 같이 각 열에 대한 정보를 연속된 공간에 저장합니다(`,`는 행의 구분을, `$`은 열의 구분을 위해 사용되었습니다).

그렇다면 왜 columnar format을 사용할까요? Columnar format은 분석 쿼리를 잘 처리하기 위해 사용됩니다. 분석 쿼리는 대부분 단일 행을 접근하는 것보다는 여러 행을 접근하여 집계를 하는 동작을 수행합니다. `SELECT SUM(age) FROM Student`라는 쿼리가 분석 쿼리의 간단한 예시가 될 수 있겠습니다. 이것을 row oriented format으로 처리한다고 가정해 보겠습니다 (이전 단락의 예시와 함께 보세요). DBMS는 물리적으로 저장된 데이터를 읽어드린 뒤에 각 행별로 가장 마지막 column `age`를 찾아가 읽어내야 합니다. 그 뒤에 이 값들을 합산하겠지요. 반면 column oriented format으로 처리한다고 생각해 보겠습니다. DBMS는 특정 열를 찾은 뒤에 연속적으로 저장되어 있는 `age`값을 한번에 읽어들이기만 하면 됩니다. 이러한 접근 방식의 차이점은 storage의 sequential I/O를 더 적극적으로 활용할 수 있고(연속된 `age`값을 읽으면 되기 때문에), CPU의 caching 측면에서도 유리하게 작용합니다. 궁극적으로 성능상 이득이 있는 것이지요.

더 나아가 vectorized execution도 column format이 유리한 조건을 만들어 냅니다. 요즘 CPU는 효율적인 vector processing을 위한 Single Instruction Multiple Data(SIMD) 연산을 지원합니다. Columnar format은 함께 처리해야 하는 데이터가 이미 연속된 공간에 존재하기 때문에(`age` 열의 값이 `29,29,30,31`과 같이 연속된 공간에 저장), 이러한 SIMD 연산을 위한 최적의 상태에 있습니다. 만약 행 기반의 데이터였다면 loop을 통해 최소 4개 이상의 instruction이 필요했겠지만, SIMD 연산으로는 하나면 충분합니다.

Apache Arrow는 이런 columnar format의 장점을 극대화합니다. Apache Arrow 사용자의 데이터를 columnar format으로 메모리 상에서 관리하고, SIMD 연산과 효율적인 구현으로 빠른 데이터 처리 성능을 제공합니다.

### 더 읽어볼 거리

- Apache Arrow의 columnar format은 구체적으로 어떻게 생겼는가? [링크](https://arrow.apache.org/docs/format/Columnar.html#physical-memory-layout)
- Apache Arrow는 구체적으로 어떻게 IPC를 실현하는가? 어떻게 shared memory를 관리하고 어떤 IPC format으로 데이터로 통신을 수행할까? [링크](https://arrow.apache.org/docs/format/Columnar.html#serialization-and-interprocess-communication-ipc)
- Column Store가 Row Store보다 분석 쿼리에서 더 좋은 성능을 내는 이유는 무엇일까? [링크](https://www.cs.umd.edu/~abadi/papers/abadi-sigmod08.pdf)
- 어떻게 columnar format에서 variable-length type(e.g., VARCHAR)나 nested object(e.g., structure)를 관리할 수 있을까? [variable-length](https://arrow.apache.org/docs/format/Columnar.html#variable-size-list-layout), [structured layout](https://arrow.apache.org/docs/format/Columnar.html#struct-layout)

## Apache DataFusion

Apache DataFusion은 **Apache Arrow를 인메모리 포멧**으로 사용하는 **확장성** 있는 **쿼리 엔진**입니다. 즉, columnar format을 기반으로 SQL 질의를 처리하는 엔진이며 이것의 확장성이 좋다는 의미이겠지요. 많은 사용자들은 DataFusion을 자신의 프로세스에 임베딩하여 SQL 혹은 DataFrame engine으로 사용합니다.

DataFusion는 새로운 DBMS를 구축할 때 고품질의 오픈소스 쿼리 엔진을 활용하는 것이 미래의 트렌드가 될 것이라고 제시하고 있습니다 ([출처](https://docs.google.com/presentation/d/1D3GDVas-8y0sA4c8EOgdCvEjVND4s2E7I6zfs67Y4j8/edit#slide=id.g22007bd2b6f_0_343)).
전통적으로 각 데이터베이스 시스템이 자체적인 쿼리 엔진을 개발해왔지만 (예를들어 MySQL과 Postgres와 같은 시스템들은 제각각 다른 구성을 가지고 있죠), 이는 구축과 유지보수에 많은 비용이 든다고 분석하고 있습니다.  DataFusion은 새로운 DBMS를 만들 때 처음부터 만들지 말고, DataFusion과 같이 잘 구성된 시스템을 모듈 형식으로 가져가 쓰자고 제안합니다.
그 뒤에 개발하고자 하는 DBMS 특징에 맞게 기능을 추가하거나 코드를 수정하자고 말하죠.

그런 이유인지 DataFusion은 전통적인 DBMS와 매우 유사하게 구성되어 있습니다. 이번 장에서는 어떤 식으로 DataFusion 쿼리 엔진이 구성되어 있고, 어떻게 확장성이 좋은지에 대해서 알아보도록 하겠습니다.

### Query Engine Components

![DataFusion architecture](/assets/images/2025-04-26-apache-arrow-and-datafusion/datafusion-arch.jpg){: width="600" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*DataFusion 개요 ([출처](https://docs.google.com/presentation/d/1D3GDVas-8y0sA4c8EOgdCvEjVND4s2E7I6zfs67Y4j8/edit#slide=id.p))* 
{: refdef}

위 그림은 DataFusion의 구성도를 표현하고 있습니다. 좌측 상단에 있는 것은 Data Sources로, 우리가 맨 처음 살펴본 DBMS 구성에서 Storage Engine이 읽고 쓰는 데이터의 원천입니다. 좌측 하단은 사용자가 SQL과 DataFrame을 이용하여 DataFusion에 쿼리를 할 수 있다는 것을 의미합니다. 그림의 중앙부에는 LogicalPlans과 ExecutionPlan으로 이루어진 Plan Representations가 배치되어 있습니다. 이것은 DBMS 설명에서 논의한 logical plan(논리 계획)과 physical plan(물리 계획)과 동일한 것입니다. 우측에 있는 것은 Optimized Execution Operators로 ExecutionPlan에 포함되어 실행될 수 있는 연산자(operator)를 의미합니다.

그림에서 알 수 있듯이, DataFusion은 DBMS의 쿼리엔진이 하는 것과 동일한 일을 하고 있습니다. FrontEnds로부터 입력된 쿼리를 가공하여 logical plan을 만들고, 이것은 변환(Transformation) 혹은 최적화(Optimization)을 통해 더 나은 logical plan으로 변환됩니다. DataFusion은 생성된 plan을 실제로 어떻게 실행할지를 나타내는 execution plan으로 변환합니다. 이 과정에서 위와 유사하게 변환과 최적화를 수행하게 됩니다. 이후에 Arrow를 기반으로 작성된 연산자를 이용하여 execution plan을 실행합니다.

### 쿼리 최적화

![DataFusion Query Optimization](/assets/images/2025-04-26-apache-arrow-and-datafusion/datafusion-qo.png){: width="600" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*DataFusion Logical Plan Optimization ([출처](https://docs.google.com/presentation/d/1ypylM3-w60kVDW7Q6S99AHzvlBgciTdjsAfqNP85K30/edit#slide=id.p))* 
{: refdef}

DataFusion의 logical plan 최적화는 위의 그림과 같이 여러 단계로 구성되어 있습니다. Logical plan이 입력으로 주어지면, optimizer pass 1을 통해 조금 더 성능이 개선된 logical plan을 만들고, 그것을 입력으로 optimizer pass 2를 통해 조금 더 성능이 개선된 logical plan을 만들고, 이런 과정을 반복하여 최종 logical plan을 만들어 냅니다. 각각의 optimizer pass는 내장된 rule에 따라 규정이 되어 있으며, 이는 rule의 조건에 맞는지 혹은 예측되는 성능에 따라 결과 logical plan에 반영이 될수도 있고 되지 않을수도 있습니다. 각각의 optimizer pass에는 아래와 같은 일들을 수행합니다.

- Pushdown: Projection, Limit, Filter 연산자를 query plan의 시작점 쪽으로 이동
- Simplify-1: 쿼리 실행 중 expression evaluation을 최소화
- Simplify-2: 필요 없는 연산자 제거
- Flatten Subqueries: Nested 쿼리(subquery)를 join으로 대체
- Optimize Joins: Join 연산 최적화
- Optimize DISTINCT: DISTINCT 연산 최적화

Execution plan은 위 과정에서 생성된 최종 logical plan으로부터 생성되며, 이 또한 logical plan과 동일하게 여러 optimizer pass를 통해 최적화됩니다. 이 과정에서 적용되는 optimization은 아래와 같습니다.

- Enforce Sort/Partitioning: 데이터가 sorting 혹은 partitioning이 필요한지 판단
- Pick Algorithm: sorting/partitioning 여부를 기반으로 join과 sort 연산의 알고리즘을 결정
- Use Statistics: 통계정보를 확인하고 가능한 경우 Scan을 대체

DataFusion의 특징 중 하나는 확장성입니다. 이에 걸맞게 쿼리 최적화 기능 또한 사용자가 직접 추가할 수 있습니다. 구체적인 내용은 더 읽어볼 거리에 남겨두도록 하겠습니다.

### 성능 측면의 특징

Apache DataFusion은 빠른 쿼리 실행 성능을 장점으로 내세우고 있습니다. 크게 asynchronous I/O, vectorized processing, partitioning을 통한 multi-core processing이 쿼리 성능에 기여를 합니다.

Asynchronous I/O는 I/O를 비동기식으로 처리하는 것을 의미합니다. I/O의 대상이 되는 disk나 network의 경우, CPU에 비해 엄청나게 느리게 동작합니다. 따라서 CPU가 I/O를 요청하게 되면 요청이 처리될 때 까지 (즉, HDD에서 데이터를 읽거나, 네트워크를 통해 데이터를 성공적으로 전송할 때 까지) CPU는 할 일이 없게 됩니다. 이 때 CPU가 다른 일을 하지 않고 기다리는 것을 Synchronous I/O라고 하고, 그 사이에 다른 일을 하는 것을 Asynchronous I/O라고 합니다. DataFusion은 Async I/O 방식을 채택하여 사용하고 있습니다.

Vectorized processing은 이전 장에서 설명한 것처럼 SIMD 연산과 궁합이 잘 맞습니다. 더욱이 DataFusion은 columnar 방식인 Arrow를 기반으로 하여 더 좋은 성능을 낼 수 있습니다.

![DataFusion Data Partitioning](/assets/images/2025-04-26-apache-arrow-and-datafusion/datafusion-partition.png){: width="700" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*DataFusion Data Partitioning ([출처](https://docs.google.com/presentation/d/1cA2WQJ2qg6tx6y4Wf8FH2WVSm9JQ5UgmBWATHdik0hg/edit#slide=id.g209d99697c0_0_11))* 
{: refdef}

DBMS 맥락에서 partitioning은 데이터를 특정한 기준에 맞게 물리적으로 그룹화 하는 것을 의미합니다. DataFusion은 위의 그림과 같이 데이터를 partition으로 쪼갠 뒤에, data-parallel 한 방식을 이용하여 쿼리를 수행합니다.

여기까지 DataFusion에 대해서 알아보았습니다.
독자들에게 유용할 수 있을 링크를 더 읽어볼 거리에 추가하였으니 기회가 된다면 살펴보시기 바랍니다.

### 더 읽어볼 거리

- DataFusion은 어떻게 각각의 연산자를 호출하고 데이터를 생성할까? 구체적으로 어떤 query execution model을 사용할까? [Volcano-style query execution model](https://docs.rs/datafusion/latest/datafusion/#execution)
- DataFusion은 구체적으로 어떻게 쿼리를 최적화할까? [doc](https://datafusion.apache.org/library-user-guide/query-optimizer.html), [code](https://github.com/apache/datafusion/tree/main/datafusion/optimizer)
- 어떻게 사용자 정의 쿼리 최적화 룰을 추가할 수 있을까? [링크](https://datafusion.apache.org/library-user-guide/query-optimizer.html#writing-optimization-rules)
- 어떻게 사용자 정의 함수 및 집계를 추가할 수 있을까? [링크](https://datafusion.apache.org/python/user-guide/common-operations/udf-and-udfa.html)

## 마치는 말

DBMS를 중점적으로 연구/개발하는 사람들은 최근 DataFusion과 Arrow를 포함하는 Flight DataFusion Arrow Parquet stack(줄여서 FDAP stack)에 관심이 많은 것 같습니다. 특히 이 stack을 이용해서 DBMS를 새로 만드는 것에 말입니다. 대표적인 예시로 InfluxDB의 경우 버전 2까지는 자신들이 개발하던 아키텍처를 사용했지만, 버전 3는 FDAP stack으로 새로운 시스템을 개발했다고 합니다 ([출처](https://youtu.be/AGS4GNGDK_4?si=61Kom1xSWZAFlmTa)).

지금까지 FDAP stack에서 Apache Arrow와 DataFusion의 기술적인 특징에 대하여 논의하였습니다.
이는 DBMS의 Query Engine(Query Processor 및 Execution Engine)과 Storage Engine의 일부를 커버하고 있습니다.
[다음 글]({% post_url 2025-05-18-apache-parquet-and-opendal-ko %})에서는 더 아래단의 Storage Engine과 밀접한 Parquet에 대해서 살펴보도록 하겠습니다. 또한 최근에 여러 data format을 통합된 레이어로 읽을 수 있는 layer인 OpenDAL에 대해서도 살펴보도록 하겠습니다.