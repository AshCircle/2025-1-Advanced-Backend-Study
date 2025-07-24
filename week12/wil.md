# Weekly I Learned 12

## B-Tree Variants

### Bw-Trees

앞서 배운 B-Tree 구조는 읽기 및 쓰기를 무조건 페이지 단위로 처리했다. 이는 다음 부수적인 문제들을 불러일으킨다.

1. Write Amplification (쓰기 증폭)

    B-Tree는 페이지 단위로 쓰기를 처리하므로, 페이지의 일부만 바꾸더라도 전체 페이지를 다시 써야 한다. 즉, 조금의 변경이 불필요하게 많은 쓰기 작업을 요한다.

2. Space Amplification (메모리 증폭)

    B-Tree는 페이지 단위로 쓰기를 처리하고, 다음 쓰기 요청을 위해서는 페이지에 빈 공간을 남겨두어야 한다. 즉, 쓸모 없는 빈 공간도 함께 디스크에 전송된다.

3. Concurrency Issues (동시성 문제)

    여러 스레드가 동시에 B-Tree를 페이지 단위로 읽으므로, 같은 페이지에 접근하려면 Latch를 통해 페이지에 Lock을 걸어야 한다. Lock을 사용하면 성능이 하락한다.

Bw-Tree(*Buzzword Tree*)는 이러한 문제를 해결하기 위해 전혀 다른 방법을 사용한다.

- **Update Chains**

    Bw-Tree에서 말하는 Node는, 여러 Node들이 묶인 Logical Node를 말한다. 이 Logical Node는 원래의 데이터를 담은 Base Node, 그리고 이에 대한 변경사항을 담은 여러 Delta Node들로 구성된다.

    좀더 자세히 알아보자면, Base Node에 변경사항이 생길 때마다 그 변경사항(ex. Insert, Update, Delete) 정보를 담은 Delta Node를 앞에 붙인다. 이렇게 Delta Node가 꼬리에 꼬리를 물고 Base Node에 붙어 있는 것을 **Update Chain** 이라고 하고, 이렇게 하나의 Logical Node가 구성되는 것이다.

    이 Logical Node를 읽으려면, 모든 Delta Node를 거슬러 올라가 Base Node까지 다 읽고 적용해야 한다. 따라서 읽는 속도는 일반적인 B-Tree보다 느릴 수 있다.

- **Taming Concurrency with Compare-and-Swap**

    Bw-Tree에서, 부모 Logical Node는 자식 Logical Node를 가리키고, 당연하게도 각 Logical Node는 자신의 최신 Delta Node를 알고 있어야 한다. 따라서 Bw-Tree는 Logical Node ID와 해당 노드의 최신 Delta Node ID를 매핑하여 저장하고 있다.

    CAS(*Compare-and-Swap*)은, 원자성을 보장하는 교체 연산이다. 어떤 스레드가 임의의 메모리 값을 바꿀 때 CAS 연산을 적용하면, 현재 메모리 주소 값이 기대한 상태인지 **비교**하고, 기대한 상태라면 **교체**한다.

    Bw-Tree에서 어떻게 Node 업데이트가 일어나고, 어디서 CAS 연산이 적용되는지 알아보자.

    1. 먼저, 루트 노드부터 리프 노드까지 순회하며, 업데이트 대상 Logical Node를 찾는다. Logical Node에서 가장 최신의 Node(Base Node일수도 있고, Delta Node일 수도 있다.)의 ID 정보를 매핑 테이블에서 가져온다.

    2. 이 최신 Node를 가리키는 새로운 Delta Node를 생성한다. 즉, 기존 Update Chain 앞에 붙는 구조를 가진다.

    3. 매핑 테이블에서 해당 Logical Node ID의 매핑 값을 새로운 Delta Node로 교체한다. 이때, **CAS**가 사용되어, 매핑 테이블의 값이 이전에 기억한 기존 포인터라면 성공하고, 아니라면 실패한다.

    이렇게 원자성을 보장하는 CAS 연산을 통해, Latch 없이도 동시성 문제를 안전하게 처리할 수 있다.

- **Structural Modification Operations**

    Logical Node들로 구현된 Bw-Tree는, 논리적으로 B-Tree와 구조가 동일하기 때문에 자연스레 노드 분할 및 병합과 같은 구조 변경 작업(SMO, *Structural Modification Operations*)이 필요하다. 그러나 Bw-Tree는 Latch가 존재하지 않으므로, 완전히 다른 방식으로 처리된다.

    - Split SMO

        1. 새로운 형제 Node에 정보를 넘기기 위하여, 분할 대상 Node의 모든 Delta를 모아 내용을 정리한다. (이것이 후술하는 Node Consolidation을 이야기하는 것은 아니다.)

        2. 새로운 형제 Node를 만들고, 기존 Node에는 Split Delta Node를 추가한다. 이 Delta Node는 아직 분할이 진행 중임을 나타내며, 분할 기준이 되는 구분 키(midpoint key)와 새로운 형제 Node 포인터 정보가 있다.

        3. 새로운 형제 Node는 당장 부모 Node와 연결되어 있지 않으나, 분할 Node의 Split Delta Node 형제 포인터를 통해 접근 가능하다.

        4. 성능 최적화를 위해, 새로운 형제 Node를 부모 Node의 자식 포인터로 연결한다.

    - Merge SMO

        1. 오른쪽 형제 Node에 곧 없어질 예정이라는 의미로 Remove Delta Node를 추가한다.

        2. 왼쪽 형제 Node에 Merge Delta Node를 추가해서 오른쪽 형제 Node를 참조하도록 만든다.

        3. 부모 Node에서 오른쪽 자식 Node의 포인터를 제거한다. (오른쪽 자식 Node는 Node Consolidation 시점까지 Merge Delta Node로 접근 가능하다.)

    여러 스레드가 동시에 SMO를 하려고 하면 충돌이 일어날 수 있다. 따라서 부모 Node에 Abort Delta를 먼저 삽입해두어 한번에 하나의 SMO만 처리되게 보장한다. SMO 이후 Abort Delta는 제거된다.

    참고로, Split Delta Node, Merge Delta Node는 남아있다가, 후술하는 Node Consolidation을 통해 정리된다.

- **Consolidation and Garbage Collection**

    Bw-Tree에서, Logical Node의 Update Chain이 너무 길어지면 이를 통합(*Cosolidation*)하고 메모리를 회수(*Garbage Collection*)해야 한다.

    Node Consolidation이 필요한 제일 큰 이유는 읽기 성능 때문이다. Update Chain이 길어질수록 Read 작업 시에 Delta를 다 따라가야 하니 느려진다. 이 통합 과정은 다음 순서로 이루어진다.

    1. Chain 길이가 기준값보다 길어지면 통합을 시작한다.

    2. Base Node에 모든 Delta Node를 적용하여 재구성한다.

    3. 이를 새로운 Base Node로 만들어 디스크의 새 위치에 저장한다.

    4. 매핑 테이블에서 Logical Node ID가 새 Base Node를 가리키도록 변경한다.

    통합이 끝나면 이전 Base Node와 Delta Node는 더 이상 사용되지 않기에, 메모리에서 지워야 가용 영역을 더 확보할 수 있다. 그러나 아직 예전 노드를 보고 있는 스레드가 존재할 수 있고, Bw-Tree 자체가 Latch를 사용하지 않기 때문에, 바로 메모리에서 삭제하면 문제가 생길 수 있다.

    Bw-Tree는 에포크 기반 교정 기법(*Epoch-Based Reclamation*)을 사용하여 각 스레드가 언제부터 읽기 시작했는지를 추적한다. 이는 다음 과정을 따라 이루어진다.

    1. 어떤 Chain이 Consolidation 과정을 통해 새로운 Node로 대체된다.

    2. 이전 Node들은 매핑 테이블에서 더 이상 접근이 불가능하다.

    3. 하지만 이전 Epoch 시간에 시작한 스레드는 아직 그 Node를 참조할 수 있다.

    4. 따라서 해당 스레드들이 모두 종료된 이후에만 메모리를 회수 가능하다.

### Cache-Oblivious B-Trees

전통적인 B-Tree는 Block 크기, Node 크기, Cache line 크기 등, 성능에 영향을 주는 하드웨어적 요소가 많다. 하드웨어 환경마다 이 값이 다 다르기 때문에, B-Tree를 하드웨어마다 최적화하는 것이 어렵다.

캐시 비인지형(Cache-Oblivious) B-Tree란, 캐시를 몰라도 효율적인(하드웨어 세부 정보를 몰라도 효율적인) B-Tree 구조이다. 알고리즘이나 자료구조가 하드웨어 환경에 의존하지 않고 자동으로 효율적인 메모리 계층 접근을 하도록 설계된다.

- **van Emde Boas Layout**

캐시 비인지형 B-Tree는 정적 B-Tree와 묶음 배열로 구성되며, 여기서 정적 B-Tree는 반 엠데 보아스 레이아웃(vEB 레이아웃)을 기반으로 생성한다.

vEB 레이아웃이란, B-Tree의 노드들을 메모리에 어떻게 배치할 것인지를 결정하는 레이아웃이다. 이 레이아웃은 캐시 미스를 최소화하기 위해, 자주 같이 쓰이는 노드들을 메모리 상에서 가까이 둔다. vEB 레이아웃은 재귀적 분할을 통해 구성되며, 그 방법은 다음과 같다.

1. 트리를 중간 깊이를 기준으로, 위쪽 반과 아래쪽 반으로 나눈다.

2. 아래쪽 트리도 재귀적으로 또 나눈다.

3. 트리의 하위 구조는 메모리 상에 연속된 블록으로 배치된다.

이론적으로는 좋아 보이나, 성능에 큰 영향을 주는 캐시와 블록을 완전히 무시하는 것은 불가능하다는 문제를 가진다.
