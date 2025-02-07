---
title: "[HanProject - Troubleshooting] JPA, mysql auto_increment 적용 안되는 오류"  # 문서 제목 입력
date: 2025-02-06 12:35:24  # 자동으로 오늘 날짜와 시간을 입력
categories: [HanProject, Troubleshooting]
# pin: true
# render_with_liquid: false
---


@GeneratedValue(strategy = GenerationType.IDENTITY)
오토인크리먼트 지정하는 애노테이션이다.

## 문제 1
오토인크리먼트를 적용할 때, int 는 안된다. **long**으로 지정해야 한다.

## 문제 2
디비테이블 설정이 update 로 되어 있으면 튜플은 계속 갱신이 되지만, 스키마는 맨 처음 스프링 서버 실행 시켰을 때로 고정된다. 따라서 **create** 으로 스프링 서버를 실행시키면 테이블을 지우고 다시 만들게 하면 스키마 변경 사항이 보인다.