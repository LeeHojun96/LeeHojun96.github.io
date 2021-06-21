---

layout : single
title:  "안티 디버깅 - 기본"
excerpt: "디버깅 여부 탐지"

categories:
  - Blog
tags:
  - [Windows, reversing]

toc: true
toc_sticky: true

date: 2021-06-21
last_modified_at: 2021-06-21
---
<!--
9주차 과제는 "Anti-Debugging"입니다.

프로그램을 리버싱하는 것을 막는 것을 Anti-Reversing이라고 하며 주요 방법 중 하나로 Anti-Debugging이 있습니다.
Anti-Debugging은 디버깅 여부를 탐지하거나 프로세스에 디버거가 부착되는 것을 막는 등 디버깅을 방해하는 행위입니다.
이번 목표는 Windows의 32-bit 프로세스에 대해 디버깅 여부를 탐지하는 것입니다.

안티-디버깅 기법을 알아두면 다음과 같은 상황에 도움이 됩니다.
- 해커들이 내가 개발한 프로그램의 구조를 분석하여 악성 행위를 수행합니다.
- 악성 코드에 안티-디버깅 기법이 적용되어 동적 분석을 수행할 수 없습니다.

- [필수 1] 다양한 안티-디버깅 기법들(예외 기반, HWBP 기반, VM 탐지, 파일/메모리 변조 탐지, TEB/PEB 기반, API 기반, 시간 기반, 디버거 특성 기반 등)의 기본 원리 이해 (Link 1)
- [필수 2] 유형이 서로 다른 2가지의 기법을 각각 사용하여 디버깅을 방지하는 프로그램 만들기
- [선택 1] 파일을 변경하지 않고 안티-디버깅 기법 우회하기 (프로그램을 만들어도 되고 Cheat Engine과 같은 도구를 사용해도 괜찮습니다)
- [선택 2] 안티-디버깅 우회 기법을 탐지하거나 막는 방법 생각해보기

Link 1 - EVERNICK & RUINA 블로그의 Windows 안티-디버깅: https://ruinick.tistory.com/category/Reversing/Anti%20Debugging
Link 2 - Kali-KM 블로그의 Windows 안티-디버깅: https://kali-km.tistory.com/entry/Anti-Debugging?category=490391
Link 3 - Anti-Debugging Techniques (pdf): http://www.cs.utah.edu/~aburtsev/malw-sem/slides/02-anti-debugging.pdf
Link 4 - The Ultimate Anti-Reversing Reference (pdf): https://anti-reversing.com/Downloads/Anti-Reversing/The_Ultimate_Anti-Reversing_Reference.pdf
Link 5 - Anti Debugging Techniques with Examples: https://www.apriorit.com/dev-blog/367-anti-reverse-engineering-protection-techniques-to-use-before-releasing-software
Link 6 - ScyllaHide(유저모드 안티-안티-디버깅 라이브러리) GitHub repository: https://github.com/x64dbg/ScyllaHide
-->
# 1. 안티 디버깅(anti-debugging)이란?
### 1.1. 개념
- 디버깅 여부를 탐지하거나 프로세스에 디버거가 부착되는 것을 막는 등 디버깅을 방해하는 행위
- 프로세스가 디버깅 당하면 디버깅 프로그램을 강제로 종료시키거나 에러를 발생시키는 등의 대응을 하는 것
### 1.2. 배우는 이유 및 쓰임
- 해커들이 내가 개발한 프로그램의 구조를 분석하여 악성 행위를 하고자 할 때 분석 자체를 방해할 수 있음
- 치트 프로그램 등의 경우 본인이 분석당하는 것을 막기 위해 안티 디버깅 기술을 활용할 것
- 위의 경우를 뚫기 위해 안티 디버깅을 배울 필요가 있음
- 리버싱 기술을 많이 배울 수 있음
### 1.3. 다양한 안티 디버깅 기법


![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-05-07-basic0_1.png)


# 출처
- [EVERNICK & RUINA 블로그의 Windows 안티-디버깅] (https://ruinick.tistory.com/category/Reversing/Anti%20Debugging)
