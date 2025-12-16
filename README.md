# 🚀 대용량 트래픽 최적화 & 성능 튜닝 프로젝트

> **Spring Boot & JPA 기반의 캠페인 관리 서비스에서 발생한 성능 병목(N+1, Bulk Delete)을 단계별로 해결한 리팩토링 기록입니다.**

이 저장소는 대용량 데이터 처리 시 발생하는 성능 이슈를 재현하고, **문제 정의 -> 가설 수립 -> 해결 -> 검증(k6)** 과정을 통해 성능을 **12배(8.6s → 0.7s)** 개선한 과정을 담고 있습니다.

---

## 🗂️ Branch Guide (Optimization Journey)

이 프로젝트는 성능 최적화 단계에 따라 **3개의 브랜치**로 나누어져 있습니다.
아래 버튼을 클릭하면 각 단계별 상세한 **문제 해결 과정(Trouble Shooting)**과 **코드**를 확인할 수 있습니다.

### 1️⃣ Main Branch (Current)
프로젝트의 최종 완성본이자, 모든 최적화가 적용된 상태입니다.
* **주요 내용:** 전체 아키텍처 및 최종 소스 코드
* **현재 상태:** `Facade Pattern` + `Bulk Delete` + `Query Optimization` 적용 완료

<br>

### 2️⃣ Step 1: 조회 성능 최적화 (N+1 문제 해결)
JPA 조회 시 발생하는 고질적인 **N+1 문제(Read)**를 해결한 과정입니다.
* **문제:** 연관 관계가 복잡한 엔티티 조회 시 불필요한 쿼리 폭발
* **해결:** `Fetch Join`, `@EntityGraph`, `BatchSize` 적용
* **성과:** 조회 성능 최적화 및 쿼리 수 감소

<a href="https://github.com/Hi-Imjaeyoung/N1-Query-Analysis-and-Improvement/tree/feat/N1-Query-Improvement">
  <img src="https://img.shields.io/badge/View_Branch-N%2B1_Query_Fix-blue?style=for-the-badge&logo=github" alt="N+1 Branch" />
</a>

<br>

### 3️⃣ Step 2: 삭제 성능 최적화 (Bulk Delete 적용)
`Cascade.REMOVE`로 인한 **삭제 성능 저하(Delete N+1)**와 **트랜잭션 지연**을 해결한 핵심 과정입니다.
* **문제:** 대량 삭제 시 수천 건의 개별 Delete 쿼리 발생, **8.6s 소요 (Time-out)**
* **해결:** `Facade Pattern` 도입 및 `JPQL Bulk Delete` 적용 (Fire and Forget)
* **성과:**응답 속도 **0.7s**로 단축 (**약 12배 성능 향상 🚀**)

<a href="https://github.com/Hi-Imjaeyoung/N1-Query-Analysis-and-Improvement/tree/feat/bulk-delete">
  <img src="https://img.shields.io/badge/View_Branch-Bulk_Delete_Optimization-success?style=for-the-badge&logo=github" alt="Bulk Delete Branch" />
</a>

---

## 📊 Performance Summary (Final Result)

최종적으로 **Bulk Delete** 브랜치에서 달성한 성능 개선 결과입니다.

| 구분 | 개선 전 (Cascade) | 개선 후 (Bulk Delete) | 개선율 |
| :--- | :---: | :---: | :---: |
| **응답 속도 (p95)** | `8.63s` (Fail) | **`0.71s` (Pass)** | **⚡️ 91.8% 단축** |
| **처리량 (TPS)** | 저조 | **안정적** | **✅ 12배 향상** |
| **쿼리 발생 수** | N개 (수천 건) | **1개 (Bulk)** | **최소화** |

> 자세한 부하 테스트 결과와 분석 내용은 **[Bulk Delete Branch]**의 README에서 확인할 수 있습니다.

---

## 🛠️ Tech Stack
* **Java 17 / Spring Boot 3.x**
* **JPA / Hibernate / QueryDSL**
* **MySQL 8.0**
* **k6 / Grafana / Prometheus** (Performance Test)
