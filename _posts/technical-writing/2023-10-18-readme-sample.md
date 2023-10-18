---
title: "readme sample"
last_modified_at: 2023-10-18T17:53:13+09:00
categories:
    - technical-writing
tags:
    - technical-writing

toc: true
toc_label: "My Table of Contents"
author_profile: true
---
# 벅스 Top 100

>[벅스](https://music.bugs.co.kr/) 일일 Top 100을 파싱해서 파이썬의 딕셔너리와 리스트로 출력해주는 라이브러리입니다. 파이썬의 콜렉션을 연습하시는 분들에게 좋은 자료가 될 수 있습니다.

## 변경 로그 요약
- v0.7
    - BS4의 의존성을 제거하기 위해 파이썬의 스탠다드 라이브러리와 내장 함수로 교체
    - github tag를 이용하여 버전 관리가능

## 현재 진행 중인 항목
- [x] A{#69}
- [x] B(#70)
- [] C(#71)

## 설치 방법
1. 파이썬을 설치하세요.
2. 라이브러리를 설치하세요!

```bash
$ pip install bugs-top100
$ pip list
Package               Version
--------------------- ------------
...
bugs-top100             0.7.0
contourpy             1.0.7
cycler                0.11.0
...
```
## 기본 사용 예시
1. VSCode를 실행하세요.

```python
>>> from bugs_top100 import get_songs
>>> get_songs()
```

## 문제 해결 단계
Q1. pip가 실행이 안되요!
> 파이썬다시 설차히세요!

Q2. bugs-top100 설치했는데, 실행이 안되요!
> 파이썬이 여러개 깔려있다면 하나만 남기고 다 지우고 다시 설차하세요

## 심화 자료와 문서 링크
- 파싱(위키피디아)
- BS4를 어떻게 제거하는 방법

## 번경 로그 소개
- v0.6
- v0.5
...

## 코드 유지 관리자
- 이름, github 링크, 블로그 주소, 이메일
- 이름, github 링크, 블로그 주소, 이메일
- 이름, github 링크, 블로그 주소, 이메일
- 이름, github 링크, 블로그 주소, 이메일

## 라이센스
- MIT