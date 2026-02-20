innodb_autoinc_lock_mode=2(interleaved)는 AUTO_INCREMENT 락을 최소화해 동시성을 높이지만, **Statement-Based Replication(SBR)** 과 함께 사용하면 **비결정적(재현 불가) 실행**으로 인해 마스터와 레플리카의 AUTO_INCREMENT 값이 달라질 수 있다.  

원인은 **마스터에서 AUTO_INCREMENT 값이 세션 간 interleave로 배정되는 순서가 동시성에 따라 달라지는데**, SBR은 binlog에 **SQL 문장만 기록**하고 레플리카에서 **문장을 단일 스레드로 재실행**하므로 동일 문장이라도 실행 시점의 상태에 따라 다른 값이 배정되기 때문이다.

---
### 발생 가능한 문제

- **multi-row INSERT / INSERT … SELECT / LOAD DATA**에서 마스터는 interleave로 인해 한 문장이 연속 범위를 고정적으로 받지 못할 수 있다.  
    예) 마스터에서는 같은 INSERT가 `101,103,106,…`처럼 배정될 수 있지만, 레플리카에서는 단독 실행되어 `101,102,103,…`처럼 연속으로 배정될 수 있다.
    
- 애플리케이션이 **LAST_INSERT_ID()**, **id 기반 샤딩/규칙**, **외부 연동 키** 등으로 생성된 ID에 의미를 부여하면 참조 무결성이나 응용 로직이 깨질 수 있다.
    
- 결과적으로 레플리카에서 **Duplicate key**, 시퀀스 불일치, 또는 **마스터/레플리카 데이터 불일치**가 발생할 수 있다(예: checksum 도구로 탐지).
    

### binlog_format 옵션별 복제 동작

- **ROW-based replication(RBR)** 에서는 변경된 row 자체(=AUTO_INCREMENT 값 포함)를 복제하므로 마스터에서 배정된 값이 그대로 적용되어 안전하다.
    
- **MIXED**는 일부 비결정적 문장을 RBR로 기록할 수 있지만, 쿼리 패턴/버전/옵션에 따라 기대와 다르게 동작할 수 있어 안정성을 최우선하면 ROW가 가장 확실하다.
    

### 실무 권장 조합

- **SBR을 사용한다면**: `innodb_autoinc_lock_mode=2`는 피하고 `innodb_autoinc_lock_mode=1(consecutive)`를 사용하거나 `binlog_format=ROW`로 전환한다.
    
- **lock_mode=2가 필요하다면**: `binlog_format=ROW`를 권장하며, MIXED를 쓸 경우 `INSERT … SELECT`, `LOAD DATA`, 대량 multi-row INSERT 등 **unsafe 가능 패턴과 경고 로그**를 반드시 모니터링한다.