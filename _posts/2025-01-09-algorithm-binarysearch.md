---
title: "[Algorithm - 이론] 이분탐색(이진탐색)"
date: 2025-01-09 14:10:00 +0900
categories: [Algorithm, 이론]
# pin: true
# render_with_liquid: false
---

## 기본 개념
- 배열 내부의 값이 정렬이 되어 있을 때 사용 가능하다
- 변수 3개(start, mid, end)를 이용하여 원하는 값을 찾는 알고리즘 이다
- 시간복잡도: O(logN), n은 이분 탐색을 진행하는 리스트의 크기

<br><br><br><br>

## 반복문을 이용한 구현
```python
def binarySearch(array, start, end, k):
    while start <= end:
        mid = (start + end) // 2
        if array[mid] == k:
            return mid
        elif array[mid] > k:
            end = mid - 1
        else:
            start = mid + 1
    else:
        return False
```

<br><br><br><br>

## 나만의 팁
### 1. 이분탐색 문제 구별법

문제에 입력 값이 매우 큰 수가 나오는 경우가 많은데, 문제 자체가 완전 탐색으로 풀면 풀릴 것 같은 느낌이 듬

### 2. 문제 접근법

1. 이분 탐색으로 풀어야 한다고 판단이 되면, 
    
    <span style="color: red;">출력값에 대한 범위를 구해보기 = start, end 에 가능한 값이 무엇일까 생각해보기</span>  
    '출력될 수 있는 최소의 값 ~ 출력될 수 있는 최대의 값' 이 범위에서 이분 탐색을 진행한다고 생각하기

2. 범위가 구해지면, 문제 조건에 맞게 범위의 값을 조절하기
       (백준의 경우 이 부분이 문제 레벨에 따라 난이도가 갈리는 듯,'나무자르기-실버'의 경우 이 부분이 쉽게 생각나고, '공유기설치-골드'의 경우 이 부분이 어려웠음)