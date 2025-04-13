# 📌 Snapshot_Error 개선 프로젝트

[![Java](https://img.shields.io/badge/Java-%23ED8B00.svg?style=for-the-badge&logo=java&logoColor=white)](https://www.java.com/)
[![Spring](https://img.shields.io/badge/Spring-%236DB33F.svg?style=for-the-badge&logo=spring&logoColor=white)](https://spring.io/)
[![Informatica](https://img.shields.io/badge/Informatica-%23FF4F00.svg?style=for-the-badge&logo=informatica&logoColor=white)](https://www.informatica.com/)

---

## 🧩 Overview

본 프로젝트는 현대카드 마케팅 시스템에서 운영 중인 **TM 대상자 배치 작업 중 발생한 SnapShot 오류**를 해결하고,  
**처리 성능 및 안정성 개선을 동시에 달성**하기 위해 진행된 최적화 사례입니다.

- **적용 대상 부서**: 마케팅본부 TM 운영
- **배포 대상 시스템**: 대상자 선별 배치 + 파일 전송 시스템
- **운영 목적**: 하루 3천만 건 이상의 대상자를 안정적으로 선별, 마케터에게 자동 전달

---

## ❗ Problem

SnapShot 오류로 인해 대상자 배치가 불완전하게 수행되거나 재처리가 빈번하게 발생하였음.

### 주요 이슈

1. **트랜잭션 충돌**
   - 배치 수행 중 회원 테이블을 동시에 접근하는 다른 배치와 충돌

2. **I/O 병목**
   - 스냅샷 수행 중 디스크 I/O 과다로 인한 전체 배치 지연

> 근본 원인은 트랜잭션 충돌이며, 성능 병목도 병렬성과 구조 개선이 필요한 상황이었습니다.

---

## 🔎 As-Is 구조

기존 구조는 ETL 없이 **순수 Java 기반 CTM 스케줄링 시스템**으로 구성되어 있었으며, 다음과 같은 순서로 운영되었습니다.

```
step 1. 대상자 테이블 truncate 및 delete  
step 2. 회원 및 마케팅 정보 select  
step 3. 대상자 insert + 변환  
step 4. txt 파일 인코딩 저장  
step 5. EAI 연계 전송
```

- 단일 트랜잭션 기반 처리 (병렬 미적용)
- Java 코드에서 직접 대량 데이터 처리 수행
- 테이블 접근 충돌 시 SnapShot 오류 빈번히 발생
- 대상자 추출 → 파일 생성 → 전송까지 단일 Job에 포함됨

---

## 💡 Solution

### 구조적 개선 포인트

- **복제 테이블 사용**
  - 대상 테이블을 별도 DB에 미리 복제
  - ETL 기반 사전 데이터 적재 → SnapShot 충돌 제거

- **병렬 처리 도입**
  - 회원번호 기준 `mod 4`로 분할 처리
  - Java 병렬 배치 도입 → insert 및 연산 속도 개선

- **ETL vs Java 판단**
  - 단일 insert-select 쿼리는 Lock 이슈로 실패
  - Java 기반 `Split 4` 방식이 성능·안정성 측면에서 최적

---

## ✅ To-Be 구조

최종 개선된 배치 흐름은 다음과 같습니다.

```
05:00 ETL 시작
 └→ 전일자 DB 조회 → 복제 테이블 적재 (mod 4)
     └→ EAI 완료 신호 수신
         └→ Step1 - 일자별 truncate + insert
             └→ Step2 - 병렬 (Read / Process / Write)
                 └→ Step3 - 후처리 insert
                     └→ Step4 - 파일 생성
                         └→ Step5 - EAI 전송
```

- `mod_split_flag` 컬럼 추가로 분할 처리 기반 마련
- ETL은 항상 truncate 후 insert 처리
- Java는 병렬로 읽고/처리하고/쓰기 (R-P-W 구조)

<img src="https://user-images.githubusercontent.com/111719007/223459638-60844406-0679-4954-951b-4517016ce132.gif" width="200" height="400"/>

---

## 📁 상세 배치 처리 흐름

### 1️⃣ ETL 배치 (Informatica)

- **시점**: 매일 05:00
- **소스**: 전일자 DB의 회원 테이블 (복제 대상)
- **처리 방식**: `회원번호 % 4` 기준 파티셔닝
- **사전 처리**: truncate → insert-select

### 2️⃣ Java 배치 (CTM 연계)

- **전제 조건**: EAI in-condition 수신 (ETL 완료)
- **Step1**: 대상자 테이블 초기화 및 insert
- **Step2**: `mod 0~3` 별로 병렬 R-P-W 처리
- **Step3**: 후처리 일자 insert
- **Step4**: `FileWriter` 활용 DAT 파일 생성
- **Step5**: DAT 파일 EAI로 자동 전송

---

## 📊 Performance Comparison

| 항목 | 개선 전 평균 | 개선 후 평균 | 개선율 |
|------|--------------|--------------|--------|
| 전체 수행 시간 | 2시간 30분 | 33~38분 | **약 75% ↓** |
| 회원정보 조회 | 50분 | 12분 | **약 76% ↓** |
| 파일 생성 및 전송 | 35분 | 17분 | **약 54% ↓** |

> 총 약 112분 단축.  
> SnapShot 오류 0건, 디스크 부하 완화, Lock 건수 0건 달성.

---

## 🚀 Improved Outcome

- SnapShot 오류 발생률 **0%**
- 일일 대상자 추출 시간 **75% 이상 단축**
- 시스템 자원 활용률 **50% 이상 개선**
- 재처리 건수 **완전 제거**

---

## 📘 Lessons Learned

### 1. 사전 복제 구조로 트랜잭션 충돌을 원천 차단하라
- 복제 테이블 + 사전 데이터 적재는 단순하지만 가장 강력한 안정화 전략

### 2. update vs truncate → insert: 성능은 분명히 다르다
- 대량 배치 환경에서는 truncate 후 insert 방식이 update보다 훨씬 효율적

### 3. Java 병렬처리 vs DB 병렬처리
- 단일 insert-select는 DB Lock 문제 발생 가능성 높음
- Java 기반 4분할 처리 방식이 확실한 이점을 제공

---

## 🗓️ Deployment & Schedule

- ✅ 2025.04.16: 개선 배치 정식 배포 완료

---

## 📎 참고 사항

- 현재 레포는 코드 비공개 상태로 운영되며, 문서 중심으로 구성되어 있습니다.
- 전체 흐름도는 `docs/flow_diagram.png` 또는 Draw.io 기반 다이어그램으로 추후 제공 예정입니다.

---