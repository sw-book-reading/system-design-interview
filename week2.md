# chpt4 처리율 제한 장치의 설계

#### 처리율 제한 장치 (rate limiter)
- 서비스가 보내는 트래픽의 처리율 제어
- 어디에 둘 것인가?
	- 클라이언트 요청은 쉽게 위변조가 가능하므로 추천 X
	- 서버 측에 두는 것이 좋다
		- API 서버
		- 미들웨어

#### 처리율 제한 알고리즘
- 토큰 버킷 (token bucket)
	- 장점
	- 단점
- 누출 버킷 (leaky bucket)
	- 장점
	- 단점
- 고정 윈도 카운터 (fixed window counter)
	- 장점
	- 단점
- 이동 윈도 로그 (sliding window log)
	- 장점
	- 단점
- 이동 윈도 카운터 (sliding window counter)
	- 장점
	- 단점

#### 처리율 한도 초과 트래픽 처리
- HTTP 헤더 활용해서 응답
	- e.g.
		- `X-Ratelimit-Remaining`
		- `X-Ratelimit-Limit`
		- `X-Ratelimit-Retry-After`
		- 사용자가 너무 많은 응답을 보내면 `429 too many requests` 오류를 `X-Ratelimit-Retry-After` 헤더와 함께 반환

#### 상세 설계
![](attatchments/스크린샷%202023-01-08%20오후%2010.19.34.png)

#### 분산 환경에서의 처리율 제한 장치 구현
- 풀어야할 과제
	- race condition
		- 해결책
			- lock
			- Lua script
			- sorted set (Redis 자료구조)
	- synchronization
		- 해결책
			- sticky session
				- 추천X. 규모 면에서 확장X, 유연성 X
			- 중앙 집중형 데이터 저장소 사용 e.g. Redis

#### 모니터링
- 처리율 제한 규칙이 효과적인지?
	- 너무 빡빡하게 설정하면 많은 유효 요청이 처리되지 못하고 버려질 것
- 처리율 제한 알고리즘이 효과적인지?
	- 깜짝 세일과 같이 트래픽이 급증할 때는 이런 트래픽 패턴을 잘 처리할 수 있도록 알고리즘 변경하는 것을 고려.
 

- Q. 실무에서 처리율 제한 장치는 어디에 설치되어 있는지? 미들웨어? api server?

--- 

# chpt5 안정 해시 설계
#### mod 방식 key 분배 문제점
- 서버 개수가 줄어들거나 늘어날 때 문제 발생
	- 1. key 가 불균등하게 분배됨. 즉, 특정 서버에 key 가 몰리게 됨.
	- 2. 특정 key 에 대한 request 가 이전과 다른 서버로 전송됨. cache miss 다량 발생.

![](attatchments/스크린샷%202023-01-08%20오전%2010.17.19.png)

![](attatchments/스크린샷%202023-01-08%20오전%2010.17.30.png)


#### consistent hashing
- hash space 의 양 끝단을 붙여서 hash ring 을 만듬.
- key 값으로부터 시계방향으로 이동하면서 가장 먼저 만나게 되는 server 에 key 값을 매칭. 
![](attatchments/스크린샷%202023-01-08%20오전%2010.33.19.png)


#### basic approach 문제점
- 1. 서버가 추가되거나 제거될 때 남은 서버들이 담당하는 영역이 불균등 (5-10)
- 2. key 가 불균등하게 분배되면 특정 서버에 key 가 몰릴 수 있음. (5-11)
![](attatchments/스크린샷%202023-01-08%20오전%2010.28.57.png)

![](attatchments/스크린샷%202023-01-08%20오전%2010.29.20.png)





#### virtual node
![](attatchments/스크린샷%202023-01-08%20오전%2010.26.47.png)

- s0, s1 서버를 참조하는 가상의 서버를 배치
- virtual node 개수를 늘릴수록 key 는 각 서버에 균일하게 분배. 즉, deviation 이 낮아짐
	- but virtual node 개수를 늘릴수록 virtual node data 를 위한 space 필요. (trade off)


#### consistent hashing 이점
- 서버가 추가되거나 제거될 때 재분배해야 하는 key 를 최소화
- data 가 비교적 균일하게 분배되어 있어 scale out 에 유리


- Q. Consistent Hashing 이용해서 Hotspot key problem 해결 방법?