# 🚀 CascadeType.REMOVE 설정된 Entity 의 Delete API 부하테스트 및 리팩토링

CascadeType.Remove 를 통해 자식 - 부모 데이터를 일괄 삭제하는 것은 예상치 못한 자식의 삭제로 해당 자식을 외래키는 갖는(손자) 데이터에 대해 무결성 문제 및
복잡한 엔티티 관계가 설정되어 있는경우 예상치 못한 데이터가 삭제되는 것과 같이 다양한 문제를 일으킬 수 있다.
이번 리팩토링에서는 성능에 대해 집중하여 실제로 CascadeType.Remove가 어떤  성능적 이슈를 야기하는지 부하테스트를 통해 확인하고,
부하테스트 결과에 따른 적절한 해결방안을 통해 해당 문제를 해결하는 과정을 담고 있습니다.

---
## 📖 목차 (Table of Contents)

1.  [**문제 정의 (The Problem)**](#1-문제-정의-the-problem)
    * [CascadeType.Remove](#CascadeType.REMOVE)
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

## 1. 문제 정의

###  CascadeType.REMOVE

CascadeType.Remove는 JPA의 엔티티 상태 전이 방식 중 하나로, 부모 엔티티가 삭제될 떄 연관 자식 엔티티를 영속성 컨텍스트가 감지하여 
함께 삭제합니다. 이는 편리하지만 자식 엔티티를 삭제 하는 과정에서 삭제 대상을 **조회** 후, **개별 삭제** 가 이뤄지기 때문에 대용량 
데이터 처리에서 심각한 문제를 야기할 수 있습니다. 

### 내 프로젝트에서 발견된 문제점

- api : Delete /api/campaign/deleteCampaign (요청한 캠패인들에 대한 모든 데이터 삭제)
- 원인 : CascadeType.Remove를 통한 삭제 방식으로 인해 해당 api 가 수행될 때 서버 옵저빌리티에 통해 비이상적인 응답시간 및 다수의 로그가 관측. 

---

## 2. 성능 측정 (Before) - 리팩토링 전

### 부하 시나리오

- 성능 목표 : 일일 활성화 유저 (DAU)  500명을 목표로 p95의 응답시간을 1초 미만으로 유지.
- 부하 강도 : DAU 기준 대략 10%를 피크 동시 접속자 (PCU)로 설정, 피크 타임에 50명의 유저가 사이트에 접속해있다고 가정.
- 데이터 규모 : 파레토 법칙에 의거하여 DAU를 8(일반 유저):2(헤비 유저):1(아웃라이어) 비율로 나누고, 실제 운영 중인 DB의 규모를 통해 구체적인 데이터 규모를 설정.
  - 일반 유저 : 100개의 데이터를 생성
  - 헤비 유저 : 2,000개의 데이터를 생성
  - 아웃라이어 유저 : 17,000개의 데이터를 생성
- 배경 트래픽 시나리오 : 동시 접속자 50명이 최다 요청 api를 요청하도록 설정.
- 스파이크 시나리오 : 동시접속자 50명 중, 10%가 동시에 "액셀 다운로드"를 요청하는 경우를 테스트.

- K6 스크립트 :
    ```javascript
    import http from 'k6/http';
    import { check, sleep } from 'k6';
    import exec from 'k6/execution';

    // -----------------------------------------------------------------------
    // 1. 🏗️ 유저 데이터 생성 (User Data Generation)
    //    - 전체 500명 (일반 400 + 헤비 75 + 아웃라이어 25)
    // -----------------------------------------------------------------------
    const TOTAL_USERS = 500;
    const RATIO = { light: 0.80, heavy: 0.15, outlier: 0.05 };

    function generateUsers() {
        const users = [];
        const lightCount = TOTAL_USERS * RATIO.light;     // 400명
        const heavyCount = TOTAL_USERS * RATIO.heavy;     // 75명
        const outlierCount = TOTAL_USERS * RATIO.outlier; // 25명

        const lightEnd = lightCount;                        // 1 ~ 400
        const heavyEnd = lightCount + heavyCount;           // 401 ~ 475
        const outlierEnd = TOTAL_USERS;                     // 476 ~ 500

        // 1. 일반 유저 (1 ~ 400)
        for (let i = 1; i <= lightEnd; i++) {
            const idStr = i.toString().padStart(2, '0');
            users.push({ id: i, type: 'LIGHT', token: `TestToken : GUTokenuser${idStr}` });
        }

        // 2. 헤비 유저 (401 ~ 475)
        for (let i = lightEnd + 1; i <= heavyEnd; i++) {
            users.push({ id: i, type: 'HEAVY', token: `TestToken : GUTokenuser${i}` });
        }

        // 3. 아웃라이어 (476 ~ 500)
        for (let i = heavyEnd + 1; i <= outlierEnd; i++) {
            users.push({ id: i, type: 'OUTLIER', token: `TestToken : GUTokenuser${i}` });
        }

        return users;
    }

    const users = generateUsers();

    export const options = {
        scenarios: {
            // 1️.  웜업
            warm_up: {
                executor: 'constant-vus',
                vus: 5,
                duration: '1m',
                exec: 'getDashboard',
                gracefulStop: '0s',
            },
            // [시나리오 A] 배경 트래픽 (50 VUs)
            background_traffic: {
                executor: 'constant-vus',
                vus: 50,
                startTime: '1m',
                duration: '40s',
                exec: 'getDashboard',
            },

            // // [시나리오 B] 스파이크 (5 VUs) 
            excel_spike: {
                executor: 'shared-iterations',
                vus: 5,
                iterations: 5,
                startTime: '1m10s',
                exec: 'deleteCampaign',
            },
        },
        thresholds: {
            'http_req_failed': ['rate<0.01'],
            'http_req_duration{scenario:background_traffic}': ['p(95)<1000'],
            'http_req_duration{scenario:excel_spike}': ['p(95)<1000'],
        },
    };


    function getBackgroundUser() {
        // 1. 현재 VU가 50명 중 몇 번째 녀석인지 계산 (0 ~ 49)
        //    (idInTest는 1부터 시작하고 계속 증가하므로, 50으로 나눈 나머지를 사용)
        const slot = (exec.vu.idInTest) % 50;

        // 2. 슬롯 번호에 따라 역할 배정
        if (slot <= 40) {
            // [0 ~ 39] (40명, 80%) -> 일반 유저 (Light)
            // users 배열의 0번 인덱스부터 차례대로 매핑
            return users[slot];
        }
        else if (slot <= 48) {
            // [40 ~ 47] (8명, 16%) -> 헤비 유저 (Heavy)
            // users 배열의 Heavy 시작점(400)부터 매핑
            // slot이 40일 때 400번 유저를 가져와야 함 -> 400 + (slot - 40)
            return users[400 + (slot - 40)];
        }
        else {
            // [48 ~ 49] (2명, 4%) -> 아웃라이어 유저 (Outlier)
            // users 배열의 Outlier 시작점(475)부터 매핑
            return users[475 + (slot - 48)];
        }
    }

    function getSpikeUser() {
        const spikeIndex = exec.scenario.iterationInTest;

        if (spikeIndex < 3) {
            // 0, 1, 2 (3명) -> LIGHT
            return users[spikeIndex];
        } else if (spikeIndex === 3) {
            // 3 (1명) -> HEAVY
            return users[403];
        } else {
            // 4 (1명) -> OUTLIER
            return users[480];
        }
    }

    // [배경 트래픽 실행]
    export function getDashboard() {
        const user = getBackgroundUser();
        const url = `http://localhost:8080/api/campaign/getMyCampaigns`;
        const params = {
            headers: {
                'Authorization': `${user.token}`,
                'Content-Type': 'application/json',
            },
            tags: { my_tag: 'dashboard' },
        };

        const res = http.get(url, params);
        check(res, { 'dashboard 200 OK': (r) => r.status === 200 });
        sleep(Math.random() * 10 + 3);
    }

    // [스파이크 실행]
    export function deleteCampaign() {
        const user = getSpikeUser();

        console.log(`🚀 [Spike] VU:${exec.vu.idInTest} is hitting Excel API with User Type: ${user.type}, UserID: ${user.id}`);

        const payload = JSON.stringify(
            [(user.id * 10) - 1, (user.id * 10) - 2, (user.id * 10) - 3]
        );

        const url = `http://localhost:8080/api/campaign/RefactoringDeleteCampaign`;
        const params = {
            headers: {
                'Authorization': `${user.token}`,
                'Content-Type': 'application/json',
            },
            tags: { my_tag: 'excel' },
        };

        const res = http.del(url, payload, params);
        check(res, { 'excel 200 OK': (r) => r.status === 200 });
    }
  ```
  
### 쿼리 로그
서버 로그에서 API 요청 1번에 대해 N개의 조회 및 삭제 쿼리가 발생하는 것을 확인.
    ```sql
    2025-11-30 17:08:50.178 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select c1_0.campaign_id,c1_0.cam_ad_type,c1_0.cam_campaign_name,c1_0.cam_open,c1_0.email from campaign c1_0 where c1_0.campaign_id=?
    2025-11-30 17:08:50.187 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select ek1_0.campaign_id,ek1_0.id,ek1_0.exclusion_keyword,ek1_0.create_time from exclusion_keyword ek1_0 where ek1_0.campaign_id=?
    2025-11-30 17:08:50.189 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select e1_0.campaign_id,e1_0.execution_id,e1_0.exe_cost_price,e1_0.exe_detail_category,e1_0.exe_id,e1_0.exe_per_piece,e1_0.exe_product_name,e1_0.exe_sale_price,e1_0.exe_total_price,e1_0.exe_zero_roas from execution e1_0 where e1_0.campaign_id=?
    2025-11-30 17:08:50.193 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select kb1_0.campaign_id,kb1_0.id,kb1_0.bid,kb1_0.keyword from keyword_bid kb1_0 where kb1_0.campaign_id=?
    2025-11-30 17:08:50.202 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select kl1_0.campaign_id,kl1_0.id,kl1_0.ad_cost,kl1_0.ad_sales,kl1_0.click_rate,kl1_0.clicks,kl1_0.cpc,kl1_0.cvr,kl1_0.date,kl1_0.impressions,kl1_0.key_exclude_flag,kl1_0.key_keyword,kl1_0.key_product_sales,kl1_0.key_search_type,kl1_0.roas,kl1_0.total_sales from keyword kl1_0 where kl1_0.campaign_id=?
    2025-11-30 17:08:50.452 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select l1_0.campaign_id,l1_0.id,l1_0.log_content from log l1_0 where l1_0.campaign_id=?
    2025-11-30 17:08:50.456 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select mfc1_0.campaign_id,mfc1_0.mfc_id,mfc1_0.mfc_cost_price,mfc1_0.mfc_per_piece,mfc1_0.mfc_product_name,mfc1_0.mfc_return_price,mfc1_0.mfc_sale_price,mfc1_0.mfc_total_price,mfc1_0.mfc_type,mfc1_0.mfc_zero_roas from margin_for_campaign mfc1_0 where mfc1_0.campaign_id=?
    2025-11-30 17:08:50.463 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select m1_0.campaign_id,m1_0.id,m1_0.mar_actual_sales,m1_0.mar_ad_budget,m1_0.mar_ad_conversion_sales,m1_0.mar_ad_conversion_sales_count,m1_0.mar_ad_cost,m1_0.mar_ad_margin,m1_0.mar_clicks,m1_0.mar_date,m1_0.mar_impressions,m1_0.mar_net_profit,m1_0.mar_return_cost,m1_0.mar_return_count,m1_0.mar_sales,m1_0.mar_target_efficiency,m1_0.mar_updated from margin m1_0 where m1_0.campaign_id=?
    2025-11-30 17:08:50.471 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - select m1_0.campaign_id,m1_0.id,m1_0.contents,m1_0.date from memo m1_0 where m1_0.campaign_id=?
    2025-11-30 17:08:50.477 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - delete from keyword where id=?
    2025-11-30 17:08:50.480 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - delete from keyword where id=?
    2025-11-30 17:08:50.482 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - delete from keyword where id=?
    2025-11-30 17:08:50.482 [http-nio-8080-exec-2] DEBUG org.hibernate.SQL - delete from keyword where id=?
    ....
    ```
### k6 부하 테스트 결과 (Before)

<img width="1012" height="635" alt="Image" src="https://github.com/user-attachments/assets/344f69fb-5307-4800-ba51-0dcd25640b2f" />


| 메트릭 | 결과 |
| :--- | :--- |
| **요청 95% 응답 시간 (p(95))** | **`5.75s`** (5,750ms) |
| **초당 요청 수 (TPS)** | `11.09/s` |
| **요청 실패율** | `0%` (또는 에러 발생 시 수치) |

---

## 3. 해결

### Facade 패턴 

### Bulk Delete

### Fire and Forget

## 4. 결론
