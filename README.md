# Snapshot_Error

[![Java](https://img.shields.io/badge/Java-%23ED8B00.svg?style=for-the-badge&logo=java&logoColor=white)](https://www.java.com/)[![Spring](https://img.shields.io/badge/Spring-%236DB33F.svg?style=for-the-badge&logo=spring&logoColor=white)](https://spring.io/)[![Informatica](https://img.shields.io/badge/Informatica-%23FF4F00.svg?style=for-the-badge&logo=informatica&logoColor=white)](https://www.informatica.com/)

## Problem
TM 대상자를 조건에 맞게 선별하여 마케터들에게 제공하는 일 단위 배치 수행 중  
SnapShot 에러가 발생하여 재수행을 진행했으나, 지속적으로 동일한 문제가 발생  

① 트랜잭션 충돌
- 스냅샷 생성 중 특정 테이블 변경이 발생하여 배치 작업과 충돌.  

② I/O 부하 증가  
- 스냅샷 생성 시 디스크 I/O가 증가하여 배치 작업 성능 저하.  

에러의 근본적인 원인은   
① 트랜잭션 충돌이지만,   
② I/O 부하 증가도 고려하여 배치 트랜잭션 속도 향상을 추가 목표로 설정.   
--- 
## As-Is

- 기존 트랜잭션은 step 1~5로 구성됨.
-- step1: TM 대상자 메인 DB를 일 단위로 truncate 후 관련 DB를 delete.
-- step2: 회원정보 및 마케팅 대상 정보를 특정 조건으로 select.
-- step3: 조회된 정보를 IO.omm으로 변환 후 truncate/delete된 테이블에 insert.
-- step4: step3 수행과 동시에 txt 파일로 해당 정보를 인코딩하여 저장.
-- step5: EAI를 통해 마케터에게 일 단위 전송.

--- 
## Solution
SnapShot 오류의 원인은 step2에서 회원정보 DB를 조회하는 동안  
다른 배치 작업에서도 동일한 테이블에 트랜잭션이 발생하여 충돌이 발생하는 것.  
해당 테이블은 약 8천만 건 이상으로, 기타 배치 작업의 속도에 영향을 받음.  

- 복제 테이블 활용
-- 해당 테이블을 복제하여 신규 ETL 배치를 생성.
-- 복제 테이블을 사용한 서브쿼리 조인을 통해 트랜잭션 충돌 방지.
- ETL vs Java 솔루션 비교
-- 기존 Java 솔루션 대비 Informatica ETL의 수행 속도가 월등히 빠름.
-- 충돌 발생 시에도 재수행 속도 및 I/O 성능에서 유리함.
- 병렬 처리 적용
-- 기존 배치를 회원번호 기준으로 mod 4 단위로 나누어 4분할 병렬 처리.
-- 수행 속도를 단축하고 서버 효율 향상.
-- 처리된 데이터를 추가 배치로 파일 변환 후, EAI 전송.
---
## To-Be
- ①  informatica ETL 신규 배치생성
-- 새롭게 추가된 A 테이블에 oracle hint를 추가한 쿼리를 적용하여
-- insert select 작업을 수행함.
-- 정리하여, 추가된 A 테이블을 활용하여 Oracle Hint 적용 insert-select 수행.

- ②  ETL 배치 종료 후, 리팩토링된 배치 수행
-- 회원번호 기준으로 4분할 병렬 처리 후,
-- 변환된 데이터를 파일로 저장하여 EAI 송신 처리.
---
## Performance comparison 
- 상기의 작업은 4월 중순까지 예정되어잇으며, 추가로 업데이트할 예정
