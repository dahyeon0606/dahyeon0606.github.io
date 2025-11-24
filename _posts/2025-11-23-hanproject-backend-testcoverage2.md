---
title: "[HanProject - Backend] Claude AI와 함께 테스트 코드 커버리지 50%→80% 개선하기 (2/N) - Flyway로 데이터 일관성 보장"
date: 2025-11-24 22:30:00 +0900
categories: [HanProject, Backend]
tags: [Spring Boot, Flyway, 데이터베이스 마이그레이션, 테스트 코드, MySQL, 리팩토링]
---

## 들어가며

[지난 글](링크)에서는 AI와 함께 테스트 코드를 작성하며 커버리지를 개선하는 과정을 다뤘습니다. 오늘은 테스트 코드를 작성하다가 발견한 **심각한 데이터 일관성 문제**와 이를 **Flyway로 해결한 과정**을 공유하려고 합니다.

## 문제의 발견

### UserService의 회원가입 로직

테스트 코드를 작성하던 중, UserService의 회원가입 메서드에서 이상한 부분을 발견했습니다:

```java
public User signUp(String name, String email, String password, String phone, LocalDate birthday){
    if(userRepository.existsByEmail(email)){
        throw new EmailAlreadyExistsException();
    }
    String encryptedPassword = passwordEncoder.encode(password);
    User user = new User(email, name, encryptedPassword, phone, birthday);
    userRepository.save(user);

    // 기본 배경, 소파, 벽장식, 바닥장식 추가
    List<UserStore> userStores = new ArrayList<>();
    List<UserItem> userItems = new ArrayList<>();

    List<Store> storeItems = storeRepository.findByPrice(0);
    for(Store store : storeItems){
        userStores.add(new UserStore(user, store)); // 구매
        userItems.add(new UserItem(user, store));   // 아이템 추가
    }

    userStoreRepository.saveAll(userStores);
    userItemRepository.saveAll(userItems);

    // 기본 이미지 도감에 추가
    handbookService.addHandbook(user,"image_0_0_0.png");

    return user;
}
```

**문제점**: `storeRepository.findByPrice(0)`이 빈 리스트를 반환하면?

회원가입은 성공하지만, 사용자는 **기본 아이템 없이** 시작하게 됩니다. 게임을 시작하자마자 배경도, 가구도 없는 빈 방만 보게 되는 것이죠.

### AI와의 대화: 첫 번째 시도

**나:** "이 코드에서 데이터베이스에 0원인 가구가 반드시 있다고 가정하고 짠 거라서 만약에 0원인 가구가 없으면 문제가 생기는데, 보통 DB 마이그레이션으로 Flyway를 사용하는 거 같은데 맞아?"

**Claude:** "네, 맞습니다! Flyway는 데이터베이스 마이그레이션 도구로 많이 사용됩니다. 기본 데이터가 없으면 문제가 생길 수 있으니 Flyway로 관리하는 것이 좋은 접근입니다."

이때부터 Flyway 설정과 마이그레이션 파일 작성이 시작되었습니다.

## Flyway 도입 과정

### 1단계: 의존성 추가

```gradle
implementation 'org.flywaydb:flyway-core'
implementation 'org.flywaydb:flyway-mysql'
```

### 2단계: application.yml 설정

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # update에서 validate로 변경

  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: 0
```

**핵심 설정 이해하기:**

**나:** "Flyway 설정 뭐 한 건지 알려줘"

**Claude의 설명:**
- `enabled: true`: Flyway 기능 활성화
- `locations`: 마이그레이션 SQL 파일 위치 (`src/main/resources/db/migration`)
- `baseline-on-migrate: true`: **이미 테이블이 존재하는 기존 DB**에 Flyway를 적용할 때 필수
- `baseline-version: 0`: 기존 DB를 "버전 0"으로 인정

마지막 두 설정이 특히 중요했습니다. 저희 프로젝트는 이미 운영 중인 DB가 있었기 때문에, 기존 스키마를 인정하고 새로운 마이그레이션만 적용해야 했습니다.

### 3단계: 마이그레이션 파일 작성 - 첫 번째 시도

`src/main/resources/db/migration/V1__init_default_store_items.sql` 파일을 생성:

```sql
-- 초안 (문제 있음)
INSERT INTO store (price, store_id, name, item_type)
SELECT 0, 1, 'No Background', 'BACKGROUND'
WHERE NOT EXISTS (SELECT 1 FROM store WHERE store_id = 1);
```

**나:** "이 파일 맞는 거야?"

**Claude:** "로직은 맞지만 한 가지 문제가 있을 수 있어요. store_id가 1, 2, 3, 4인지 확인이 필요합니다."

실제로 서버 DB를 확인해보니, store_id가 다른 값들이었습니다. AUTO_INCREMENT로 생성된 ID였기 때문이죠.

### 4단계: 마이그레이션 파일 개선

**나:** "오토 인크리먼트로 생성하는 ID 맞는데, 번호에 구애받지 않는 SQL문으로 작성하면 안 되나?"

**Claude:** "당연히 그게 더 좋습니다! store_id를 제거하고 AUTO_INCREMENT에 맡기는 것이 훨씬 안전합니다."

**최종 마이그레이션 파일:**

```sql
-- 기본 무료 아이템이 없을 경우에만 INSERT
INSERT INTO store (price, name, item_type)
SELECT 0, 'No Background', 'BACKGROUND'
WHERE NOT EXISTS (
    SELECT 1 FROM store 
    WHERE price = 0 AND item_type = 'BACKGROUND'
);

INSERT INTO store (price, name, item_type)
SELECT 0, 'No Sofa', 'SOFA'
WHERE NOT EXISTS (
    SELECT 1 FROM store 
    WHERE price = 0 AND item_type = 'SOFA'
);

INSERT INTO store (price, name, item_type)
SELECT 0, 'No Wall Decoration', 'WALL'
WHERE NOT EXISTS (
    SELECT 1 FROM store 
    WHERE price = 0 AND item_type = 'WALL'
);

INSERT INTO store (price, name, item_type)
SELECT 0, 'No Floor Decoration', 'FLOOR'
WHERE NOT EXISTS (
    SELECT 1 FROM store 
    WHERE price = 0 AND item_type = 'FLOOR'
);
```

### SQL 구문 이해하기

**나:** "작성한 SQL문 설명해줘"

**Claude의 설명:**

```sql
INSERT INTO store (price, name, item_type)  -- 1. 테이블과 컬럼 지정
SELECT 0, 'No Background', 'BACKGROUND'     -- 2. 삽입할 값
WHERE NOT EXISTS (                          -- 3. 조건부 삽입
    SELECT 1 FROM store                     -- 4. 존재 여부 확인
    WHERE price = 0 AND item_type = 'BACKGROUND'  -- 5. 검색 조건
);
```

**핵심 포인트:**
- `INSERT INTO ... SELECT ... WHERE NOT EXISTS`: 조건부 삽입 패턴
- `SELECT 1`: 실제 값이 중요하지 않고 "존재 여부"만 확인
- `NOT EXISTS`: 해당 조건의 데이터가 없을 때만 실행
- store_id 생략: AUTO_INCREMENT가 자동으로 번호 부여

**동작 흐름:**
1. price=0이고 item_type='BACKGROUND'인 데이터가 있는지 확인
2. 없으면 → INSERT 실행
3. 있으면 → 아무것도 안 함 (중복 방지)

이 패턴 덕분에 **여러 번 실행해도 중복 데이터가 생기지 않습니다**. 이것이 마이그레이션 파일에서 가장 중요한 특성입니다.

## 테스트: 모든 시나리오 검증

### 테스트 환경 구성

Docker Compose로 로컬 환경을 구성했습니다:

**나:** "docker-compose 명령어 실행하는 법"

**Claude가 알려준 핵심 명령어:**
```bash
# 실행
docker-compose -f docker-compose.prod.yml up -d

# 중지 및 삭제
docker-compose -f docker-compose.prod.yml down

# 빌드하면서 실행 - flyway 정상 동작 하는지 테스트 할 때 이 명령어를 사용해야 함
docker-compose -f docker-compose.prod.yml up -d --build
```

### 테스트 시나리오

**나:** "방금 한 마이그레이션이 정상 동작하는지 확인하려면 데이터베이스에서 해당 값을 지워보면 돼?"

**Claude:** "네, 맞습니다! 하지만 Flyway의 마이그레이션 이력도 함께 관리해야 제대로 테스트할 수 있어요."

#### 시나리오 1: 초기값이 전혀 없을 때

```sql
DROP TABLE flyway_schema_history;
DELETE FROM store WHERE price = 0;
```

```bash
docker-compose -f docker-compose.prod.yml down
docker-compose -f docker-compose.prod.yml up -d --build
```

**결과:** ✅ 4개 모두 새로 생성됨
```
store_id | price | name               | item_type
---------|-------|--------------------|-----------
x        | 0     | No Background      | BACKGROUND
x        | 0     | No Sofa           | SOFA
x        | 0     | No Wall Decoration | WALL
x        | 0     | No Floor Decoration| FLOOR
```

#### 시나리오 2: 초기값이 모두 있을 때

```sql
DROP TABLE flyway_schema_history;
```

**결과:** ✅ 중복 생성 안 됨 (4개 그대로 유지)

#### 시나리오 3: 일부만 없을 때 (SOFA만 삭제)

```sql
DROP TABLE flyway_schema_history;
DELETE FROM store WHERE price = 0 AND item_type = "SOFA";
```

**결과:** ✅ SOFA만 새로 생성됨
- BACKGROUND, WALL, FLOOR: 기존 유지
- SOFA: 새로 생성

**나:** "이 경우들에 모두 정상 동작했어"

**Claude:** "완벽합니다! 🎉 세 가지 시나리오 모두 정상 동작했다는 것은 Flyway 마이그레이션이 매우 안전하게 설정되었다는 의미입니다!"

### Flyway 실행 로그 확인

```sql
SELECT * FROM flyway_schema_history;
```

```
version | description              | type     | success
--------|--------------------------|----------|--------
0       | << Flyway Baseline >>    | BASELINE | 1
1       | init default store items | SQL      | 1
```

- **version 0**: 기존 DB 구조를 인정
- **version 1**: V1 마이그레이션 실행 완료

## 실전 환경에서의 의미

이제 다음 상황들에서 안전합니다:

### 상황 1: 새 서버 배포
```bash
# 빈 DB에 배포
→ 4개 자동 생성 ✅
```

### 상황 2: 기존 서버 업데이트
```bash
# 이미 운영 중인 서버
→ 중복 생성 없음 ✅
```

### 상황 3: 개발자가 실수로 데이터 삭제
```bash
# SOFA만 지워진 상황
→ 애플리케이션 재시작하면 자동 복구 ✅
```

### 상황 4: 향후 새 아이템 타입 추가
```sql
-- V2__add_carpet_item.sql
INSERT INTO store (price, name, item_type)
SELECT 0, 'No Carpet', 'CARPET'
WHERE NOT EXISTS (
    SELECT 1 FROM store 
    WHERE price = 0 AND item_type = 'CARPET'
);
```
→ 같은 패턴으로 안전하게 추가 가능 ✅

## ddl-auto 설정 변경

### 개발 단계의 고민

**나:** "hibernate.ddl-auto를 validate로 변경하라고 했는데, 개발 단계에서는 update가 편한데?"

**Claude의 설명:**

```yaml
# 개발 환경
ddl-auto: update  # 편리하지만 Flyway와 충돌 가능

# 운영 환경
ddl-auto: validate  # 반드시 이 설정 사용
```

**개발에서는:**
- `update` 유지 가능 (편리함)
- Entity 클래스를 보고 테이블을 자동 수정/생성
- 컬럼 추가는 해주나, 컬럼 삭제는 안 함
- 하지만 Flyway와 충돌 가능성 있음

**운영에서는:**
- 반드시 `validate` 또는 `none` 사용
- Entity와 실제 DB 스키마를 비교만 하고, 다를 경우 애플리케이션 실행 실패
- 절대 DB를 수정하지 않음
- 모든 스키마 변경은 Flyway로만 관리

결국 저는 `validate`로 변경했습니다. 스키마 변경이 필요하면 Flyway 마이그레이션 파일을 작성하는 습관을 들이기로 했습니다.

## 배운 점

### 1. 마이그레이션 도구의 필요성

초기 데이터, 스키마 변경 등을 **체계적으로 관리**하는 것이 얼마나 중요한지 깨달았습니다.

### 2. SQL의 중요성

```sql
WHERE NOT EXISTS (...)
```

이런 패턴을 알고 있으면 **안전한 데이터 마이그레이션**을 작성할 수 있습니다.

### 3. 테스트의 중요성

테스트 코드를 작성하다가 위와 같은 문제를 발견했습니다. 따라서 **테스트는 버그를 찾는 도구**이기도 합니다. 앞으로 좀 더 꼼꼼하게 service, repository, controller에 대해 테스트 코드를 작성하며 안 보였던 오류들을 찾고 싶습니다.

## 마무리

테스트 코드를 작성하다가 발견한 작은 문제가, Flyway 도입이라는 큰 개선으로 이어졌습니다.

**핵심 교훈:**
1. ✅ **테스트는 버그를 찾는 도구다**
2. ✅ **데이터 일관성은 매우 중요하다**
3. ✅ **마이그레이션 도구를 사용하자**
4. ✅ **AI와 함께 배우면 더 빠르다**

이제 회원가입 시 기본 아이템이 보장되며, 향후 새로운 아이템 타입을 추가할 때도 안전하게 마이그레이션할 수 있게 되었습니다.

다음 글에서는 남은 Service 레이어들의 테스트 코드를 작성하며, 더 복잡한 비즈니스 로직을 다루는 방법에 대해 공유하겠습니다.