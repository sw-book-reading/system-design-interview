# Chpt6. 키-값 저장소 설계

#### CAP 정리
- Consistency
	- 클라이언트는 어떤 노드에 접속했느냐에 관계없이 언제나 같은 데이터를 보게 된다.
- Availability
	- 일부 노드에 장애가 발생하더라도 항상 응답을 받을 수 있다.
- Partition tolerance
	- partition : 두 노드 사이에 통신 장애가 발생한 것을 의미.
	- network partition 이 발생하더라도 시스템이 계속 동작하는 것
- 위 세가지를 동시에 만족하는 분산 시스템을 설계하는 것은 불가능
- 현실세계에서 network partition 은 일어날 수 밖에 없고, CAP 정리는 우리가 CP 혹은 AP 중 하나를 선택해야 함을 함축하고 있음.
	- 즉, consistency or availability 중 하나를 선택

#### 정족수 합의 프로토콜 (Quorum Consensus)
- N
	- 사본 개수
- W
	- 쓰기 연산에 대한 정족수. 쓰기 연산이 성공으로 간주되려면 적어도 W개의 서버로부터 쓰기 연산이 성공했다는 응답을 받아야 함
- R
	- 읽기 연산에 대한 정족수
- W,R,N 값을 정하는 것은 **응답 지연**과 **데이터 일관성** 사이의 타협점을 찾는 과정

#### 일관성 모델
- 데이터 일관성 수준을 결정
- strong consistency
	- 모든 읽기 연산은 가장 최근에 갱신된 결과를 반환
	- 달성하기 위한 방법 중 하나는 모든 사본에 현재 write 결과가 반영될 때까지 해당 데이터에 대한 read/write 를 금지하는 것.
		- 고가용성 시스템에는 적합하지 않음.
- weak consistency
	- 읽기 연산이 가장 최근에 갱신된 결과를 반환하지 못할 수도 있음.
- eventual consistency
	- weak consistency 의 한 형태로, 갱신 결과가 결국에는 모든 사본에 동기화됨


#### 비일관성 해소 기법 
- 배경
	- write 가 여러 노드에 병렬로 이루어지는 경우 노드 간 비일관된 데이터가 생길 수 있음.
- 벡터 시계
	- 데이터에 [서버, 버전] 순서쌍을 매다는 방법
	- 충돌을 감지할 수 있음.
		- 충돌을 해결하는 것은 보통 클라이언트 단에서 수행

![](attatchments/스크린샷%202023-01-15%20오전%2012.16.52.png)

#### 장애 감지
- 모든 노드 사이에  multicasting 채널 구축
	- 노드가 많을 경우 비효율적
- Gossip protocol
	- 각 노드는 membership(serverId, heartbeat counter) list 유지
	- 각 노드는 주기적으로 자신의 heartbeat counter 증가시킴
	- 각 노드는 무작위로 선정한 노드들에게 자신의 heartbeat counter 목록을 보냄
	- heartbeat counter 목록을 받은 노드는 membership list 를 최신값으로 갱신
	- 어떤 멤버의 heartbeat counter 가 지정된 시간 동안 갱신되지 않으면 해당 member 를 장애 상태인 것으로 간주


![](attatchments/스크린샷%202023-01-15%20오전%2012.33.23.png)

#### 일시적 장애 처리
- sloppy quorum (느슨한 정족수) 접근법
	- strict quorum 과 달리, 정족수를 충족하지 못할 때 쓰기 연산시 W개 서버, 읽기 연산시 R개 서버를 hash ring 에서 고른다.
- hinted handoff 기법
	- 장애 상태인 서버로 가는 요청을 다른 서버가 맡아서 처리하고, 해당 서버가 복구 되었을 때 그동안 발생한 변경사항을 일괄 반영하는 것

#### 영구 장애 처리
- anti-entropy 프로토콜
	- 사본들을 비교하여 데이터를 최신 버전으로 갱신하는 과정
- Merkle tree
	- 각 노드에 그 자식 노드들에 보관된 값의 해시 또는 자식 노드들의 레이블로부터 계산된 해시값을 레이블로 붙여두는 트리
	- 다른 데이터를 갖는 버킷을 빠르게 찾을 수 있다.
		- 해당 버킷들만 동기화하면 된다.

![](attatchments/스크린샷%202023-01-15%20오전%2012.42.57.png)
![](attatchments/스크린샷%202023-01-15%20오전%2012.43.04.png)

Q. 109p. figure 6-15 버킷별로 해시값을 계산한다는 게 어떤 의미일까?


#### 시스템 아키텍처 다이어그램
![](attatchments/스크린샷%202023-01-15%20오전%2012.46.26.png)
- Q. 111p. '모든 노드가 같은 책임을 지므로 Single Point Of Failure 는 존재하지 않는다' 고 되어 있지만 coordinator 에 대한 fail over 는 어떻게 할까..?

#### 쓰기 연산
![](attatchments/스크린샷%202023-01-15%20오전%2012.48.06.png)
- Q. SSTable?

#### 읽기 연산
![](attatchments/스크린샷%202023-01-15%20오전%2012.48.17.png)
- Q. Bloom filter?


---
# Chpt7. 분산 시스템을 위한 유일 ID 생성기 설계

- multi-master repliction
- Universally Unique Identifier
- Ticket server
- Twitter snowflake

### multi-master repliction
- DB 의 `auto_increment` 기능을 활용하되 다음 ID 값을 구할 때 1이 아닌 서버 개수 만큼 값을 증가시킴
#### 단점
- 여러 데이터 센터에 걸쳐 규모 확장 어려움
- 서버가 추가하거나 삭제되었을 때 문제 발생
### Universally Unique Identifier
- UUID : 정보를 유일하게 식별하기 위한 128 bit 짜리 수
	- 충돌 가능성이 매우 낮음
#### 장점
- 서버 사이의 조율이 필요 없어서 동기화 문제에서 자유로움
- 규모 확장이 쉬움
#### 단점
- ID 가 128 bit 로 길다.
- ID 를 시간 순으로 정렬할 수 없음
- ID 에 숫자가 아닌 값이 포함될 수 있음
### Ticket server
- `auto_increment` 기능을 갖춘 데이터베이스 서버 한대 (ticket server) 를 중앙 집중형으로 사용하는 것
#### 장점
- 유일성이 보장되는 숫자로만 구성된 ID를 쉽게 생성
- 구현이 쉽고 중소 규모 app 에 적합
#### 단점
- ticket server 가 Single Point Of Failure 가 된다.
### Twitter snowflake
![](attatchments/스크린샷%202023-01-15%20오전%2010.01.16.png)
- sign bit
	- 음수, 양수 구분
- timestamp
	- 기원 시간 이후로 몇 ms 가 경과했는지 나타내는 값
	- 시간 흐름에 따라 점점 더 큰 값을 갖게 되므로 ID 를 시간 순으로 정렬할 수 있게 함
	- 2^41 == 대략 69년. 69년이 지나면 기원 시간을 바꾸거나 ID 체계를 다른 것으로 migration 해야 함
- datacenter ID
	- 32 개의 datacenter 지원
- machine ID
	- 서버 ID. 데이터 센터당 32개 서버 사용 가능
- sequence number
	- 1 ms 당 2^12 개 만큼의 ID 를 생성할 수 있음
- application 성격에 따라 위 section 의 bit 수는 조정 가능. e.g. 동시성이 낮고 수명이 긴 app 이라면 seq number bit 수를 줄이고 timestamp bit 수를 늘린다.


---
# chpt 8. URL 단축기 설계

![](attatchments/스크린샷%202023-01-15%20오전%2010.12.07.png)
![](attatchments/스크린샷%202023-01-15%20오전%2010.12.26.png)
#### HTTP 301 vs 302 
- 301 Permaently Moved
	- 해당 HTTP 요청의 처리 책임이 **영구적으로** Location Header 에 반환된 URL 로 이전 되었음을 의미. 브라우저는 이 응답을 Cache 한다.
- 302 Found
	- 해당 HTTP 요청의 처리 책임이 **일시적으로** Location Header 가 지정하는 URL 에 의해 처리되어야 함을 의미. 따라서 클라이언트의 요청은 언제나 단축 URL 서버에 먼저 보내진 후 원래 URL 로 redirection 되어야 함

#### 데이터 모델
- <shortURL, longURL> hash table 에 값을 저장하는 것은 시스템 규모가 커지면 한계가 있음
- Relational DB 에 id, shortURL, longURL 을 저장하는 것을 권장

#### 해시 값 길이
- 요구사항에 따르면 해시 값은 3650 억개의 URL 을 구분할 수 있어야 함.
	- 해시 값으로는 62 개 문자가 올 수 있음.
	- 62^7 > 3650억 이므로 해시 값을 7자리로 설정

#### 해시 후 충돌 해소
![](attatchments/스크린샷%202023-01-15%20오전%2010.23.20.png)
- CRC32, MD5, SHA-1 과 같이 잘 알려진 해시 함수 활용
	- 위 계산 값에서 처음 7개 글자만 사용
	- 충돌 발생시 사전에 정한 문자열을 longURL 뒤에 덧붙여서 다시 해시 함수를 적용한다.
##### 단점
- 단축 URL 을 생성할 때마다 중복 검사를 위한 DB 조회 필요

#### base-62 변환
- longURL 을 입력받은 후 Unique ID 값 생성
- Unique ID 값을 62진법(입력 가능한 문자 종류 개수) 으로 치환

#### 장점
- 단축 URL 간 충돌이 발생하지 않음

#### 단점
- 단축 URL 의 길이가 가변적. ID 값이 커지면 단축 URL 길이도 길어짐
- 유일성 보장 ID 생성기 필요 (chpt 7)
- ID 가 1씩 증가한다고 가정하면 다음에 쓸 수 있는 단축 URL 이 무엇인지 쉽게 알 수 있어 보안상 취약


---
- vector clock
- Q. scheduler heartbeat 처리 코드?


