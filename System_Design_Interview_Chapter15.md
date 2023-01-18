# 15장 - 구글 드라이브 설계

- 파일 저장 및 동기화 서비스
  - 문서, 사진, 비디오, 기타 파일을 클라우드에 보관
  - 어떤 단말에서도 이용 가능해야 함
  - 친구, 가족, 동료 들과 손쉽게 공유할 수 있어야 함

## 1단계 : 문제 이해 및 설계 범위 확정

- 가장 중요하게 지원해야 할 기능?
  - 파일 업로드/다운로드, 파일 동기화, 알림
- 모바일 앱, 웹 지원?
  - 둘 다 지원
- 파일 암호화는?
  - 지원해야 함
- 파일 크기 제한
  - 10GB 제한
- 사용자는 얼마나 되는지?
  - DAU 기준으로 천만 명
- 설계에 집중해야 하는 기능
  - 파일 추가 (drag-and-drop)
  - 파일 다운로드
  - 여러 단말에 동기화
  - 파일 갱신 이력 조회
  - 파일 공유
  - 파일이 편집되거나 삭제되거나 새롭게 공유되었을 때 알림 표시
- 논의하지 않을 기능
  - 구글 문서 편집 및 협업 기능
- 중요한 비-기능적 요구사항
  - 안정성
  - 빠른 동기화 속도
  - 네트워크 대역폭
  - 규모 확장성
  - 높은 가용성

### 개략적 추정치

- 가입 사용자는 오천만 명, 천만 명의 DAU 사용자
- 모든 사용자에게 10GB 무료 저장 공간 할당
- 매일 각 사용자가 평균 2개 파일 업로드 가정
- 읽기:쓰기 비율은 1:1
- 필요한 저장 공간 총량 = 5천만 사용자 x 10GB = 500 페타 바이트
- 업로드 API QPS = 1천만 사용자 x 2회 업로드/24시간/3600초 = 약 240
- 최대 QPS = QPS x 2 = 480

## 2단계 : 개략적 설계안 제시 및 동의 구하기

- 우선 서버 한 대로 구성을 시작
  - 파일을 올리고 다운로드하는 과정을 처리할 웹 서버
  - 메타데이터를 보관할 DB
  - 파일 저장소 시스템 (1TB)
- 아파치 웹 서버, MySQL DB, drive/ 경로의 디렉터리 준비
  - drive/ 디렉터리 아래 네임스페이스라 불리는 하위 디렉터리들을 둠
  - 각 네임스페이스 안에는 특정 사용자가 올린 파일이 보관

### API

- 파일 업로드 API
  - 단순 업로드 : 파일 크기가 작을 때 사용
  - 이어 올리기(resumable upload) : 파일 사이즈가 크고 네트워크 문제로 업로드가 중단될 가능성이 높다고 생각되면 사용
  - 이어 올리기 URL을 받기 위한 최초 요청 전송 -> 데이터를 업로드하고 업로드 상태 모니터링 -> 업로드 장애 발생 시 장애 발생 시점부터 업로드 재시작
- 파일 다운로드 API
  - 인자로 다운로드할 파일의 경로인 path 값을 받음
- 파일 갱신 히스토리 API
  - 인자로 갱신 히스토리를 가져올 파일의 경로인 path 값을 받음
  - 인자로 히스토리 길이의 최대치인 limit 값을 받음
- 모든 API는 사용자 인증과 HTTPS 프로토콜을 사용 (데이터 보호)

### 한 대 서버의 제약 극복

- 데이터를 샤딩 할 수도 있음 (user_id 기준 등등)
- 아마존 S3 사용 가능
  - 업계 최고 수준의 규모 확장성, 가용성, 보안, 성능
  - S3는 다중화 지원 (같은 지역 or 여러 지역)
  - S3 버킷은 파일 시스템의 폴더와도 같은 것
- 로드밸런서
  - 네트워크 트래픽 고르게 분산
  - 특정 웹서버에 장애 발생 시 해당 서버 우회
- 웹 서버
  - 로드밸런서 추가 후 더 많은 웹 서버를 손쉽게 추가 가능
- 메타데이터 데이터베이스
  - SPOF 회피
  - 다중화 및 샤딩 정책 적용
- 파일 저장소
  - 두 개 이상의 지역에 데이터 다중화

### 동기화 충돌

- 먼저 처리되는 변경은 성공한 것으로, 나중에 처리되는 변경은 충돌이 발생한 것으로 표기
- 오류가 발생한 시점엔 로컬 사본과 서버의 최신 버전 두 버전의 파일이 존재
  - 이 상태에서 사용자가 결정

### 개략적 설계안

- 사용자 단말
  - 사용자가 이용하는 웹 브라우저, 모바일 앱 등
- 블록 저장소 서버
  - 파일 블록을 클라우드 저장소에 업로드하는 서버
  - 블록 수준 저장소로도 불림
  - 파일을 여러 개의 블록으로 나눠 저장
  - 각 블록엔 고유한 해시값 할당 (메타데이터 DB에 저장)
- 클라우드 저장소
  - 파일은 블록 단위로 나눠져 클라우드 저장소에 보관
- 아카이빙 저장소
  - 오랫 동안 사용되지 않은 비활성 데이터를 저장하기 위한 컴퓨터 시스템
- 로드밸런서
  - 요청을 모든 API 서버에 고르게 분산
- API 서버
  - 파일 업로드 외에 거의 모든 것을 담당
  - 사용자 인증, 파일 메타데이터 갱신 등
- 메타데이터 데이터베이스
  - 사용자, 파일, 블록, 버전 등의 메타데이터 정보 관리
- 메타데이터 캐시
  - 성능을 높이기 위해 자주 쓰이는 메타데이터는 캐시
- 알림 서비스
  - 특정 이벤트 발생 시 클라이언트한테 알리는데 쓰이는 발생/구독 프로토콜 기반 시스템
- 오프라인 사용자 백업 큐
  - 클라이언트가 접속 시 파일의 최신 상태를 동기화 시켜줌

## 3단계 : 상세 설계

### 블록 저장소 서버

- 파일 업데이트 시 최적화 방법
  - 델타 동기화 : 파일 수정 시 수정이 일어난 블록만 동기화
  - 압축 : 블록 단위로 압축해 두어 데이터 크기를 많이 줄임 (gzip, bzip2 ...)
- 블록 저장소 서버의 역할
  - 블록 단위로 파일 나눔, 각 블록에 압축 알고리즘 적용, 암호화, 수정된 블록만 시스템으로 전송 ...
- 새 파일 추가 시 블록 저장소 동작 과정
  - 주어진 파일을 작은 블록들로 분할
  - 각 블록 압축
  - 암호화
  - 클라우드 저장소로 보냄

### 높은 일관성 요구사항

- 강한 일관성 모델을 기본으로 지원해야 함
  - 같은 파일이 단말이나 사용자에 따라 다르게 보이는 것 허용 X
- 메모리 캐시는 보통 최종 일관성 모델을 지원
- 강한 일관성을 지원하기 위해 다음이 필요
  - 캐시에 보관된 사본과 데이터베이스에 있는 원본이 일치
  - 데이터베이스에 보관된 원본에 변경이 발생하면 캐시에 있는 사본을 무효화
- NoSQL은 ACID를 기본으로 지원 X -> 동기화 로직 안에 프로그램 해야 함

### 메타데이터 데이터베이스

- user
  - 이름, 이메일, 프로파일 사진 등 사용자 기본 정보
- device
  - 단말 정보
  - push_id : 모바일 푸시 알림을 보내고 받기 위한 것
  - 한 사용자가 여러 대의 단말을 가질 수 있음
- namespace
  - 사용자의 루트 디렉터리 정보
- file
  - 파일의 최신 정보 보관
- file_version
  - 파일의 갱신 이력이 보관
  - 전부 읽기 전용 (갱신 이력 훼손 방지)
- block
  - 파일 블록에 대한 정보 보관

### 업로드 절차

- 파일 메타데이터 추가
  - 클라이언트 1이 새 파일의 메타데이터 추가 요청 전송
  - 새 파일의 메타데이터를 DB에 저장하고 업로드 상태를 대기 중으로 변경
  - 새 파일이 추가되었음을 알림 서비스에 통지
  - 알림 서비스는 관련된 클라이언트에게 알림
- 파일을 클라우드 저장소에 업로드
  - 클라이언트 1이 파일을 블록 저장소 서버에 업로드
  - 블록 저장소 서버는 파일을 블록 단위로 쪼개고 압축, 암호화 후 클라우드 저장소에 전송
  - 업로드가 끝나면 스토리지는 완료 콜백을 호출
  - 메타데이터 DB에 해당 파일 상태를 완료로 변경
  - 알림 서비스에 통지
  - 관련된 클라이언트에게 알림
- 파일 수정도 이 흐름과 유사

### 다운로드 절차

- 다른 클라이언트가 파일을 편집하거나 추가했다는 사실 감지 방법
  - 변경 시 다른 클라이언트에게 새 버전을 끌어가야 한다고 알림
  - 다른 클라이언트가 연결된 상태가 아닐 시 데이터를 캐시에 보관 (접속하면 새 버전 가져감)
- 과정
  - 알림 서버가 다른 클라이언트에게 알림
  - 클라이언트는 새 메타데이터 요청
  - API 서버가 메타데이터 DB에서 새 메타데이터를 가져온 후 클라이언트에게 반환
  - 클라이언트는 블록 저장소 서버를 거쳐 클라우드 저장소에서 블록 다운로드
  - 클라이언트는 전송된 블록으로 파일 재구성

### 알림 서비스

- 충돌 가능성을 줄이기 위해 사용
- 롱 폴링, 웹 소켓 방식이 있음
  - 알림 서비스와 양방향 통신이 필요하지 않을 때 롱 폴링이 적합
- 롱 폴링은 특정 파일에 대한 변경 감지 시 연결을 끊음 (클라이언트-알림 서버 연결)
  - 클라이언트는 메타데이터 서버에서 최신 내역을 다운로드해야 함

### 저장소 공간 절약

- 백업 비용을 절감할 수 있는 방법
  - 중복 제거 (해시 값을 비교하여 중복된 파일 블록을 제거)
  - 지능적 백업 전략 (한도 설정, 중요한 버전만 보관)
  - 자주 쓰이지 않는 데이터는 아카이빙 저장소로 옮김 (S3 글래시어 등)

### 장애 처리

- 로드밸런서 장애
  - 부 로드밸런서 활성화
  - 로드밸런서끼리는 박동 신호로 상태 모니터링
- 블록 저장소 서버 장애
  - 다른 서버가 작업을 이어받아야 함
- 클라우드 저장소 장애
  - S3 버킷은 여러 지역에 다중화 해야 함
- API 서버 장애
  - API 서버들은 무상태
  - 로드밸런서들이 장애가 발생한 API는 트래픽을 보내지 않고 장애 서버 격리
- 메타데이터 캐시 장애
  - 메타데이터 캐시 서버도 다중화 할 것
- 메타데이터 데이터베이스 장애
  - 주 데이터베이스 서버 장애 : 부 데이터베이스 하나를 주 서버로 바꾼 뒤 다른 부 데이터베이스 서버 추가
  - 부 데이터베이스 서버 장애 : 다른 부 데이터베이스 서버가 읽기 연산 처리하도록 하고 서버는 새것으로 교체
- 알림 서비스 장애
  - 서버에 장애 발생 시 롱 폴링 연결을 다시 만들어야 함
- 오프라인 사용자 백업 큐 장애
  - 이 큐 또한 다중화 할 것

## 4단계 : 마무리

- 파일의 메타데이터를 관리하는 부분, 파일 동기화를 처리하는 부분으로 설계안이 나눠졌었음