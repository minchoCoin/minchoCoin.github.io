---
title: "μC/OS-III ch.12 Timer Managment"
last_modified_at: 2023-11-01T19:18:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
μC/OS-III는 애플리케이션 프로그래머에게 타이머 서비스를 제공하고, os_tmr.c에 타이머를 처리하는 코드가 있다. os_cfg.h에서 OS_CFG_TMR_EN을 1로 설정하면 타이머 서비스가 활성화된다.

타이머는 카운터가 0에 도달했을 때 동작을 수행하는 다운 카운터이다. 사용자는 콜백 함수(또는 단순히, 콜백(callback))을 통해 동작을 제공한다. 콜백은 타이머가 만료되면 호출될, 사용자가 선언한 함수이다. 콜백은 불을 켜고 끄거나, 모터를 구동하는 등 동작을 수행하는데 사용될 수 있다. 그러나 콜백 함수내에서 절대로 blocking 호출을 하지 않는 것이 중요하다(즉, OSTimeDly(), OSTimeDlyHMSM(), OS???Pend(), 또는 타이머 task를 막거나 삭제하는 기타 모든 것)