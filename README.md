# 포트폴리오

| 프로젝트 | 설명 | 형태 | |
|---|---|---|---|
| [**Bustory**](#1-bustory--군-지역-버스-정보-서비스) | 정보 접근성이 낮은 군 지역의 버스 시간표·실시간 도착정보 | 개인 · **운영 중** | [🔗 bustory.kr](https://bustory.kr) |
| [**Feelio**](#2-feelio--ai-감정-일기-서비스) | 음성으로 기록하고 AI가 감정을 분석하는 일기 서비스 | 해커톤 · 팀 5명 | [💻 GitHub](https://github.com/bonun00/feelio-server) |

### 두 프로젝트에서 다룬 문제

**Bustory** — 공공 API가 반환하는 "몇 초 남았다"는 상대값을 캐싱하면서 조회 시점에 따라 오차가 발생했다. API 동작을 직접 측정해 갱신 주기를 파악하고, 캐싱 기준점과 갱신 방식을 재설계했다.

**Feelio** — LLM 연동과 pgvector 기반 RAG 검색을 구현했다. 이후 코드를 검토하는 과정에서 벡터 검색이 인덱스를 활용하지 못하는 구조임을 발견하고, 실행 계획으로 원인을 확인했다.

공통적으로 **동작하는 것과 올바른 것은 다르다**는 점을 다뤘습니다. 두 문제 모두 결과 자체는 정상으로 보였고, 측정을 통해서만 드러났습니다.

---
---

# 1. Bustory — 군 지역 버스 정보 서비스

<img width="823" height="649" alt="image" src="https://github.com/user-attachments/assets/4ab73dc5-5061-47e0-9419-b1ac841b370d" />

> **"함안, 마산 등 버스 정보가 부족한 군 지역 주민들을 위한 맞춤형 실시간 버스 정보 서비스"**
>
> 국토교통부 공공데이터를 활용하여 정보 접근성이 낮은 군 단위 지역의 버스 노선, 실시간 위치, 도착 예정 시간을 빠르고 직관적인 UI로 제공합니다.

🔗 **[Bustory 공식 웹사이트 바로가기](https://bustory.kr)**

## 기술 스택

| 분야 | 기술 |
| :--- | :--- |
| **Frontend** | TypeScript, Kakao Maps SDK |
| **Backend** | Java, Spring Boot |
| **Infra & DB** | Render, Redis |
| **External API** | 국토교통부 공공데이터포털 |

## 트러블슈팅 — 실시간 도착정보 오차 개선

<!-- ![오차 그래프](images/before_error.png) -->

### 문제 상황

사용자로부터 "도착 시간이 부정확하다"는 피드백을 받았다. 확인해보니 **항상 틀린 것이 아니라 조회 시점에 따라 오차가 달랐다.** 이미 지나간 버스가 화면에 남아 있는 경우도 있었다.

### 원인 분석

프론트엔드는 서버가 내려준 도착 예정시각에서 현재 시각을 빼는 방식으로 정상 동작하고 있었다. **문제는 서버가 그 예정시각을 잘못 계산한 것이었다.**

```java
long expireAt = System.currentTimeMillis() + arrTime * 1000L;
```

공공 API의 `arrtime`은 "몇 초 남았다"는 상대값이다. 위 코드처럼 현재 시각에 남은 시간을 더해 절대시각으로 변환한 것까지는 맞았으나, 기준을 **API 호출 시각**으로 잡은 것이 문제였다.

공공 API는 일정 시간 동안 값을 갱신하지 않는데, 그동안 재호출할 때마다 도착 예정시각이 뒤로 밀렸다.

| API 호출 시각 | API 응답 | 계산된 도착 예정시각 | 밀린 정도 |
|---|---|---|---|
| 15:28:58 | 1123초 | 15:47:41 | 0초 |
| 15:30:30 | 1123초 | 15:49:13 | 92초 |
| 15:31:30 | 1123초 | 15:50:13 | 152초 |
| 15:31:31 | 1048초 | 15:48:59 | 0초 |

### 측정

파이썬 스크립트를 작성해 공공데이터 API를 15초 간격으로 호출하고 **139분간 488건**을 수집했다. 버스 도착까지 남은 시간인 `arrtime`의 변화를 기록하고, 값이 실제로 바뀐 시점과 응답에서 사라진 시점을 분석했다.

| 항목 | 측정값 |
|---|---|
| API 갱신 간격 중앙값 | 122초 |
| 갱신 간격 범위 | 30 ~ 454초 |
| **관측된 최대 오차** | **306초** |
| 소멸 시점의 남은 정류장 수 | 8건 중 8건이 2 이하 (예외 1건: 6) |

측정값을 근거로 세 개의 상수를 결정했다.

```java
POLL_INTERVAL   = 60;    // 갱신 간격 중앙값 122초의 절반
ARRIVED_PREVCNT = 2;     // 소멸 8건 중 8건이 2 이하, 예외 1건은 6
REDIS_TTL       = 300;   // 갱신 연속 실패 시 이전 값 유지용
```

**폴링 주기 60초** — 갱신 간격 중앙값의 절반으로 잡았다. 주기를 갱신 간격과 같게 두면 타이밍이 어긋날 때 변경을 최대 244초까지 늦게 포착하지만, 절반이면 상한이 122초로 줄어든다. 그 이하로 줄여도 원본이 갱신되지 않으므로 호출량만 증가한다.

**도착 판정 2정거장** — 응답에서 항목이 사라진 8건 중 8건이 남은 정류장 2개 이하 시점이었고, 유일한 예외가 6개였다. 2 이하는 도착으로 처리한다.

**캐시 TTL 300초** — 스케줄러가 60초마다 덮어쓰므로 평소에는 만료되지 않는다. 갱신이 계속 실패할 때 이전 데이터를 얼마나 보여줄지 정하는 값이다. 짧게 잡으면 실패 한두 번에 캐시가 비어 도착 시간을 표시하지 못하고, 5분이 지나면 이미 버스가 지나갔을 시간이라 정보로서의 가치가 없어 이를 상한으로 잡았다.

### 해결

**1. 기준점을 "값이 변경된 시각"으로 전환**

이전 응답과 `arrtime`을 비교해, 값이 동일하면 기존 예정시각을 유지한다.

```java
if (prev != null && prev.arrTime() == arrTime) {
    expireAt = prev.expireAt();          // 값 미변경 → 기준점 유지
} else {
    expireAt = now + arrTime * 1000L;    // 실제 갱신
}
```

**2. 조회 기반 캐싱 → 스케줄러 기반 갱신**

값의 변경 시점을 포착하려면 일정 주기 호출이 필요하다.**세 가지를 검토했다.**

**조회 시 TTL 단축** — 가장 간단하지만, TTL을 줄여도 동일한 값만 더 자주 받는다. 호출량만 증가하고 정확도는 개선되지 않는다.

**Cache-Aside + 백그라운드 갱신** — 캐시 만료 임박 시 비동기로 미리 채우는 방식. 사용자 응답을 막지 않는다는 장점은 있으나, 여전히 조회가 있어야 갱신이 트리거된다. 사용자가 보지 않는 동안의 값 변화를 놓쳐 기준점 로직이 성립하지 않는다.

**스케줄러 기반 주기 갱신** — 조회와 무관하게 고정 주기로 관측한다. 변경 시점을 최대 폴링 주기 오차 내로 포착할 수 있는 유일한 방식이라 이를 선택했다.

### 결과

| | 개선 전 | 개선 후 |
|---|---|---|
| 오차 범위 | 0 ~ 306초 | 0 ~ 60초 |
| 조회 시점 영향 | 있음 | 없음 |

스케줄러를 사용해도 매번 호출하지 않는 이상 폴링 주기만큼의 오차는 남는다. 다만 오차의 성격이 달라졌다 
조회 시점에 종속되고 재호출마다 누적되던 오차를, 폴링 주기가 오차를 결정하는 고정 오차로 전환했고 범위를 80% 가량 줄일 수 있었다.

## 서비스 운영 및 수요 분석

<img width="40%" alt="네이버 애널리틱스 분석 화면" src="https://github.com/user-attachments/assets/c7386387-66af-448a-ab58-a8a9d6c3abb5" />

단순한 서비스 개발에 그치지 않고, 네이버 애널리틱스를 연동하여 실제 사용자들의 유입과 행동 패턴을 모니터링하며 서비스를 지속적으로 개선하고 있습니다.

---
---

# 2. Feelio — AI 감정 일기 서비스
<img width="823" height="649" alt="image" src="https://github.com/user-attachments/assets/23ac38cf-1d97-458a-923f-69b59587e0b1" />

> **음성으로 기록하고, AI가 감정을 분석해 맞춤형 피드백을 제공하는 감정 일기 서비스**

**해커톤 · 팀 5명 · 48시간** · 담당: 백엔드 전반, AI 연동
💻 **[GitHub 저장소](https://github.com/bonun00/feelio-server)**

## 기술 스택

| 분야 | 기술 |
| :--- | :--- |
| **Frontend** | React |
| **Backend** | Java 17, Spring Boot, Spring AI |
| **Infra & DB** | Supabase (PostgreSQL + pgvector) |
| **AI** | Gemini API |

## AI 연동 구조

```
일기 작성 → 임베딩 생성 → pgvector 유사도 검색 → 과거 일기를 컨텍스트로 주입 → LLM 분석 → 구조화된 결과 저장
```

단순 LLM 호출이 아니라 **과거 일기를 검색해 맥락으로 제공하는 RAG 구조**를 구성했다. "지난달에도 비슷한 일로 힘들어했는데" 같은 연속성 있는 피드백을 위해서다.

## 트러블슈팅 — 벡터 검색이 인덱스를 타지 않는 구조였다

### 발견 경위

해커톤 당시 시간 제약으로 벡터 검색 쿼리를 AI 도구에 의존해 작성했다. 이후 코드를 검토하는 과정에서 **PostgreSQL 함수가 pgvector 인덱스를 활용할 수 없는 구조임을 확인**했다.

### 원인

기존 함수는 코사인 거리(`<=>`)를 유사도로 변환한 뒤, 그 값으로 필터링과 정렬을 수행했다.

```sql
SELECT sub.id, sub.content, sub.emotion, sub.created_at, sub.sim AS similarity
FROM (
  SELECT d.id, d.content, a.emotion::varchar AS emotion, d.created_at,
         (1 - (d.embedding <=> query_embedding))::float AS sim
  FROM diary d
  LEFT JOIN ai_analysis a ON d.id = a.diary_id
  WHERE d.user_id = p_user_id
) sub
WHERE sub.sim >= match_threshold
ORDER BY sub.sim DESC
LIMIT match_count;
```

pgvector 인덱스는 **`ORDER BY 벡터컬럼 <=> 기준벡터` 형태일 때만** 동작한다. `1 - 거리`로 변환하면 플래너가 이를 일반 표현식으로 취급해 인덱스를 사용하지 못한다. 결과적으로 전체 행의 거리를 계산한 뒤 정렬하는 구조가 된다.

결과 자체는 정확했고 데이터가 적을 때는 성능 문제도 드러나지 않아, 실행 계획을 확인하기 전까지는 발견되지 않았다.

### 해결
 
연산 순서를 바꾸는 것이 핵심이다. 기존은 *전부 계산 → 필터 → 정렬 → 자르기* 순서였고, 개선안은 *인덱스로 자르기 → 조인 → 필터* 순서다.
 
<table class="sqlcmp">
    
<tr><th>기존 — 전체 스캔</th ><th>개선 — 인덱스 사용</th></tr>
<tr><td>
    
```sql
    
SELECT sub.id, sub.content, sub.emotion,
       sub.created_at, sub.sim AS similarity
FROM (
  SELECT d.id, d.content, d.created_at,
         a.emotion::varchar AS emotion,
         (1 - (d.embedding <=> query_embedding))
           ::float AS sim
  FROM diary d
  LEFT JOIN ai_analysis a
         ON d.id = a.diary_id
  WHERE d.user_id = p_user_id
) sub
WHERE sub.sim >= match_threshold
ORDER BY sub.sim DESC
LIMIT match_count;
```
 
전체 행의 `sim`을 계산한 뒤 그 값으로 정렬한다. 플래너가 벡터 인덱스를 사용하지 못한다.
 
</td><td>
    
```sql
    
SELECT sub.*, a.emotion::varchar
FROM (
  SELECT d.id, d.content, d.created_at,
         1 - (d.embedding <=> query_embedding)
           AS sim
  FROM diary d
  WHERE d.user_id = p_user_id
  ORDER BY d.embedding <=> query_embedding
  LIMIT match_count
) sub
LEFT JOIN ai_analysis a
       ON sub.id = a.diary_id
WHERE sub.sim >= match_threshold;
```
 
거리 기준으로 정렬해 인덱스로 상위 N건만 읽는다. 조인과 임계값 필터는 추려낸 행에만 적용된다.
 
</td></tr>
</table>
유사도(`1 - 거리`)는 표시용으로만 계산하고, 정렬은 거리 기준으로 수행한다.

### 검증

더미 데이터 5,000건(768차원)으로 `EXPLAIN ANALYZE`를 실행했다.

**기존 방식**
```
Sort (top-N heapsort)
  -> Hash Left Join
       -> Seq Scan on diary d   (actual rows=5000 loops=1)
            Filter: ((1 - (embedding <=> ...)) >= 0)
Execution Time: 114.013 ms
```

**거리 기준 정렬로 변경**
```
Index Scan using diary_embedding_idx on diary d   (actual rows=3 loops=1)
  Order By: (embedding <=> ...)
  Filter: (user_id = 1)
Execution Time: 0.750 ms
```

전체 5,000건을 읽던 것이 3건으로 줄었다.

### 결과

전체 5,000건을 읽던 것이 3건으로 줄었다. 현재 데이터 규모에서는 인덱스 없이도 실용적인 응답 속도가 나오지만, 기존 구조는 인덱스를 추가해도 효과가 없어 데이터가 증가했을 때 대응할 수 없다는 점이 문제였다.
 
**생성된 코드는 동작 여부만으로 검증되지 않는다.** 결과가 정확하고 응답이 빨라도 구조적 한계는 남아 있을 수 있다. 이후로는 데이터 접근 쿼리에 대해 `EXPLAIN ANALYZE`로 실행 계획을 확인하는 것을 기본 절차로 삼았다.
