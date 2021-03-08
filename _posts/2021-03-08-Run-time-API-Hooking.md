---

layout : single
title:  "Run-time API Hooking"
excerpt: "런타임 API Hooking 실습"

categories:
  - Blog
tags:
  - [Windows, Hooking]

toc: true
toc_sticky: true

date: 2021-03-08
last_modified_at: 2021-03-08
---
<!--
6주차 과제는 "런타임 API Hooking 수행" 입니다.
타겟 프로그램의 소스코드와 바이너리가 주어집니다.
실행 중인 타겟 프로세스의 API를 후킹하는 것이 목적이며, 타겟 프로세스의 메모리를 변조하기 위해 5주차에 학습한 방법을 사용합니다.

- [필수 1] 디버거/디스어셈블러를 사용하여 타겟 프로그램을 분석합니다. (x64dbg 추천)
- [필수 2] 실행 중인 타겟 프로세스의 메모리를 변조하는 프로그램을 작성하여 숨겨진 메시지 박스를 표시합니다. (콘솔 프로그램도 무관. 과제 5 참고)
- [선택 1] MessageBoxW 함수를 후킹하여 해당 함수가 호출될 때마다 로그를 출력합니다. (Link 1, 2)
- [선택 2] MessageBoxW 함수를 후킹하여 함수 호출에 사용된 인자를 변경합니다. (Link 3)
- [선택 3] 클래스의 멤버 함수를 후킹합니다. (Link 4, 5)

Link 1 - 5바이트 인라인 후킹, 7바이트 핫패칭: https://m.blog.naver.com/ryong0906/220615617530
Link 2 - WinAPI에 대한 7바이트 핫패칭 개념 설명: https://devblogs.microsoft.com/oldnewthing/20110921-00/?p=9583
Link 3 - 훅 함수 정의 시 주의사항: https://stackoverflow.com/questions/54852191/hooking-windows-api-function-crashes-application
Link 4 - 멤버 함수 후킹 (__fastcall 사용): https://guidedhacking.com/threads/thiscall-member-function-hooking.4036/
Link 5 - 멤버 함수 후킹 (__fastcall 미사용): https://stackoverflow.com/questions/21636482/hooking-thiscall-without-using-fastcall

이번 과제보면 선택1부터가 후킹이라고 할 수 있고, 필수 1,2는 저번 과제의 선택1의 상위 개념이러고 보시면 될거에요 (저번 선택1 과제는 코드 실행흐름을 런타임 api 후킹을 이용해서 바꾸는거고 이번 필수12는 실행흐름을 메모리 변조를 이용해서 바꾸는거고, 후킹을 하려면 일반적으로 메모리 변조가 필요합니다)
-->
