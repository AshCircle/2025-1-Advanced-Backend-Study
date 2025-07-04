# Weekly I Learned 9

## Transaction Processing and Recovery

### Recovery

DBMS는 여러 하드웨어와 소프트웨어 컴포넌트로 구성되고, 컴포넌트 각각은 각자 다른 이유로 장애가 발생할 수 있다. DBMS 개발자는 이러한 장애들을 고려하여 약속된 데이터가 실제로 저장될 수 있게 해야 한다.

DBMS의 전원 공급이 중단되는 상황을 고려해보자. RAM에 저장되는 페이지 캐시는 데이터가 날아갈 것이다. 그렇다면 페이지 캐시에서 Flush하지 않은 Dirty 페이지는, DB 장애(정전 등)가 일어났을 때 이를 반영하지 못하므로 ACID의 지속성 원칙을 위배하는 것이 아닐까?

DB는 트랜잭션의 커밋을 완료하는 조건으로 **WAL**가 디스크에 기록되어야 함을 두고 있다. WAL(*Write-Ahead-Logging*, 로그 선행 기입 or 선행 기록 로그)란, 장애 및 트랜잭션 복구를 위해 디스크에 저장하는 추가 전용 보조 자료 구조이다.

페이지 변경 사항을 디스크에 바로 반영하면 안전하지만 너무 느리고, RAM(페이지 캐시)에만 반영하면 빠르지만 장애시 복구가 불가능하다. WAL은 페이지 변경 사항을 디스크에 바로 반영하는 대신 디스크에 로그로 기록하여, 안정성과 속도의 균형을 잡는다.

- **Log Semantics**

    DBMS는 데이터를 디스크에 반영하기 전에 변경 내용을 WAL로 먼저 기록한다. WAL은 한 줄씩 덧붙이는(append-only) 방식이기 때문에, 이전 로그는 절대 바뀌지 않는다. 

    WAL은 Log Record라는 단위로 구성되며, LSN(*Log Sequence Number*)가 시간 순서대로 커지며 각 Log Record마다 부여된다. 이 Log Record는 먼저 메모리 상의 Log Buffer에 저장되며, force 작업 시 디스크에 기록된다. Log Buffer가 가득 찬 경우, 또는 하나의 트랜잭션이 커밋되는 경우가 바로 force 작업이다. 따라서, DB가 트랜잭션의 커밋이 완료되었다고 판단하는 조건은, 트랜잭션의 WAL, 즉 모든 Log Record가 디스크에 기록되는 것이다.

    - Checkpoint

        DB와의 트랜잭션이 계속 이어지고 WAL이 쌓이고 있다고 가정하자. 시스템 장애로 인해 DBMS가 WAL을 참고하여 Recovery를 시도한다면, 어느 시점부터 참고하여 복구를 해야 할까? DB의 첫 WAL부터 시도해야 할까? 수많은 WAL이 쌓였을 텐데, 처음부터 모든 WAL를 참고하는 것은 굉장히 비효율적일 것임이 자명하다.
        
        DBMS는 이를 판단하는 기준으로 Checkpoint를 사용한다. Checkpoint는 해당 시점 이전의 변경사항이 전부 디스크에 반영되었으므로 이전 WAL은 Recovery 작업에 필요 없음을 시사한다.

        DBMS는 Checkpoint를 다음의 과정을 거쳐 만든다.

        1. Log Buffer의 모든 Log Record를 디스크에 저장한다.
        2. 페이지 캐시의 모든 Dirty 페이지를 디스크에 저장한다.
        3. 진행중이던 트랜잭션의 정보를 포함해서 Checkpoint 로그를 남기고 디스크에 저장한다.

        Sync Checkpoint 방식은 동기적으로 작동한다. 트랜잭션이 실행되고 있다면, Log Buffer 및 페이지 캐시가 계속 변경될 것이다. Checkpoint는 **특정 시점**까지 안전하게 저장되었음을 나타내는 기준인데, Checkpoint 설정 작업 중에 Log 및 Dirty 페이지가 계속 생긴다면 **특정 시점**을 고정시키가 어렵다. 따라서 Sync Checkpoint 방식에서는 시스템 중단이 무조건적으로 필요하다.

        Fuzzy Checkpoint는 Sync Checkpoint의 시스템 중단 문제를 해결하는 비동기적 방식이다. 이 방식은 시스템 중단을 하지 않고, 특정 시점의 dirty 페이지, 트랜잭션 테이블을 스냅샷으로 따로 보관한 후, 백그라운드에서 이를 flush 한다. 이를 통해 시스템 중단 없이 특정 시점의 Checkpoint를 만들 수 있다.

- **Operation Versus Data Log**

    섀도 페이징(*Shadow Paging*)이란, 페이지 수정 시 기존 페이지를 건드리지 않고 새로 만든 복사본을 수정하는 기법이다. 이를 쓰기 시 복사(*copy-on-write*)라고도 부르며, 이전 페이지가 남아 있으므로 문제가 생기면 쉽게 Roll-back이 가능하다는 점이 장점이다.

    WAL은 물리적 로그 또는 논리적 로그로 기록할 수 있다.

    - 물리적 로그: 작업 수행 전후 상태를 저장한다.
    - 논리적 로그: 작업 자체를 저장한다.

    보통 DBMS는 Undo(*After -> Before*) 작업의 경우 논리적 로그(동시성 및 성능 보장), Redo(*Before -> After*) 작업의 경우 물리적 로그(상태 덮어쓰면 되므로 시간 단축)를 사용한다.

- **Steal and Force Policies**

    DBMS에서 데이터를 RAM에서 수정한 후(Dirty 페이지가 만들어진 후), 언제 디스크에 반영할지 결정해야 한다. 반영 여부를 결정하는 정책은 Undo 작업, Redo 작업 관련으로 분류할 수 있다.

    Undo 작업과 관련된 정책으로는 Steal, No-Steal 정책이 있다.

    - Steal

        트랜잭션이 커밋되지 않았더라도 Dirty 페이지를 디스크에 반영할 수 있다. Recovery 작업 시, 커밋되지 않았는데 디스크에 반영된 페이지가 있을 수 있으므로, 이 페이지에 대한 Undo 로그가 필요하다.

    - No-Steal
    
        트랜잭션이 커밋되지 않으면 Dirty 페이지를 디스크에 반영하지 않는다. Recovery 작업 시, 커밋되지 않은 페이지는 디스크에 반영되지 않았으므로, Undo 로그는 필요없다.

    Redo 작업과 관련된 정책으로는 Force, No-Force 정책이 있다.

    - Force

        트랜잭션이 커밋될 때, 트랜잭션이 변경한 모든 Dirty 페이지를 디스크에 반영한다. Recovery 작업 시, 커밋된 페이지는 전부 디스크에 반영되어 있으므로, Redo 로그는 필요없다.

    - No-Force

        트랜잭션이 커밋되더라도, 트랜잭션이 변경한 Dirty 페이지는 나중에 디스크에 반영한다. Recovery 작업 시, 커밋되었으나 디스크에 반영되지 않은 페이지가 있으므로, 이 페이지에 대한 Redo 로그가 필요하다.

- **ARIES**

    ARIES(*Algorithms for Recovery and Isolation Exploiting Semantics*)는 Steal/No-Force 정책 기반의 복구 알고리즘이다. 물리적 Redo 로그, 논리적 Undo 로그를 사용하고, WAL, CLR(복구 작업 중 로그), Fuzzy Checkpoint 등 우리가 배운 복구 정책을 대부분 사용한다.

    ARIES의 Recovery 작업은 다음 3단계로 진행된다.

    1. Analysis

        장애 발생 시 어떤 트랜잭션이 진행 중이었는지 조사하고, 이를 바탕으로 Undo 대상 트랜잭션을 정한다. 또한 디스크에 기록되지 않은 Dirty 페이지를 확인하여, 이를 바탕으로 Redo 시작 지점을 결정한다.

    2. Redo

        장애 전까지의 작업을 모두 Redo 한다. 커밋되지 않은 트랜잭션도 포함해서 Redo한다.

    3. Undo

        Redo 하면서 커밋되지 않은 트랜잭션도 Redo 했었는데, Undo 과정에서는 이렇게 Redo한 커밋되지 않은 트랜잭션을 다시 Undo한다.

    왜 ARIES는 커밋되지 않은 트랜잭션도 Redo 해서 Undo라는 작업을 한번 더 할까? 그 이유는 트랜잭션 간의 의존성을 정확히 처리하고, 복구 중 재장애를 견디기 위해서이다. undo를 통해 커밋되지 않은 트랜잭션에 대해 CLR 로그를 남길 수 있기 때문에, 복구 중 재장애가 발생해도 중복되는 undo 없이 정확히 이어서 할 수 있다.

### Concurrency Control

동시성 제어는 다음과 같이 분류할 수 있다.

- 낙관적 동시성 제어(*OCC, Optimistic Concurrency Control*)

    트랜잭션 간 충돌이 나지 않을 것이라 가정하고 트랜잭션을 자유롭게 실행한다. 각 트랜잭션 커밋 전에 충돌이 발생하는 것이 확인될 경우, 트랜잭션 하나를 Rollback 한다.

- 다중 버전 동시성 제어(*MVCC, Multiversion Concurrency Control*)

    하나의 레코드에 여러 버전을 저장한다. 각 트랜잭션은 과거의 특정 버전을 읽는 것처럼 작동하여 트랜잭션 간 충돌을 방지한다.

- 비관적 동시성 제어(*PCC, Pessimistic Concurrency Control*)

    트랜잭션 간 충돌이 날 것이라 생각하고 미리 대비한다. 이 대비책 중 대표적인 것이 바로 락(*Lock*)이다. 트랜잭션이 어떤 레코드를 읽거나 쓰려면, 락을 걸어서 다른 트랜잭션이 접근하지 못하게 한다. 여러 트랜잭션이 서로의 락을 기다리는 데드락(*DeadLock*)이 발생할 수 있다.

- **Serializability**

    스케줄(*Schedule*)이란, DB 입장에서 본 트랜잭션 작업(Read, Write, Commit, Abort)의 실행 순서를 말한다. 관련 트랜잭션의 모든 작업이 포함된 스케줄을 Complete Schedule이라 하며, 여러 트랜잭션이 어떻게 수행되더라도(ex. 병렬 실행), 트랜잭션을 순서대로 하나씩 실행했을 때와 같다면 Correct Schedule이라 한다.

    Correct Schedule을 보장하는 가장 쉬운 방법은 직렬 스케줄(*Serial Schedule*)이다. 트랜잭션이 교차하지 않고 하나씩 차례로 완전히 실행된 후 다음 트랜잭션이 실행된다면 이를 Serial Schedule이라고 한다. 그러나 항상 이렇게 실행하면 처리량이 제한적이므로 성능 면에서 좋지 않을 것이다.

    처리량을 늘리기 위해서는 병렬 처리가 필요하고, 동시에 Correct Schedule임이 보장되어야 한다. 트랜잭션을 병렬로 실행하더라도, 결과가 어떤 Serial Schedule과 동일하다면 이를 직렬 가능 스케줄(*Serializable Schedule*)이라고 한다. Serializable Schedule은 성능과 데이터 무결성을 둘 다 만족하며, 이를 찾는 것이 DBMS 동시성 제어의 목표이다.

- **Transaction Isolation**

    트랜잭션 격리 수준은, 트랜잭션이 다른 트랜잭션으로부터 얼마나 독립적으로 실행되는지를 결정한다.

    이러한 격리 수준에는 데이터 일관성과 성능 사이에 Trade-Off가 존재한다. 격리 수준을 엄격하게 하면 데이터 일관성이 좋아지는 대신 성능이 나빠지며, 격리 수준을 느슨하게 하면 성능이 좋아지는 대신 이상 현상(*Anomaly*)이 발생할 수 있다.

- **Read and Write Anomalies**

    트랜잭션의 격리 수준에 따라, 발생할 수 있는 Anomaly의 종류가 상이하다.

    읽기 이상(*Read Anomalies*)으로는 다음 이상 현상이 발생할 수 있다.

    - 더티 읽기 (*Dirty Read*)

        커밋되지 않는 값(Rollback된 값)을 읽는 현상이다.

    - 반복 불가능 읽기 (*Non-repeatable Read*)

        같은 데이터를 두번 읽었는데 값이 서로 다른 현상이다. 

    - 팬텀 읽기 (*Phantom Read*)

        같은 '범위' 데이터를 두번 읽었는데 값이 서로 다른 현상이다.

    쓰기 이상(*Write Anomalies*)으로는 다음 이상 현상이 발생할 수 있다.

    - 갱신 분실 (*Lost Update*)

        두 트랜잭션이 같은 데이터를 수정할 때, 한 쪽의 변경이 덮어씌워지는 현상이다.

    - 더티 쓰기 (*Dirty Write*)

        커밋되지 않은 값(Rollback된 값)을 기반으로 수정 및 커밋하는 현상이다.

    - 쓰기 치우침(*Write Skew*)

        각 트랜잭션이 개별적으로는 조건을 만족하지만, 동시에 실행되어 조건을 불만족시키는 경우이다.

- **Isolation Levels**

    트랜잭션의 격리 수준으로는 다음 4가지 격리 수준이 존재하며, 하위로 갈 수록 강력한 격리 수준을 보장한다.

    - Read Uncommitted

        커밋되지 않은 데이터도 읽을 수 있다. Dirty Read, Non-repeatable Read, Phantom Read가 발생할 수 있다.

    - Read Commited

        커밋된 데이터만 읽을 수 있다. Non-repeatable Read, Phantom Read가 발생할 수 있다.

    - Repeatable Read

        같은 데이터를 두번 읽어도 항상 같은 값이 보장된다. Phantom Read가 발생할 수 있다.

    - Serializable

        모든 트랜잭션을 Serial하게 실행한 것과 같은 결과를 보장한다. Read Anomalies를 모두 방지하나, 다른 격리 수준에 비해 성능이 떨어질 수 있다.
