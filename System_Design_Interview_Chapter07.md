# 7장 - 분산 시스템을 위한 유일 ID 생성기 설계

- `auto_increment` 속성이 설정된 관계형 데이터 베이스의 기본 키는?
  - 데이터베이스 한 대로는 그 요구를 감당할 수 없음
  - 여러 데이터베이스 서버를 쓰는 경우에는 지연 시간을 낮추기가 무척 힘듦

## 1단계 : 문제 이해 및 설계 범위 확정

- 이번 문제 이해에서 만족해야 할 요구사항
  - ID는 유일해야 함
  - ID는 숫자로만 구성
  - ID는 64비트로 표현 가능해야 함
  - ID는 발급 날짜에 따라 정렬 가능해야 함
  - 초당 10,000개의 ID를 만들 수 있어야 함

## 2단계 : 개략적 설계안 제시 및 동의 구하기

### 다중 마스터 복제

- 데이터베이스의 auto_increment 기능을 활용한 것
  - 다만 다음 ID의 값은 k 만큼 증가
- 문제점
  - 여러 데이터 센터에 걸쳐 규모를 늘리기 어려움
  - ID의 유일성은 보장되겠지만 그 값이 시간 흐름에 맞추어 커지도록은 보장 X
  - 서버를 추가하거나 삭제할 때도 잘 동작하도록 만들기 어려움

### UUID

- 유일성이 보장되는 ID를 만드는 또 하나의 간단한 방법
- UUID는 컴퓨터 시스템에 저장되는 정보를 유일하게 식별하기 위한 128비트짜리 수
  - 충돌 가능성이 지극히 낮음
  - 서버 간 조율 없이 독립적으로 생성 가능
- 장점
  - UUID를 만드는 것은 단순
  - 서버 사이의 조율이 필요 없으므로 동기화 이슈 X
  - 각 서버가 자기가 쓸 ID를 알아서 만드는 구조이므로 규모 확장도 쉬움
- 단점
  - ID가 128비트로 길다
  - ID를 시간순으로 정렬 불가능
  - ID에 숫자가 아닌 값이 포함

### 티켓 서버

- auto_increment 기능을 갖춘 데이터베이스 서버인 티켓 서버를 중앙 집중형으로 하나만 사용하는 것
- 장점
  - 유일성이 보장되는 오직 숫자로만 구성된 ID를 쉽게 만들 수 있음
  - 구현하기 쉽고 중소 규모 애플리케이션에 적합
- 단점
  - 티켓 서버가 SPOF됨
  - SPOF를 막기 위해 티켓 서버를 여러 대 준비한다면 데이터 동기화 같은 새로운 문제 발생

### 트위터 스노플레이크 접근법

- 각개격파 전략을 적용
  - 생성해야 하는 ID의 구조를 여러 절로 분할하는 것
- 64비트 각개격파 전략 구조에서 각 절의 쓰임새
  - 사인 비트 : 1비트, 나중을 위해 유보 (음수와 양수 구별 등)
  - 타임스탬프 : 41비트, 기원시각 이후로 몇 밀리초 경과했는지를 볼 수 있음
  - 데이터센터 ID : 5비트, 2^5 = 32개 데이터센터 지원 가능
  - 서버 ID : 5비트를 할당, 데이터센터당 32개 서버 사용 가능
  - 일련번호 : 12비트 할당, 각 서버에서는 ID를 생성할 때마다 번호를 1씩 증가, 1밀리초가 경과할 때마다 0으로 초기화

## 3단계 : 상세 설계

- 타임스템프가 ID 구조에서 가장 중요한 41비트를 차지
  - ID를 시간순으로 정렬할 수 있게 해줌
  - 이진수 -> 십진수 -> epoch를 더함 -> 밀리초 UTC 시각으로 변환해서 타임스템프 값으로 변환 가능
  - 41비트는 대략 69년에 해당하므로 기원 시각을 현재에 가깝게 맞춰서 오버플로를 방지
- 일련번호
  - 12비트이므로 4096개의 값을 가질 수 있음
  - 어떤 서버가 같은 밀리초 동안 하나 이상의 ID를 만들어 낸 경우에만 0보다 큰 값을 갖게 됨

## 4단계 : 마무리

- 추가로 논의할 수 있는 것
  - 시계 동기화 : 하나의 서버가 여러 코어로 실행되거나 여러 서버가 물리적으로 독립된 장비에서 실행되는 경우 생길 수 있는 문제, NTP(Network Time Protocol)이 이 문제를 해결할 수 있는 보편적 수단
  - 각 절의 길이 최적화 : 애플리케이션에 따라 일련번호 절의 길이를 줄이고 타임스탬프 절의 길이를 늘이는 등의 최적화 가능
  - 고가용성 : ID 생성기는 필수 불가결 컴포넌트므로 아주 높은 가용성을 제공해야 함