---
title: "[HanProject - Backend] Claude AI와 함께 테스트 코드 커버리지 50%→80% 개선하기 (1/N)"
date: 2025-11-23 21:21:07 +0900
categories: [HanProject, Backend]
tags: [Spring Boot, 테스트 코드, JUnit, Mockito, AI, Claude]
---

## 시작하게 된 계기

한달이 프로젝트를 개발하면서 기능 구현에만 집중하다 보니, 테스트 코드 작성이 뒤로 밀렸습니다. 문득 IntelliJ의 커버리지 측정 기능으로 확인해보니 **서비스 레이어의 테스트 커버리지가 50%**밖에 되지 않았습니다.

12개의 서비스 중 절반은 테스트가 없는 상태였고, 리팩토링이나 기능 추가 시 항상 불안했습니다. "이 코드를 수정하면 다른 곳이 망가지지 않을까?" 하는 걱정이 계속 들었죠.

혼자서 모든 테스트를 작성하려니 시간도 오래 걸릴 것 같고, 테스트 코드 작성 패턴에 대한 이해도 부족했습니다. 그래서 **Claude AI의 도움을 받아 효율적으로 테스트 코드를 작성**하기로 결정했습니다.

## 현재 상태 파악

### IntelliJ 커버리지 측정 방법

IntelliJ에서 테스트 커버리지를 측정하는 방법은 간단합니다:

1. `test` 폴더 우클릭
2. `그 외 실행/디버그` 선택
3. `커버리지로 실행` 클릭

![커버리지 측정 방법](../../assets/images/hanproject-backend-testcoverage_1.png)

### 측정 결과

서비스 레이어의 커버리지가 약 50%로 나왔습니다. 12개의 서비스 중 절반 정도만 테스트가 작성되어 있었습니다.

![커버리지 측정 결과](../../assets/images/hanproject-backend-testcoverage_2.png)

**목표: 80% 이상으로 개선하기**

## AI와 함께 테스트 코드 작성하기

### 1단계: AI에게 효과적으로 요청하기

단순히 "테스트 코드 작성해줘"라고 하면 프로젝트 맥락에 맞지 않는 코드가 나올 수 있습니다. 저는 **구체적인 컨텍스트를 제공**했습니다:

```
나: "이 부분에 대한 테스트 코드를 작성하고 싶어"
    [실제 ApartmentService.java 코드 첨부]
    
나: "이미 작성한 테스트 코드가 있고, 같은 서비스 계층의 메서드인데 
     이 메서드에 대한 테스트 코드 작성해줘"
    [기존 HabitServiceTest.java 코드 첨부]
```

이렇게 하니 **프로젝트의 코딩 스타일을 유지**한 테스트가 생성되었습니다.

### 2단계: AI 코드 리뷰하며 개선하기

AI가 생성한 코드를 무작정 사용하는 게 아니라, **꼼꼼히 리뷰하고 문제를 찾아 개선**했습니다. 이 과정에서 많은 것을 배웠습니다.

#### 발견한 문제 1: 연도 경계 테스트

**AI 초안:**
```java
//12월에서 1월로 넘어가는 경우 (연도 경계)
@Test
public void testRefreshLastMonthHabits_yearBoundary(){
    //given
    when(userService.tokenToUser(token)).thenReturn(user);
    
    //현재가 1월이라고 가정 (지난달은 12월)
    int lastMonthValue = LocalDate.now().minusMonths(1).getMonthValue();
    int thisMonthValue = LocalDate.now().getMonthValue();
    // ...
}
```

코드를 읽다가 이상한 점을 발견했습니다.

**나:** "이부분 가정하면 안되고, 실제로 12월 1월로 설정해야 하는거 아닌가?"

**배운 점:**
- 테스트는 **실행 시점에 관계없이 일관된 결과**를 보장해야 합니다
- `LocalDate.now()`를 사용하면 실행할 때마다 다른 결과가 나올 수 있습니다
- AI가 만든 코드도 항상 검증이 필요합니다


#### 발견한 문제 2: 카테고리 불일치

AI가 생성한 테스트 코드에 `STUDY`, `HEALTH`, `DAILY_ROUTINE` 같은 카테고리가 사용되고 있었습니다.

**나:** "습관의 카테고리는 ACTIVITY, ART, INTELLIGENT야"

**결과:**
- AI가 즉시 모든 테스트 케이스의 카테고리를 수정
- 실제 도메인에 맞는 테스트 생성

**배운 점:**
- AI는 프로젝트의 도메인 지식을 모릅니다
- **구체적인 피드백**을 주면 즉시 수정됩니다
- 도메인 지식은 개발자가 제공해야 합니다

### 3단계: 이해 안 되는 부분 질문하며 학습하기

테스트 코드를 리뷰하다가 Service 코드에서 자주 보이던 패턴들이 궁금해졌습니다.

#### 질문 1: Stream과 map의 관계

**나:** "이 부분에서 리스트를 스트림으로 바꾸고 맵 사용하는 부분 문법이 어려운데 설명해줘"

```java
List<HabitDTO.getHabitResponse> habitResponses = habits.stream()
    .map(habit -> {
        HabitDTO.getHabitResponse habitResponse = new HabitDTO.getHabitResponse();
        habitResponse.setDetail(habit.getDetailedHabitName());
        return habitResponse;
    })
    .toList();
```

AI의 설명을 듣고 나니:
- Stream은 **컨베이어 벨트**처럼 데이터를 하나씩 처리하는 것
- `map()`은 각 요소를 **다른 형태로 변환**하는 작업
- `toList()`는 변환된 요소들을 **다시 리스트로 모으는** 것

**결과:** Service 레이어에서 자주 보이던 Entity → DTO 변환 패턴을 완전히 이해했습니다!

#### 질문 2: Map 자료형

**나:** "Map 자료형에 대해 알려줘"

AI의 설명:
- Map은 Key-Value 쌍으로 저장하는 자료구조
- Python의 Dictionary와 완전히 같은 개념

**나:** "그럼 stream과 map은 같이 쓰이는 경우가 많겠네"

**깨달음:**
- Python을 할 줄 안다면 Java Map은 이미 아는 것!
- Stream + map은 실무에서 90% 이상 함께 사용되는 패턴

이런 식으로 **이해 안 되는 부분을 즉시 질문**하니, 단순히 테스트만 작성하는 게 아니라 **Spring Boot의 핵심 패턴들을 학습**할 수 있었습니다.

## 실제로 작성한 테스트들

### ApartmentService (9개 테스트)

```java
@Test
@DisplayName("한달이에게 아파트 배정 성공")
void assignApartmentToHandali_Success() {
    // given
    Handali handali = new Handali("5월이", LocalDate.of(2024, 5, 1), testUser);
    
    // when
    Apart result = apartmentService.assignApartmentToHandali(handali);
    
    // then
    assertThat(result).isNotNull();
    assertThat(result.getFloor()).isEqualTo(5);
    assertThat(result.getApartId()).isEqualTo(2024);
    verify(apartRepository, times(1)).save(any(Apart.class));
}
```

**주요 테스트 케이스:**
- ✅ 아파트 배정 성공
- ✅ 이미 존재하는 위치에 생성 시도 → 예외 발생
- ✅ 해당 년도에 한달이가 이미 존재 → 예외 발생
- ✅ 여러 한달이 조회 시 apart_id 기준 정렬

### HabitService (15개 테스트)

```java
@Test
public void testCreateUserHabit_newHabit_newUserHabit() {
    //given
    when(userService.tokenToUser(token)).thenReturn(user);
    when(habitRepository.findByCategoryNameAndDetailedHabitName(category, details))
            .thenReturn(Optional.empty());
    
    //when
    habitService.createUserHabit(token, addHabitApiRequest);
    
    //then
    ArgumentCaptor<Habit> habitCaptor = ArgumentCaptor.forClass(Habit.class);
    verify(habitRepository).save(habitCaptor.capture());
    
    Habit capturedHabit = habitCaptor.getValue();
    assertEquals(category, capturedHabit.getCategoryName());
    assertEquals(details, capturedHabit.getDetailedHabitName());
}
```

**주요 테스트 케이스:**
- ✅ 습관 생성 (새 습관 + 새 관계)
- ✅ 습관 생성 (기존 습관 + 새 관계)
- ✅ 습관 생성 (기존 습관 + 기존 관계)
- ✅ 이번 달 습관 추가
- ✅ 지난달 습관 갱신
- ✅ 카테고리별 습관 조회 (사용자/개발자 구분)

**총 24개의 테스트 케이스 작성 완료!**

## AI와 협업하며 배운 테스트 패턴

### 1. ArgumentCaptor로 저장 데이터 검증

```java
ArgumentCaptor<UserHabit> userHabitCaptor = ArgumentCaptor.forClass(UserHabit.class);
verify(userHabitRepository).save(userHabitCaptor.capture());

UserHabit savedUserHabit = userHabitCaptor.getValue();
assertEquals(user, savedUserHabit.getUser());
assertEquals(habit, savedUserHabit.getHabit());
```

Repository에 저장되는 데이터의 내용까지 정확히 검증할 수 있습니다.

### 2. 예외 검증 패턴

```java
assertThatThrownBy(() -> habitService.addHabitsForCurrentMonth(token, addHabitApiRequest))
    .isInstanceOf(HabitNotExistsException.class)
    .hasMessage("지난달 습관이 존재하지 않습니다.");
```

AssertJ의 `assertThatThrownBy`를 사용하면 예외도 깔끔하게 테스트할 수 있습니다.

### 3. 여러 시나리오 조합 테스트

습관 생성 시 가능한 모든 조합을 테스트:
- 새 습관 + 새 관계 → 둘 다 저장
- 기존 습관 + 새 관계 → 관계만 저장  
- 기존 습관 + 기존 관계 → 아무것도 안 함

이런 엣지 케이스들을 혼자서는 생각하기 어려웠는데, AI가 제안해줘서 더 견고한 테스트를 작성할 수 있었습니다.

## AI 활용의 장단점

### ✅ 장점

**1. 속도**
- 24개의 테스트를 약 2시간 만에 작성
- 혼자 했다면 8-10시간 걸렸을 것

**2. 패턴 학습**
- Mockito의 다양한 기능 학습
- ArgumentCaptor, verify 등의 활용법
- 실무에서 자주 쓰이는 테스트 패턴 습득

**3. 다양한 케이스 고려**
- 내가 생각 못한 엣지 케이스 제안
- 경계값 테스트, 조합 테스트 등

**4. 즉시 학습 가능**
- 이해 안 되는 부분을 바로 질문
- 실시간 피드백으로 빠른 학습

### ⚠️ 주의할 점

**1. 맹신 금지**
- AI 코드도 반드시 리뷰 필요
- 연도 경계 테스트 같은 문제 발견

**2. 도메인 지식 필요**
- 프로젝트 맥락은 개발자가 제공
- 카테고리 같은 도메인 정보 수정

**3. 학습 병행 필수**
- 코드만 복붙하면 실력 안 늘음
- 이해하고 내 것으로 만들기

### 💡 효과적인 AI 활용법

1. **구체적인 컨텍스트 제공**
   - 실제 코드 첨부
   - 기존 테스트 스타일 공유

2. **생성된 코드 꼼꼼히 리뷰**
   - 문제점 찾아내기
   - 프로젝트에 맞게 수정

3. **이해 안 되면 즉시 질문**
   - "이 문법 어려운데 설명해줘"
   - "왜 이렇게 작성했어?"

4. **배운 내용 정리**
   - 블로그 글로 정리
   - 다음에 혼자서도 작성 가능

## 결과 및 배운 점

### 커버리지 개선
- **Before**: Service 레이어 50%
- **After**: Service 레이어 55% (2개 서비스 완료)
- **목표**: 80% (12개 서비스 완료 시)

### 시간 효율
- 예상 소요 시간: 8-10시간
- 실제 소요 시간: 2시간
- **약 4-5배 빠른 작업 속도**

### 학습 효과
- Mockito 패턴 완전 정복
- Stream API 깊이 이해
- Map과 Dictionary의 관계 깨달음
- 테스트 작성 시 고려해야 할 엣지 케이스 학습

### 가장 중요한 깨달음

**"AI는 도구일 뿐이다."**

중요한 것은:
- AI가 작성한 코드를 **리뷰하는 능력**
- 문제를 **발견하고 개선하는 과정**
- 이해 안 되는 부분을 **질문하며 학습하는 자세**

AI를 사용했지만, 오히려 더 많이 배우고 성장할 수 있었습니다.

## 앞으로의 계획

이제 10개의 서비스가 남았습니다:
- UserService (인증/인가 로직)
- HandaliService
- JobService
- ItemService
- ShopService
- RecordService
- ... 기타 서비스들

**최종 목표: 서비스 레이어 커버리지 80% 달성**

다음 글에서는 UserService의 인증/인가 로직을 테스트하면서, 더 복잡한 시나리오를 다루는 방법에 대해 공유하겠습니다.

## 마무리

테스트 코드 작성을 미루고 있었는데, AI의 도움으로 효율적으로 시작할 수 있었습니다. 

하지만 더 중요한 것은:
- 생성된 코드를 리뷰하며 문제를 찾아낸 것
- 이해 안 되는 부분을 질문하며 학습한 것
- 테스트 작성 패턴을 내 것으로 만든 것

**AI는 생산성을 높여주는 훌륭한 도구**입니다. 하지만 그것을 어떻게 활용하느냐는 결국 개발자의 역량입니다.

여러분도 AI를 도구로 활용하되, 항상 리뷰하고 학습하는 자세를 유지하세요! 🚀