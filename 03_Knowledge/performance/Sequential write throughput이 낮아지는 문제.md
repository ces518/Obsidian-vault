---
type: troubleshooting
area: performance
status: draft
created: 2026-03-08
updated: 2026-06-03
tags:
  - performance
---

# 개요
**Sequential write throughput이 낮아지는 문제**는 보통 스토리지나 파일 시스템이 **순차 쓰기를 실제로는 순차적으로 처리하지 못할 때** 발생한다
실무에서 자주 나타나는 대표적인 원인은 다음과 같습니다.

## 한 줄 요약

> 순차 쓰기 처리량 저하는 fsync flush, 작은 write, 파일 시스템 fragmentation, page cache writeback throttling, SSD write amplification, RAID write penalty 등으로 인해 순차 write가 실제로는 순차적으로 처리되지 못할 때 발생한다.

---
# 1. fsync / sync 호출로 인한 flush

애플리케이션이 자주 **fsync**를 호출하면 OS는 page cache의 데이터를 디스크에 즉시 flush합니다.

흐름:

write → page cache  
fsync → disk flush

이 경우 디스크가 **연속 write를 batching하지 못하고 작은 write를 반복**하게 됩니다.

결과

- I/O batching 감소
    
- 디스크 seek 증가
    
- throughput 감소
    

대표적으로 다음 시스템에서 많이 발생합니다.

- WAL (Write Ahead Log)
    
- 로그 시스템
    
- 데이터베이스 commit
    

---

# 2. 작은 write (Small Write)

순차 쓰기라도 **write size가 너무 작으면 throughput이 크게 떨어집니다.**

예:

4KB write × 1,000,000

vs

1MB write × 4000

작은 write는 다음 문제를 만듭니다.

- syscall overhead
    
- page cache 관리 비용
    
- 디스크 write coalescing 실패
    

그래서 많은 시스템이 **write batching**을 사용합니다.

예

- log batching
    
- group commit
    
- buffer flush
    

---

# 3. 파일 시스템 fragmentation

파일이 디스크에서 **연속 공간에 저장되지 않으면** sequential write가 실제로는 **random write**처럼 됩니다.

예:

logical file  
  
block1 → disk block 20  
block2 → disk block 900  
block3 → disk block 105

특히 다음 상황에서 자주 발생합니다.

- 오래된 filesystem
    
- 파일 append 패턴
    
- 많은 파일 생성/삭제
    

대표적인 영향:

- HDD → seek 증가
    
- SSD → write amplification 증가
    

---

# 4. Page Cache Writeback

OS는 page cache에 쌓인 dirty page를 **background writeback thread**가 디스크에 기록합니다.

예: **Linux kernel**

write 흐름

application write  
→ page cache  
→ dirty page  
→ background flush

문제는 dirty page가 많아지면 OS가 **throttling**을 시작한다는 것입니다.

balance_dirty_pages()

결과

- write 속도 제한
    
- throughput drop
    

---

# 5. SSD Write Amplification

SSD는 내부적으로 다음 구조를 사용합니다.

page write  
block erase

특성

- overwrite 불가능
    
- block erase 필요
    

그래서 sequential write라도 다음 상황에서 throughput이 떨어질 수 있습니다.

- GC (Garbage Collection)
    
- FTL remapping
    
- internal write amplification
    

특히 **SSD가 꽉 차 있을 때** 많이 발생합니다.

---

# 6. RAID Write Penalty

RAID 환경에서는 write가 여러 디스크에 나뉘어 기록됩니다.

특히

- RAID5
    
- RAID6
    

에서는 **read-modify-write**가 발생합니다.

예

old data read  
old parity read  
new data write  
new parity write

이 과정 때문에 sequential write 성능이 감소할 수 있습니다.

---

# 정리

Sequential write throughput이 떨어지는 대표적인 원인

|원인|설명|
|---|---|
|fsync|flush로 인해 batching 불가|
|small write|write size가 너무 작음|
|filesystem fragmentation|실제 디스크 layout이 순차적이지 않음|
|page cache writeback|dirty page throttling|
|SSD write amplification|GC / FTL 영향|
|RAID penalty|parity 계산|

---

## 관련 문서

- [[MySQL 로그 시스템 (Binlog, Relay, Redo, Undo)]]
