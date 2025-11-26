---
title: "[HanProject] Claude AI와 함께 테스트 코드 커버리지 50%→80% 개선하기 (3/N) - 테스트로 발견한 N+1 문제와 Flyway 완전 정복"
date: 2025-11-27 00:00:00 +0900
categories: [HanProject, TestCode]
tags: [Spring Boot, 테스트 코드, JUnit, Mockito, N+1 문제, Flyway, Docker, 리팩토링]
---

## 들어가며

[지난 글](https://dahyeon0606.github.io/posts/hanproject-backend-testcoverage2/)에서는 Flyway를 도입하여 데이터 일관성을 보장하는 과정을 다뤘습니다. 오늘은 **HandaliService와 HandbookService의 테스트 코드를 작성**하면서 발견한 성능 문제, Flyway의 심화 내용, 그리고 Docker 빌드 최적화까지 다양한 내용을 다루려고 합니다.

## HandaliService 테스트 작성

### 주급 계산 메서드 테스트

HandaliService에는 사용자의 한달이들이 받을 주급을 계산하는 복잡한 메서드가 있습니다. AI에게 테스트 코드 작성을 요청했습니다.

**생성된 테스트 코드:**

```java
@Test
public void testGetWeekSalaryInfo_singleHandaliWithJob(){
    // given
    when(userService.tokenToUser(token)).thenReturn(user);

    Job job = new Job();
    job.setName("개발자");

    Handali handali = new Handali("5월이", LocalDate.of(2024, 5, 1), user);
    handali.setJob(job);
    handali.setStartDate(LocalDate.of(2024, 11, 1));

    List<Handali> handalis = List.of(handali);

    // 스탯 생성
    List<HandaliStat> handaliStats = createHandaliStatsForSalary(handali, 50.0f, 70.0f, 30.0f);
    List<TypeName> typeNames = List.of(TypeName.ACTIVITY_SKILL, TypeName.INTELLIGENT_SKILL, TypeName.ART_SKILL);

    // when
    when(handaliRepository.findByUserAndJobIsNotNull(user)).thenReturn(handalis);
    when(handaliScheduler.calculateSalaryFor(handali)).thenReturn(5000);
    when(handaliStatRepository.findByHandaliAndStatType(handali, typeNames)).thenReturn(handaliStats);
    when(statService.checkHandaliStatForLevel(50.0f)).thenReturn(2);
    when(statService.checkHandaliStatForLevel(70.0f)).thenReturn(3);
    when(statService.checkHandaliStatForLevel(30.0f)).thenReturn(1);

    HandaliDTO.GetWeekSalaryApiResponseDTO response = handaliService.getWeekSalaryInfo(token);

    // then
    assertEquals(1, response.getHandalis_salary().size());
    assertEquals(5000, response.getTotal_salary());
    assertEquals(1, response.getTotal_handali());
}
```

기존 테스트 스타일에 맞춰 7개의 테스트 케이스를 작성했습니다:
- ✅ 한달이 1명
- ✅ 여러 한달이
- ✅ 직업 없는 경우
- ✅ 스탯이 0인 경우
- ✅ 스탯이 매우 높은 경우
- ✅ 일부 스탯만 있는 경우
- ✅ 3명의 다양한 한달이

## 테스트 중 발견한 심각한 문제: N+1

테스트 코드를 작성하고 나서 실제 코드를 다시 살펴보던 중...

**나:** "주급 계산하는 코드 자체는 괜찮아?"

**Claude:** "로직은 정확하지만 **N+1 문제**가 있습니다."

### N+1 문제란?

```java
List<Handali> handalis = handaliRepository.findByUserAndJobIsNotNull(user);  // 1번 쿼리

for (Handali handali : handalis) {
    // 매번 DB 조회!
    List<HandaliStat> handaliStats = handaliStatRepository.findByHandaliAndStatType(handali, typeNames);  // N번 쿼리
}
```

**문제의 심각성:**
- 한달이 1명: 총 2번 쿼리 (1 + 1)
- 한달이 10명: 총 11번 쿼리 (1 + 10) 🚨
- 한달이 100명: 총 101번 쿼리 (1 + 100) 🚨🚨🚨

### 또 다른 N+1 발견

```java
public HandaliDTO.HandaliStatusResponse getHandaliStatusByMonth(String token) {
    // ...
    
    // 각각 DB 조회 (4번)
    String backgroundImg = getUserItemName(user, ItemType.BACKGROUND);
    String wallImg = getUserItemName(user, ItemType.WALL);
    String sofaImg = getUserItemName(user, ItemType.SOFA);
    String floorImg = getUserItemName(user, ItemType.FLOOR);

    // 각각 DB 조회 (3번)
    Optional<HandaliStat> activityStat = handaliStatRepository.findByHandaliAndType(handali, TypeName.ACTIVITY_SKILL);
    Optional<HandaliStat> artStat = handaliStatRepository.findByHandaliAndType(handali, TypeName.ART_SKILL);
    Optional<HandaliStat> intelligentStat = handaliStatRepository.findByHandaliAndType(handali, TypeName.INTELLIGENT_SKILL);
    
    // 총 8번 쿼리! (1 + 4 + 3)
}
```

**나:** "이 부분 코드가 너무 복잡해 보이는데 로직은 맞단 말이지?"

**Claude:** "네, 로직은 정확하지만 코드가 너무 복잡하고 반복적입니다. 리팩토링해드릴게요!"

| 항목 | Before | After (개선 후) |
|------|--------|-----------------|
| **코드 줄 수** | ~50줄 | ~30줄 |
| **DB 조회** | 8번 | 3번 |
| **반복 코드** | 많음 | 없음 |
| **가독성** | 낮음 | 높음 |

## 테스트 먼저? 리팩토링 먼저?

여기서 중요한 질문이 생겼습니다.

**나:** "지금 한달이 서비스의 테스트 코드 커버리지가 낮아서, 테스트 코드를 먼저 작성하고 있는데... 일단 작성을 끝내고 로직 검토를 할까, 아님 테스트 코드 작성하면서 로직도 수정할까? 나는 전자가 나은 거 같아서 일단 테스트 코드만 작성하고 있기는 해."

**Claude:** "네, **전자(테스트 코드 먼저 작성 → 로직 개선)가 훨씬 좋은 접근**입니다! 👍"

### 왜 테스트를 먼저 작성해야 하나?

#### 1. 테스트가 안전망 역할

```
현재 상태:
- 로직: 동작은 하지만 N+1 문제
- 테스트: 없음 ❌

테스트 작성 후:
- 로직: 동작은 하지만 N+1 문제
- 테스트: ✅ 있음 (안전망 확보)

리팩토링 후:
- 로직: N+1 해결 ✅
- 테스트: ✅ 통과 (안전하게 검증)
```

#### 2. 빠른 피드백

```
수동 테스트:
1. 서버 재시작
2. 로그인
3. API 호출
4. 결과 확인
5. 반복...
→ 10분 소요 😫

자동 테스트:
1. 테스트 실행
2. 결과 확인
→ 10초 소요 😎
```

#### 3. 회귀 버그 방지

```
나중에 다른 개발자가 (또는 미래의 나):
"이 코드를 조금 수정해도 되겠지?"
→ 테스트 실행 → 실패 🚨
→ "아, 이건 건드리면 안 되는 부분이구나"
```

### 추천 작업 순서

**Phase 1: 테스트 커버리지 확보 (현재 진행 중)**
```
✅ UserService 테스트 완료
✅ HabitService 테스트 완료
🔄 HandaliService 테스트 작성 중
→ 다른 Service 테스트 작성
```

**목표: 80% 커버리지**

**Phase 2: 리팩토링 (테스트 완료 후)**
```
✅ 테스트 80% 완료
→ N+1 문제 해결
→ 코드 개선
→ 테스트 재실행으로 검증
```

**"Make it work, make it right, make it fast"**
- Make it work: 동작하게 만들기 ✅ (이미 됨)
- Make it right: 테스트 작성하기 🔄 (지금 하는 중)
- Make it fast: 성능 개선하기 ⏸️ (나중에)

## 기존 테스트 코드 리뷰

이미 작성된 테스트 코드도 검토를 받았습니다.

**나:** "테스트 코드 잘 작성된 건지 검토해줘"

**Claude:** "기본적으로 잘 작성되었지만, 몇 가지 개선할 점이 있습니다."

### 발견한 문제 1: any() 사용의 모호함

```java
// ❌ 모호함
when(userItemRepository.findByUserAndItemType(eq(user), any())).thenReturn(Optional.empty());

// ✅ 명확함
when(userItemRepository.findByUserAndItemType(user, ItemType.BACKGROUND)).thenReturn(Optional.empty());
when(userItemRepository.findByUserAndItemType(user, ItemType.WALL)).thenReturn(Optional.empty());
when(userItemRepository.findByUserAndItemType(user, ItemType.SOFA)).thenReturn(Optional.empty());
when(userItemRepository.findByUserAndItemType(user, ItemType.FLOOR)).thenReturn(Optional.empty());
```

### 발견한 문제 2: 검증 부족

```java
// ❌ 일부만 검증
assertEquals("테스트한달이", response.getNickname());
assertEquals(5, response.getDays_since_created());

// ✅ 완전한 검증
assertEquals("테스트한달이", response.getNickname());
assertEquals(5, response.getDays_since_created());
assertEquals(50.0f, response.getActivity_value());
assertEquals(70.0f, response.getIntelligence_value());
assertEquals(30.0f, response.getArt_value());
assertEquals(200, response.getMax_stat_activity());
```

### 발견한 문제 3: 엣지 케이스 누락

현재는 성공 케이스만 있었습니다. 다음 케이스들을 추가했습니다:
- ✅ 한달이가 없는 경우
- ✅ 유저 아이템이 있는 경우
- ✅ 스탯이 0인 경우
- ✅ 여러 개의 도감이 있는 경우

## HandbookService 테스트 개선

HandbookService도 검토하면서 누락된 테스트를 발견했습니다.

### 누락되었던 테스트

```java
/**
 * ✅ [테스트 목적]
 * 도감 코드가 Handbook 테이블에 존재하지 않는 경우
 * → 아무 작업도 수행되지 않고 조용히 종료되는지 검증
 */
@Test
void testAddHandbook_HandbookNotFound() {
    // given
    when(handbookRepository.findByCode("invalid_code.png")).thenReturn(null);

    // when
    handbookService.addHandbook(user, "invalid_code.png");

    // then
    verify(userHandbookRepository, never()).existsByUserAndHandbook(any(), any());
    verify(userHandbookRepository, never()).save(any());
}
```

**배운 점:** 성공 케이스뿐만 아니라 **실패 케이스, 엣지 케이스**도 테스트해야 완벽한 커버리지를 달성할 수 있습니다.

## Flyway 심화: V__ vs R__

테스트를 작성하면서 Flyway에 대한 궁금증이 생겼습니다.

**나:** "V로 시작하는 SQL문은 한번 생성되면, 딱 한번만 실행되는 거야?"

**Claude:** "네, 정확합니다!"

### V__ (Versioned Migration) - 한 번만 실행

```sql
-- V1__init_default_store_items.sql
-- V2__init_handbook_data.sql
```

**특징:**
- ✅ **딱 한 번만 실행**
- ✅ `flyway_schema_history`에 기록
- ✅ 같은 버전은 절대 재실행 안 됨
- ❌ 파일 내용 수정하면 에러 발생

**동작:**
```
첫 실행:
→ V2 실행 ✅
→ flyway_schema_history에 기록 (version: 2, checksum: abc123)

두 번째 실행 (애플리케이션 재시작):
→ version 2가 이미 있음 → 건너뜀 ❌

파일 수정 후:
→ 체크섬 불일치 감지
→ 에러 발생! 🚨
Migration checksum mismatch for migration version 2
```

### R__ (Repeatable Migration) - 여러 번 실행 가능

```sql
-- R__ensure_store_items.sql
-- R__ensure_handbook_data.sql
```

**특징:**
- ✅ **파일 내용이 변경되면 재실행**
- ✅ 체크섬(checksum) 변경 감지
- ✅ 버전 번호 없음
- ✅ 데이터 복구에 활용

**동작:**
```
첫 실행:
→ R__ 실행 ✅
→ 체크섬 기록 (checksum: 111111)

파일 수정 후 (주석 추가 등):
→ 체크섬 변경 감지 (111111 → 222222)
→ R__ 재실행 ✅
→ 데이터 복구!
```

**핵심 차이:**
- **V__**: version 숫자 있음, 한 번만 실행
- **R__**: version NULL, 체크섬 변경 시 재실행


## Docker 빌드의 진실

```
파일 생성/수정 (로컬)
    ↓
빌드 (./gradlew build)
    ↓
Docker 이미지 생성 (docker-compose build)
    ↓
컨테이너 실행 (docker-compose up)
```

### 올바른 Docker 빌드 순서

```bash
# ❌ 잘못된 순서
V2 파일 생성
→ docker-compose up -d (바로 실행)
→ V2 실행 안 됨! (jar에 V2가 없음)

# ✅ 올바른 순서
V2 파일 생성
→ ./gradlew clean build (애플리케이션 빌드)
→ docker-compose build (Docker 이미지 빌드)
→ docker-compose up -d (컨테이너 실행)
→ V2 실행됨! ✅
```

**핵심:** `--build`는 Docker 이미지만 재빌드하므로, **애플리케이션 빌드(`./gradlew build`)는 별도로 필요**합니다!


## 테스트 작성의 진정한 가치

오늘 작업을 하면서 느낀 가장 중요한 교훈입니다.

### 테스트 작성 중 발견한 것들

1. **N+1 문제 발견**
   - 테스트를 작성하지 않았다면 모르고 지나갔을 성능 문제
   - 한달이 10명일 때 11번 쿼리 → 추후 해결 예정

2. **복잡한 코드 발견**
   - 50줄 넘는 메서드, 반복되는 Optional 처리
   - 리팩토링이 필요한 부분 명확히 인식

3. **누락된 엣지 케이스**
   - 성공 케이스만 있고 실패 케이스 없음
   - 한달이 없는 경우, 스탯이 0인 경우 등 추가

4. **Flyway 마이그레이션 안전성 검증**
   - 3가지 시나리오로 완벽하게 테스트
   - 운영 환경에서도 안전하다는 확신

### "Make it work, make it right, make it fast"

```
1. Make it work ✅ - 동작하게 만들기 (이미 완료)
2. Make it right 🔄 - 테스트 작성하기 (진행 중)
3. Make it fast ⏸️ - 성능 개선하기 (다음 단계)
```

현재 2단계에 있으며, 테스트가 충분히 작성되면 3단계(N+1 해결)로 안전하게 넘어갈 수 있습니다.

## 커버리지 현황

### 현재 진행 상황

```
✅ UserService (완료)
✅ HabitService (완료)
✅ HandaliService (완료)
  ├─ changeHandali ✅
  ├─ handaliCreate ✅
  ├─ getRecentHandali ✅
  ├─ getHandaliStatusByMonth ✅
  └─ getWeekSalaryInfo ✅
✅ HandbookService (완료)
```

**현재 커버리지:** 약 60%
**목표 커버리지:** 80%

### 남은 작업

```
⏸️ StoreService
⏸️ UserItemService
⏸️ StatService
⏸️ JobService
... 기타 Service들
```

## 앞으로의 계획

### Phase 1: 테스트 커버리지 80% 달성 (진행 중)

```
현재: 60%
목표: 80%
예상 기간: 1-2주
```

### Phase 2: 리팩토링 (테스트 완료 후)

```
1. N+1 문제 해결
   - HandaliService.getWeekSalaryInfo
   - HandaliService.getHandaliStatusByMonth
   
2. 복잡한 코드 개선
   - Optional 반복 처리 → Stream + Map
   - Switch 문 → Map 자료구조
   - 50줄 메서드 → 작은 메서드로 분리
   
3. Repository 쿼리 최적화
   - Fetch Join 추가
   - IN 절 활용
   - findByHandaliAndStatType → findByHandaliInAndTypeIn
```

### Phase 3: 성능 테스트 (선택)

```
- 개선 전후 쿼리 수 측정
- 응답 시간 비교
- 부하 테스트
```

## 마무리

오늘은 HandaliService와 HandbookService의 테스트 코드를 작성하면서:

1. ✅ **N+1 문제 발견** - 테스트의 진정한 가치 실감
2. ✅ **복잡한 코드 인식** - 리팩토링 필요성 명확히
3. ✅ **Flyway 완전 정복** - V__ vs R__ 차이 완벽 이해
4. ✅ **Docker 빌드 최적화** - 효율적인 워크플로우 확립
5. ✅ **테스트 리뷰** - 기존 코드 품질 개선

**가장 중요한 깨달음:**

테스트는 단순히 "동작 확인"이 아니라:
- 🔍 **문제를 발견**하는 도구
- 🛡️ **안전하게 리팩토링**할 수 있는 안전망
- 📚 **코드를 이해**하는 문서
- 🚀 **자신감을 주는** 기반

**"테스트 없이 리팩토링하는 것은 안전벨트 없이 운전하는 것과 같다"**

다음 글에서는 남은 Service들의 테스트를 완료하고, N+1 문제를 실제로 해결하며 성능을 개선하는 과정을 공유하겠습니다.

테스트 커버리지 80% 달성과 성능 최적화까지... 화이팅! 🚀