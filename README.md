<br>

## 📖 목차 (Table of Contents)

1.  [**문제 정의 (The Problem)**](#1-문제-정의-the-problem)
    * [N+1 쿼리 문제란?](#n1-쿼리-문제란)
    * [내 프로젝트에서 발견된 문제점](#내-프로젝트에서-발견된-문제점)
2.  [**성능 측정 (Before) - 리팩토링 전**](#2-성능-측정-before---리팩토링-전)
    * [쿼리 로그 (N+1 발생 증거)](#쿼리-로그-n1-발생-증거)
    * [k6 부하 테스트 결과 (Before)](#k6-부하-테스트-결과-before)
3.  [**해결 (Solution) - @EntityGraph 적용**](#3-해결-solution---entitygraph-적용)
    * [Code: Before](#code-before)
    * [Code: After](#code-after)
4.  [**검증 (After) - 리팩토링 후**](#4-검증-after---리팩토링-후)
    * [쿼리 로그 (N+1 해결 증거)](#쿼리-로그-n1-해결-증거)
    * [k6 부하 테스트 결과 (After)](#k6-부하-테스트-결과-after)
5.  [**결론 (Conclusion)**](#5-결론-conclusion)
    * [성능 개선 비교](#성능-개선-비교)
    * [배운 점 (Takeaways)](#배운-점-takeaways)
    * [향후 과제](#향후-과제)

---
# 🚀 JPA N+1 쿼리 최적화 사례 연구 (JPA Query Optimization Case Study)

이 리포지토리는 Spring Boot 애플리케이션에서 발생하는 고질적인 **JPA N+1 쿼리 문제**를 식별하고, **부하 테스트(k6** 를 통해 성능을 측정하며, **`@EntityGraph` (Fetch Join)** 를 이용해 이를 해결하고 개선 성과를 정량적으로 분석하는 과정을 담은 사례 연구 프로젝트입니다.

---

## 1. 문제 정의 (The Problem)

###  N+1 쿼리 문제란?

JPA 사용 시, 연관된 엔티티를 지연 로딩(LAZY Loading)할 때 발생하는 대표적인 성능 저하 문제입니다.

1.  첫 번째 쿼리(1)로 메인 엔티티 리스트를 조회합니다. (`SELECT * FROM user;`)
2.  이후, 조회된 리스트를 순회하며 연관 엔티티에 접근할 때마다 **추가 쿼리**가 발생합니다. (`SELECT * FROM post WHERE user_id = ?`)

결과적으로, 단 한 번의 요청을 처리하기 위해 **(N+1)번**의 쿼리가 데이터베이스로 전송되어 심각한 성능 병목을 유발합니다.

###  내 프로젝트에서 발견된 문제점

* **API:** `GET /api/marginforcam/downloadExcel` (로그인한 유저의 모든 캠페인 정보 조회)
* **원인:** `marginforcam`을 조회한 후, `marginforcam.getCampaign()`처럼 연관된 `Campaign` 엔티티에 접근하는 로직에서 N+1 문제 발생이 의심되었습니다.

---

## 2. 성능 측정 (Before) - 리팩토링 전

N+1 문제가 실제로 성능에 미치는 영향을 확인하기 위해 **k6** 부하 테스트 툴을 사용했습니다.

* **시나리오:** 100명의 가상 유저(VU)가 10초 동안 동시에 `downloadExcel` API를 요청
* **k6 스크립트:**
    ```javascript
    import http from 'k6/http';
    import { check, sleep } from 'k6';
    
    const users = [
        { vu_id: 1, token: 'TestToken : GUTokenuser01' },
        { vu_id: 2, token: 'TestToken : GUTokenuser02' },
        { vu_id: 3, token: 'TestToken : GUTokenuser03' },
        { vu_id: 4, token: 'TestToken : GUTokenuser04' },
        { vu_id: 5, token: 'TestToken : GUTokenuser05' },
    ];
    
    export const options = {
        stages: [
            // 1. 5초 동안 유저를 0명에서 100명까지 서서히 늘린다. (Ramp-up)
            //    (서버에 갑작스러운 부하를 주지 않기 위한 예열 단계)
            { duration: '5s', target: 100 },
            // 2. 100명의 유저를 10초 동안 유지한다. (요구사항 핵심)
            { duration: '10s', target: 100 },
            // 3. 5초 동안 유저를 100명에서 0명으로 서서히 줄인다. (Ramp-down)
            { duration: '5s', target: 0 },
        ],
        thresholds: {
            'http_req_failed': ['rate<0.01'],   
            'http_req_duration': ['p(95)<10000'], 
        },
    };
    
    export default function () {
        // 100명의 VU가 5개의 토큰을 순환하며 사용하게 된다. 
        const currentUser = users[(__VU - 1) % users.length];
        const url = `http://localhost:8080/api/marginforcam/downloadExcel`;
    
        const params = {
            headers: {
                'Authorization': `${currentUser.token}`,
                'Content-Type': 'application/json',
            },
        };
    
        const res = http.get(url, params);
    
        check(res, {
            'is status 200': (r) => r.status === 200,
        })
        sleep(1);
    }
    ```

###  쿼리 로그 (N+1 발생 증거)

서버 로그에서 API 요청 1번에 대해 **1개의 메인 쿼리**와 **N개의 추가 쿼리**가 발생하는 것을 확인했습니다.

```sql
# 1. 메인 쿼리 (1번)
Hibernate: select c1_0.campaign_id, ... from campaign c1_0 where c1_0.email=?

# 2. 추가 쿼리 (N번)
Hibernate: select m1_0.email, ... from member m1_0 where m1_0.email=?
Hibernate: select mfc1_0.mfc_id, ... from margin_for_campaign mfc1_0 ... where mfc1_0.campaign_id=?
Hibernate: select mfc1_0.mfc_id, ... from margin_for_campaign mfc1_0 ... where mfc1_0.campaign_id=?
Hibernate: select mfc1_0.mfc_id, ... from margin_for_campaign mfc1_0 ... where mfc1_0.campaign_id=?
... (계속 반복) ...
```

###  k6 부하 테스트 결과 (Before)

<img width="1012" height="635" alt="Image" src="https://github.com/user-attachments/assets/344f69fb-5307-4800-ba51-0dcd25640b2f" />

| 메트릭 | 결과 |
| :--- | :--- |
| **요청 95% 응답 시간 (p(95))** | **`5.75s`** (5,750ms) |
| **초당 요청 수 (TPS)** | `11.09/s` |
| **요청 실패율** | `0%` (또는 에러 발생 시 수치) |

> **분석:** N+1 쿼리로 인한 과도한 DB I/O로 인해, 응답 시간이 5초를 초과하여 **사실상 서비스 불가능**한 수준임을 확인했습니다.

---

## 3. 해결 (Solution) - `@EntityGraph` 적용

이 문제를 해결하기 위해, JPQL `JOIN FETCH`보다 코드가 간결하고 의도가 명확한 **`@EntityGraph`** 어노테이션을 사용했습니다.

**핵심:** `Repository` 메소드에 `@EntityGraph`를 추가하여, JPA가 쿼리를 생성할 때 연관된 엔티티(`campaign.member`)까지 **즉시 `JOIN`**해서 가져오도록 명시했습니다.

###  Code: Before

`MarginForCampaignRepository.java`
```java
// 메소드 이름만으로 쿼리를 생성
// campaign.member에 접근할 때 N+1 발생
List<MarginForCampaign> findByCampaignMemberEmail(String email);
```

###  Code: After

`MarginForCampaignRepository.java`
```java
// @EntityGraph 추가!
@EntityGraph(attributePaths = {"campaign", "campaign.member"}) 
List<MarginForCampaign> findByCampaignMemberEmail(String email);
```

---

## 4. 검증 (After) - 리팩토링 후

동일한 시나리오로 k6 부하 테스트를 다시 수행했습니다.

###  쿼리 로그 (N+1 해결 증거)

서버 로그에서 API 요청 1번에 대해 **단 1개의 `JOIN` 쿼리**만 실행되는 것을 확인했습니다.

```sql
# 1. 단 하나의 쿼리 (JOIN 포함)
Hibernate: 
    select
        mfc1_0.mfc_id,
        c1_0.campaign_id,
        m1_0.email,
        ... (모든 필드) ... 
    from
        margin_for_campaign mfc1_0 
    left join
        campaign c1_0 
            on c1_0.campaign_id=mfc1_0.campaign_id 
    left join
        member m1_0 
            on m1_0.email=c1_0.email 
    where
        m1_0.email=?
```

###  k6 부하 테스트 결과 (After)

<img width="1121" height="688" alt="Image" src="https://github.com/user-attachments/assets/fe99c939-79af-46aa-840f-925f3f4a7aa3" />

| 메트릭 | 결과 |
| :--- | :--- |
| **요청 95% 응답 시간 (p(95))** | **`3.45s`** (3450ms) |
| **초당 요청 수 (TPS)** | `17.38/s` |
| **요청 실패율** | `0%` |

---

## 5. 결론 (Conclusion)

###  성능 개선 비교

| 메트릭 | Before (N+1) | After (Fetch Join) | 개선율 |
| :--- | :---: | :---: | :---: |
| **p(95) 응답 시간** | `5.75s` | `3.45s` | **약 40% 개선** |
| **초당 요청 수 (TPS)** | `11.09/s` | `17.38/s` | **약 1.57배 (56.7%) 향상** |

`@EntityGraph`를 통한 Fetch Join 적용으로 N+1 쿼리 문제를 해결한 결과, API의 응답 속도는 **40% 이상 개선**되었고, 서버가 초당 처리할 수 있는 요청량(TPS)은 **1.57배 이상** 증가했습니다.

###  배운 점 (Takeaways)

* JPA의 지연 로딩은 편리하지만, 서비스 로직에서 연관 엔티티를 순회할 때 N+1 문제를 쉽게 유발할 수 있음을 확인했습니다.
* "감이 아닌 **데이터**"로 성능을 측정하는 것이 중요합니다. `k6`와 같은 부하 테스트 툴은 리팩토링의 성과를 **정량적으로 증명**하는 강력한 도구입니다.
* `@EntityGraph`는 JPQL을 복잡하게 수정하지 않고도 N+1 문제를 우아하게 해결할 수 있는 매우 효과적인 방법임을 배웠습니다.

### 향후 과제

- 부하 시나리오에 대해 명확하게 설정하여, 부하 테스트를 진행 및 해석이 필요성을 인식.
- 스파이크 후 복구에 대한 검증이 부족. 
- N+1 쿼리 튜닝에도 불구하고 해당 스파이크에 대해 **p(95) 3.45초**는 동기 방식 API로서 서비스 불가 수준이라 생각.
  엑셀 생성이라는 "무거운 작업(Heavy Task)"은 @Async 또는 **메시지 큐(MQ)**를 도입해 비동기 방식으로 전환, API 응답 시간을 1초 미만으로 개선해야 함을 목표로 삼고 리팩토링을 진행 예정.
