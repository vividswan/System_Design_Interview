# 8장 - URL 단축기 설계

## 1단계 : 문제 이해 및 설계 범위 확정

- URL 단축기가 어떻게 동작해야 하는지?
  - 긴 주소가 입력으로 주어졌을 때 단축 URL을 결과로 제공해야 함
  - 단축 URL에 접속하면 원래 URL로 갈 수도 있어야 함
- 트래픽 규모는?
  - 매일 1억(100million) 개의 단축 URL
- 단축 URL의 길이는?
  - 짧으면 짧을수록 좋음
- 단축 URL에 포함될 문자에 제한은?
  - 숫자(0부터 9까지)와 영문자(a부터 z, A부터 Z까지)만 사용 가능
- 단축된 URL을 시스템에서 지우거나 갱신할 수 있는지?
  - 시스템을 단순화하기 위해 삭제나 갱신은 할 수 없다고 가정

## 2단계 : 개략적 설계안 제시 및 동의 구하기

### API 엔드포인트

- 클라이언트는 서버가 제공하는 API 엔드포인트를 위해 서버와 통신
- URL 단축기는 기본적으로 두 개의 엔드포인트를 필요
  - URL 단축용 엔드포인트 : 새 단축 URL을 생성하고자 하는 클라이언트는 이 엔드포인트에 단축할 URL을 인자로 실어서 POST 요청을 보내야 함
  - URL 리디렉션용 엔드포인트 : 단축 URL에 대해서 HTTP 요청이 오면 원래 URL로 보내주기 위한 용도의 엔드포인트

### URL 리디렉션

- 단축 URL을 받은 서버는 그 URL을 원래 URL로 바꾸어서 301 응답의 Location 헤더에 넣어 반환
- 301 응답과 302 응답의 차이
  - 둘 다 리디렉션 응답이긴 하지만 차이가 존재
- 301 Permanently Moved
  - 해당 URL에 대한 HTTP 요청의 처리 책임이 영구적으로 Location 헤더에 반환된 URL로 이전되었다는 응답
  - 영구적 이전이므로 브라우저는 이 응답을 캐시(cache)
  - 추후 같은 단축 URL에 대한 요청 시 브라우저는 캐시 된 원래 URL로 요청을 보냄
  - 서버 부하를 줄이는 것이 중요할 시 301 (첫 번째 요청만 단축 URL 서버로 전송될 것이기 때문에)
- 302 Found
  - 주어진 URL로의 요청이 '일시적으로' Location 헤더가 지정하는 URL에 의해 처리되어야 함
  - 클라이언트의 요청은 언제나 단축 URL 서버에 먼저 보내진 후에 원래 URL로 리디렉션 되어야 함
  - 트래픽 분석이 중요할 때는 302 Found
- 가장 직관적인 구현 방법은 해시 테이블

### URL 단축

- URL 단축 값으로 대응시킬 해시 함수가 만족해야 하는 요구사항들
  - 입력으로 주어진 긴 URL이 다른 값이면 해시 값도 달라야 함
  - 계산된 해시 값은 원래 입력으로 주어졌던 긴 URL로 복원할 수 있어야 함

## 3단계 : 상세 설계

### 데이터 모델

- 메모리는 유한하고 비싸기 때문에 <단축 URL, 원래 URL>의 순서쌍을 관계형 데이터베이스에 저장하는 것이 더 나은 방법
- id, shortURL, longURL 세 개의 칼럼

### 해시 함수

- 원래 URL을 단축 URL로 변환하는데 쓰임
  - 편의상 단축 URL 값을 hashValue로 지칭
- 해시 값 길이
  - hashValue는 [0-9, a-z, A-Z]의 문자들로 구성 (개수는 10 + 26 + 26 = 62개)
  - hashValue의 길이를 정하기 위해선 62^n >= 3650억 인 n의 최솟값을 찾아야 함
  - n = 7 잀시 3.5조 개의 URL을 만들 수 있음
  - 해시 함수 구현에 쓰일 두 가지 기술 : 해시 후 충돌 해소, base-62 변환
- 해시 후 충돌 해소
  - 잘 알려진 해시 함수들도 7보다는 길다.
  - 해시 값에서 처음 7개 글자만 이용할 시 충돌할 확률이 높아짐
  - 충돌이 실제로 발생 시 충돌이 해소될 때까지 사전에 정한 문자열을 해시값에 덧붙임
  - 이 방법은 한 번 이상 데이터베이스를 질의해야 하므로 오버헤드가 큼
  - 데이터베이스 대신 블룸 필터를 사용하면 성능을 높일 수 있음 (어떤 집합에 특정 원소가 있는지 검사할 수 있도록 하는 확률론에 기초한 공간 효율이 좋은 기술)
- base-62 변환
  - 진법 변환은 URL 단축기를 구현할 때 흔히 사용되는 접근법 중 하나
  - hashValue에 사용할 수 있는 문자 개수가 62개이므로 62진법 사용
- 두 접근법 비교
  - 해시 후 충돌 해소 전략 : 단축 URL 길이가 고정, 유일성이 보장되는 ID 생성기 필요 X, 충돌이 가능해서 전략 필요, 다음에 쓸 수 있는 URL을 알아내는 것이 불가능
  - base-62 변환 : 단축 URL의 길이가 가변적(ID 값에 비례), 유일성 보장 ID 생성기가 필요, ID 유일성이 보장된 후에야 적용하므로 충돌 아예 불가능, 보안상 문제 가능

### URL 단축키 상세 설계

- URL 단축기는 시스템의 핵심 컴포넌트
  - 처리 흐름이 논리적으로 단순해야 하고 기능적으로는 언제나 동작하는 상태로 유지되어야 함
- 흐름
  - 입력으로 긴 URL을 받음
  - DB에 해당 URL이 있는지 검사
  - 있다면 단축 URL을 클라이언트에게 반환
  - 없다면 유일한 ID 생성
  - 62진법 변환을 적용하여 ID를 단축 URL로 만듦
  - ID, 단축 URL, 원래 URL로 세 데이터베이스 레코드를 만든 후 단축 URL을 클라이언트에게 전달
- ID 생성기
  - 주된 용도는 단축 URL을 만들 때 사용할 ID를 만드는 것
  - ID는 전역적 유일성이 보장되어야 함

### URL 리디렉션 상세 설계

- 쓰기보다 읽기를 더 자주 하는 시스템이므로 <단축 URL, 원래 URL>의 쌍을 캐시에 저장하여 성능을 높일 수 있음
- 로드밸런스의 동작 흐름
  - 사용자가 단축 URL을 클릭
  - 로드밸런서가 해당 클릭으로 발생한 요청을 웹 서버에 전달
  - 단축 URL이 이미 캐시에 있는 경우엔 원래 URL을 바로 꺼내서 전달
  - 캐시에 해당 단축 URL이 없는 경우엔 데이터베이스에서 꺼냄 (데이터베이스에 없다면 잘못된 단축 URL)
  - 데이터베이스에서 꺼낸 URL을 캐시에 넣은 후 사용자에게 반환

## 4단계 : 마무리

- 처리율 제한 장치
  - IP 주소를 비롯한 필터링 규칙들을 이용해 요청을 걸러낼 수 있음
- 웹 서버의 규모 확장
  - 본 설계에 포함된 웹 계층은 무상태 계층
  - 웹 서버를 자유로이 증설하거나 삭제 가능
- 데이터베이스의 규모 확장
  - 데이터베이스 다중화, 샤딩 등
- 데이터 분석 솔루션
  - URL 단축기에 데이터 분석 솔루션을 통합해 두면 어떤 링크를 얼마나 많은 사용자가 클릭했는지, 언제 클릭했는지 등 파악 가능
- 가용성, 데이터 일관성, 안정성
  - 대규모 시스템이 성공적으로 운영되기 위해서 반드시 갖추어야 할 속성들