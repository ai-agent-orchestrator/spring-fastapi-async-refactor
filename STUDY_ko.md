# 비동기 리팩토링 교과서 (초보자용)

> 코딩 초보를 위한 설명서. 내 법률 상담 서비스를 "기다리는 동안 자리를 비우는" 구조로 바꾸는 과정을 식당 비유와 그림으로 따라갑니다.
> 설계 문서(영문): [README.md](README.md)

## 1장 · 서버는 대부분의 시간을 "기다리며" 보낸다

상담 요청 하나를 처리하는 시간 대부분은 계산이 아니라 기다림입니다. LLM이 답을 만드는 2–10초 동안 서버는 사실상 아무것도 안 합니다. 리팩토링의 핵심 질문은 "그 기다리는 시간 동안 무엇을 붙잡고 있느냐"입니다.

```mermaid
pie showData
  title 상담 1건에서 시간이 가는 곳 (예시)
  "LLM 응답 기다림" : 2500
  "DB·네트워크 기다림" : 80
  "실제 계산 (가드·마스킹·검색)" : 60
```

> **확인 문제** CPU를 더 빠른 것으로 바꾸면 상담 처리량이 크게 늘까요?
>
> <details><summary>답 보기</summary>거의 늘지 않습니다. 시간 대부분이 계산이 아니라 기다림이기 때문입니다. 기다리는 방법을 바꿔야 합니다.</details>

## 2장 · 스레드 = 웨이터, 지금은 웨이터가 테이블 옆에 서서 기다린다

지금 Python 서버는 웨이터(스레드)가 약 40명입니다. 웨이터는 주문(LLM 호출)을 넣고 요리가 나올 때까지 그 테이블 옆에 서 있습니다. 41번째 손님은 홀이 비어 있어도 기다려야 합니다.

```mermaid
flowchart LR
  subgraph 홀[식당 · 웨이터 40명]
    W1[웨이터 1] -->|서서 기다림| T1[테이블 1 · 요리 대기 3초]
    W2[웨이터 2] -->|서서 기다림| T2[테이블 2 · 요리 대기 3초]
    W40[웨이터 40] -->|서서 기다림| T40[테이블 40]
  end
  N[손님 41] -.->|웨이터가 없어 입구에서 대기| 홀
```

**지금 코드 (일반 def)**

```python
@router.post("/screenings")
def create_screening(situation, ctx, s):
    r = s.screening.run(...)   # 안에서 LLM invoke
    return {...}               # 끝날 때까지 스레드 점유
```

**FastAPI가 하는 일**

```python
# def 함수는 스레드 풀(약 40개)에서 실행
# → 동시 처리 한도 ≈ 스레드 수
```

## 3장 · 비동기(async/await) = 주문 넣고 다른 테이블로 가는 웨이터

`await`는 "여기서 기다릴 테니, 그동안 다른 일을 해도 돼"라는 표시입니다. 웨이터 한 명(이벤트 루프)이 수십 테이블을 동시에 맡을 수 있습니다.

```mermaid
sequenceDiagram
  participant W as 웨이터 1명 (이벤트 루프)
  participant A as 테이블 A
  participant B as 테이블 B
  participant K as 주방 (LLM)
  W->>K: A 주문 넣기 (await)
  W->>B: B 주문 받기
  W->>K: B 주문 넣기 (await)
  K-->>W: A 요리 완성
  W->>A: A에 서빙
  K-->>W: B 요리 완성
  W->>B: B에 서빙
```

**바꾼 코드 (async def)**

```python
@router.post("/screenings")
async def create_screening(situation, ctx, s):
    r = await s.screening.arun(...)
    return {...}
```

**LLM 호출도 바꿔야 효과**

```python
# 전: msg = chat_model.invoke(messages)
msg = await chat_model.ainvoke(messages)
```

> **확인 문제** 함수만 `async def`로 바꾸고 안에서 여전히 `invoke`(동기)를 부르면 어떻게 될까요?
>
> <details><summary>답 보기</summary>더 나빠집니다. async 함수는 이벤트 루프 위에서 돌기 때문에, 안의 동기 호출이 루프 전체를 멈춰 세웁니다. 그 웨이터가 맡은 모든 손님이 같이 기다리게 됩니다(7장).</details>

## 4장 · 가상 스레드(Java 21) = 기다리는 동안 투명해지는 웨이터

Java의 가상 스레드는 코드를 "서서 기다리는" 쉬운 모양 그대로 두는데, Java가 기다리는 순간을 알아채고 실제 일꾼(캐리어 스레드)을 다른 요청에 빌려줍니다. 비동기의 효과를 코드 변경 거의 없이 얻는 방법입니다.

```mermaid
flowchart LR
  subgraph JVM[JVM]
    V1[가상 스레드 1<br/>Python 응답 대기] -.->|기다리는 동안 내려옴| CS
    V2[가상 스레드 2<br/>계산 중] --> C1[캐리어 스레드 A]
    V3[가상 스레드 3<br/>DB 대기] -.->|내려옴| CS
    V4[가상 스레드 4<br/>계산 중] --> C2[캐리어 스레드 B]
    CS((주차장<br/>메모리 몇 KB))
  end
```

**application.yml**

```
spring:
  threads:
    virtual:
      enabled: true
```

**주의 (JDK 21)**

```
// synchronized 안에서 I/O를 하면
// 가상 스레드가 캐리어에 "고정(pinning)"됨
// → ReentrantLock 사용
```

<table>
      <thead><tr><th></th><th>Python async/await</th><th>Java 가상 스레드</th></tr></thead>
      <tbody>
        <tr><td>코드 모양</td><td>바뀜 (<code>async</code>, <code>await</code> 표시)</td><td>거의 그대로 (동기 코드)</td></tr>
        <tr><td>기다림 처리</td><td>개발자가 <code>await</code>로 표시</td><td>JVM이 자동으로 알아챔</td></tr>
        <tr><td>실수하면</td><td>동기 호출 하나가 루프 전체를 멈춤</td><td>synchronized + I/O면 효과가 사라짐</td></tr>
      </tbody>
    </table>

## 5장 · 지금 구조 (v1-sync): 어디서 막히나

```mermaid
flowchart TB
  C[손님] --> S[Spring<br/>일반 스레드 약 200]
  S --> P[FastAPI<br/>def 함수]
  P --> TP[스레드 풀 약 40]
  TP --> L[LLM 대기 2~10초<br/>스레드 점유]
  TP --> DB[(SQLite 연결 1개<br/>잠금으로 줄 세움)]
  X[41번째 요청] -.->|대기| TP
```

그래도 지금 설계에서 잘한 점은 그대로 지킵니다: 속도 제한을 맨 앞에, 싼 검사를 먼저, LLM은 정확히 1번, 실패하면 크레딧 돌려주기.

## 6장 · 새 구조 (v2-async): 한 장으로 보기

```mermaid
flowchart TB
  C[손님] -->|상담 제출| S1
  subgraph SP[Spring · 가상 스레드]
    S1[상담 저장] --> S2[202 접수 완료 + 작업 번호]
    S2 --> S3[보호 장치<br/>동시 호출 한도 · 차단기 · 시간 제한]
  end
  S3 -->|내부 토큰 + 같은 작업 번호| P1
  subgraph PY[FastAPI · 이벤트 루프]
    P1[async def] --> P2[가드 · 마스킹<br/>빠른 계산]
    P2 --> P3[판례 검색<br/>to_thread로 옆 작업대]
    P3 --> P4[크레딧 예약]
    P4 --> P5[await LLM<br/>동시 호출 한도 · 시간 제한]
    P5 --> P6[정산 또는 돌려주기]
  end
  P6 --> S4[결과 저장]
  S4 -->|SSE 알림| C
```

## 7장 · 이벤트 루프에서 절대 하면 안 되는 것: 막는 호출

웨이터가 한 명뿐이라, 그 웨이터가 한 테이블에서 멍하니 서 있으면 식당 전체가 멈춥니다. 오래 걸리는 계산은 "옆 작업대"(`asyncio.to_thread`)로 보냅니다.

```mermaid
flowchart LR
  Q{이 작업은?} -->|네트워크·DB 기다림| A[await 비동기 함수]
  Q -->|짧은 계산<br/>가드·마스킹| B[루프에서 그냥 실행]
  Q -->|긴 계산·동기 라이브러리<br/>벡터 검색·PDF| C[await asyncio.to_thread]
  Q -->|time.sleep · requests.get · 동기 DB| D[금지: 루프 전체 정지]
```

**나쁜 예**

```python
async def search(q):
    return store.search(q)   # 동기 + 무거움
                             # → 루프 정지
```

**좋은 예**

```python
async def search(q):
    return await asyncio.to_thread(store.search, q)
```

## 8장 · 돈을 잃지 않는 법: 취소돼도 크레딧은 돌려준다

비동기에서는 요청이 중간에 "취소"될 수 있습니다(시간 초과, 손님이 나감). 그래서 크레딧 돌려주기를 `finally`에 둡니다. 어떤 길로 끝나도 반드시 실행되는 자리입니다.

```mermaid
stateDiagram-v2
  [*] --> 예약
  예약 --> LLM대기
  LLM대기 --> 정산: 답 받음
  LLM대기 --> 돌려주기: 시간 초과
  LLM대기 --> 돌려주기: 취소됨
  LLM대기 --> 돌려주기: 제공자 오류
  정산 --> [*]
  돌려주기 --> [*]
```

```python
rid = await wallet.reserve(tenant, 1)
settled = False
try:
    async with asyncio.timeout(30):
        msg = await chat_model.ainvoke(messages)
    await wallet.settle(tenant, rid, credits, usd)
    settled = True
finally:
    if not settled:
        await wallet.release(tenant, rid)   # 취소·오류·시간 초과 모두 여기로
```

## 9장 · 가상 스레드만으로는 부족한 이유: 보호 장치

가상 스레드는 Spring의 "웨이터 수 한도"만 없앱니다. 주방(LLM)과 창고(DB)의 크기는 그대로입니다. 그래서 문 앞에 "동시에 주방에 넣는 주문 수"를 제한하는 장치(벌크헤드)를 둡니다.

```mermaid
flowchart TB
  V[가상 스레드 1만 개] --> B{동시 호출 한도<br/>남은 자리?}
  B -->|없음| F[빠른 대체 응답<br/>AI 검토 대기 중]
  B -->|있음| CB{차단기<br/>Python 건강?}
  CB -->|열림| F
  CB -->|닫힘| CALL[호출 · 시간 제한]
  CALL -->|시간 초과| R{한 번 더?<br/>같은 작업 번호}
  R -->|예| CALL
  R -->|아니오| F
  CALL -->|성공| OK[결과 저장]
```

"같은 작업 번호(Idempotency-Key)"로 다시 보내면, Python은 이미 처리한 요청인지 알아보고 두 번 실행(두 번 과금)하지 않습니다.

## 10장 · 202와 작업 번호: 손님을 기다리게 하지 않는 법

```mermaid
sequenceDiagram
  participant C as 손님 화면
  participant S as Spring
  participant P as Python
  C->>S: 상담 제출
  S-->>C: 202 접수 완료 (작업 번호 7)
  Note over C: "AI 검토 중" 표시
  S->>P: 스크리닝 (작업 번호 7)
  P-->>S: 리포트
  S-->>C: SSE "작업 7 완료"
  C->>S: 리포트 보기
```

## 11장 · 리팩토링 단계: 한 번에 바꾸지 않는다

```mermaid
flowchart LR
  A[0 원본 보존<br/>v1-sync 태그<br/>속도 측정] --> B[1 계약 테스트<br/>바깥 모양 고정]
  B --> C[2 Python 비동기<br/>ainvoke · to_thread<br/>한도 · 취소 안전]
  C --> D[3 비동기 DB<br/>저장소 인터페이스 뒤에서 교체]
  D --> E[4 Spring<br/>가상 스레드 · 보호 장치 · 202]
  E --> F[5 다시 측정<br/>전후 비교 기록]
```

> 각 단계가 끝날 때마다 전체 테스트가 통과해야 다음으로 갑니다. 원본(v1-sync)은 언제든 다시 실행해서 비교할 수 있게 남겨 둡니다.

## 12장 · 정말 빨라졌는지 재는 법

진짜 LLM으로 재면 돈도 들고, LLM 속도에 따라 결과가 흔들립니다. 그래서 "항상 2초 걸리는 가짜 LLM"을 두고, 같은 조건에서 v1과 v2를 비교합니다.

```mermaid
flowchart LR
  G[부하 생성기<br/>동시 10 · 50 · 200] --> V1[v1-sync]
  G --> V2[v2-async]
  V1 & V2 --> FK[가짜 LLM<br/>항상 2초 · 비용 0]
  V1 & V2 --> M[기록<br/>중앙값 · 95% 응답시간<br/>초당 처리량 · 메모리 · 오류]
  M --> R[RESULTS.md 비교표]
```

> **확인 문제** 동시 200명일 때 v1의 95% 응답시간이 크게 늘어나는 이유는?
>
> <details><summary>답 보기</summary>스레드 40개가 모두 LLM을 기다리며 잡혀 있어서, 나머지 요청은 줄을 서기 때문입니다. 2초짜리 작업이 줄 선 시간만큼 길어집니다.</details>

## 13장 · 용어 정리

<table>
      <thead><tr><th>용어</th><th>쉬운 뜻</th><th>이 프로젝트에서</th></tr></thead>
      <tbody>
        <tr><td>스레드</td><td>일을 처리하는 웨이터</td><td>지금 Python은 약 40명</td></tr>
        <tr><td>이벤트 루프</td><td>여러 테이블을 오가는 웨이터 한 명</td><td>uvicorn 작업자마다 1개</td></tr>
        <tr><td>await</td><td>"여기서 기다릴게, 그동안 딴 일 해"</td><td>LLM·DB·HTTP 호출 앞</td></tr>
        <tr><td>to_thread</td><td>무거운 일을 옆 작업대로</td><td>벡터 검색</td></tr>
        <tr><td>가상 스레드</td><td>기다리면 투명해지는 웨이터</td><td>Spring 요청 처리</td></tr>
        <tr><td>세마포어 · 벌크헤드</td><td>동시에 들어갈 수 있는 인원 제한</td><td>LLM 동시 호출, Spring→Python 호출</td></tr>
        <tr><td>서킷 브레이커</td><td>고장 난 곳으로 계속 보내지 않는 차단기</td><td>Python이 느리면 대체 응답</td></tr>
        <tr><td>Idempotency-Key</td><td>같은 주문 번호면 한 번만 처리</td><td>재시도해도 두 번 과금 없음</td></tr>
        <tr><td>p95</td><td>100명 중 95번째로 느린 사람의 대기 시간</td><td>측정표의 핵심 숫자</td></tr>
      </tbody>
    </table>
