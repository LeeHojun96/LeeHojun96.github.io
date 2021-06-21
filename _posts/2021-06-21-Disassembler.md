---

layout : single
title:  "오픈소스 디스어셈블러 개발"
excerpt: "Zydis 디스어셈블러 개발 실습"

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
12주 차 과제는 오픈 소스 디스어셈블러 개발 실습입니다

악성 코드, 치트(게임 핵) 및 기타 소스 코드가 공개되지 않은 프로그램을 분석하기 위해서는 리버스 엔지니어링을 수행해야 합니다.
대상 프로그램들을 한정된 시간 안에 분석하기 위해서는 자동화가 불가피한데, 이번 과제에서는 여러 가지 분석 방법 중 가장 대표적인 디스어셈블링을 통해 명령 코드 및 데이터의 해석을 자동화합니다. 여러 훌륭한 오픈 소스 디스어셈블러 라이브러리가 있지만, 안정성이 뛰어나고 비교적 가벼우며 현재 x64dbg에서 사용하고 있는 Zydis를 사용합니다. (가장 많이 사용되는 것은 Capstone이라는 디스어셈블리 프레임워크입니다)

- [필수 1] 정적(정적 연결) 라이브러리와 동적(동적 연결) 라이브러리에 대해 이해합니다. (Link 1)
- [필수 2] 프로그램 구성 형식을 정적 라이브러리로, CRT 라이브러리를 동적으로 하여 Zydis 라이브러리를 빌드합니다, Link 2)
- [필수 3] 빌드한 Zydis 정적 라이브러리를 사용하여 Zydis 예제를 빌드합니다. (Link 3, 4, 5)
- [선택 1] 선형 스윕(Linear sweep) 디스어셈블러를 개발합니다.
- [선택 2] 재귀 하향(Recursive Descent) 디스어셈블러를 개발합니다.

Link 1 - 동적 라이브러리와 정적 라이브러리: https://goodgid.github.io/Static-VS-Dynamic-Libray/
Link 2 - Visual Sutdio 2019 용도로 사전 설정된 Zydis 프로젝트: https://github.com/zyantific/zydis/tree/master/msvc
=> VS 2019 IDE 설치 시 "MSVC v142 - VS 2019 C++ x64/x86 빌드 도구"를 선택해야 합니다.
Link 3 - C/C++ 프로젝트에서 정적 라이브러리 만들고 사용하기 (MSDN): https://docs.microsoft.com/ko-kr/cpp/build/walkthrough-creating-and-using-a-static-library-cpp?view=msvc-160
Link 4 - C/C++ 외부 라이브러리(dll, lib) 사용하기: https://wnsgml972.github.io/setting/2018/11/01/dll_lib/
Link 5 - GitHub Zydis 예제: https://github.com/zyantific/zydis/tree/master/examples
-->

![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-05-07-basic0_1.png)
