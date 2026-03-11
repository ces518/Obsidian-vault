## 1. 토픽과 파티션

Kafka는 **처리량 확장과 병렬 처리를 위해 토픽을 여러 파티션으로 분할**한다.

- 하나의 토픽은 여러 **Partition** 으로 구성된다.
    
- 각 파티션은 **append-only 로그** 형태로 메시지를 저장한다.
    
- 파티션은 **Kafka의 병렬 처리 단위**가 된다.
    

Topic  
 ├ Partition 0  
 ├ Partition 1  
 └ Partition 2

---

# 2. 메시지 라우팅 (Partitioner)

Producer 내부에는 **Partitioner(라우팅 계층)** 가 존재하며  
메시지를 **어떤 파티션으로 보낼지 결정**한다.

대표적인 방식

- key hash 기반
    
- round-robin
    
- custom partitioner
    

Producer  
   ↓  
Partitioner  
   ↓  
Partition 선택

---

# 3. 파티션 복제 (Replication)

각 파티션은 **여러 브로커에 replica 형태로 저장**된다.

구조

Partition 0  
 ├ Leader   → Broker1  
 ├ Follower → Broker2  
 └ Follower → Broker3

특징

- Producer / Consumer는 **Leader와만 통신**
    
- Follower는 **Leader 로그를 복제**
    

목적

- **데이터 내구성 (Durability)**
    
- **고가용성 (High Availability)**
    

---

# 4. ISR (In-Sync Replicas)

Kafka는 **leader와 충분히 동기화된 replica 집합을 ISR** 이라고 한다.

Replica = {1,2,3}  
ISR     = {1,2}

ISR은 다음 역할을 한다.

- **leader election 후보**
    
- **write durability 기준**
    

---

# 5. 메시지 Commit 기준 (High Watermark)

Kafka는 **consumer가 읽을 수 있는 offset을 제한**한다.

High Watermark (HW)

HW 의미

> ISR replica들이 모두 복제한 마지막 offset

따라서

consumer read ≤ HW

이 구조로 인해

- follower lag 존재 가능
    
- 하지만 **존재하지 않는 메시지 읽는 상황은 발생하지 않음**
    

---

# 6. 장애 발생 시 동작

### Leader 장애

Leader down  
↓  
ISR follower 중 하나가 leader 승격

→ 서비스 지속

---

### 모든 replica 장애

Partition replica 전부 다운

결과

Partition unavailable

- read/write 불가
    
- 데이터는 디스크에 존재
    
- 브로커 복구 시 자동 복구
    

---

# 7. Kafka 구조 핵심 역할 분리

Partition   → 처리량 확장 / 병렬 처리  
Replica     → 고가용성 / 데이터 보호  
ISR         → 동기화 상태 관리  
HW          → 읽기 일관성 보장  
Partitioner → 메시지 라우팅

---

# 한 줄 핵심 정리

Kafka는 토픽을 여러 파티션으로 분할해 처리량과 병렬성을 확보하고,  
각 파티션을 여러 브로커에 복제하여 leader-follower 구조로  
고가용성과 데이터 내구성을 보장한다.

---
