---

layout : single
title:  "dreamhack.io Rev_basic 문제풀이"
excerpt: "리버싱 워게임 문제 실습"

categories:
  - Blog
tags:
  - [Windows, reversing]

toc: true
toc_sticky: true

date: 2021-04-01
last_modified_at: 2021-04-01
---
<!--
10주차 과제는 디버깅 실습입니다

이론만 배우다 보면 아무래도 직접 경험하는 것보다 와닿지 않아 기억하기 어려운 경우가 많습니다. 직접 디버깅을 수행해보며 메모리(코드, 데이터, 스택 등), cpu 레지스터 등 컴퓨터 구조에 대해 생각해보면서 어셈블리 언어 해석 및 도구 사용에 익숙해지는 것이 목적입니다. 이번 주는 따로 레퍼런스는 없고 모르는 부분 있으면 따로 검색하지 말고 바로바로 질문하시면서 수행하면 됩니다.

- [필수 1] 디스어셈블러(IDA 및 기타 도구) 또는 디버거(x64dbg 등)를 사용하여 dreamhack.io 사이트의 Wargame (리버싱 분야)을 풀어봅니다. (rev-basic 0부터 2까지)
- [선택 1] 어셈블리 코드를 임의의 프로그래밍 언어(C언어 등)로 변환하여 표시합니다.
- 주의 사항: 디컴파일러(hex-rays 등) 사용 금지 (IDA 또는 x64dbg의 경우 그래프 기능은 사용해도 괜찮음)
-->

# 1. rev-basic-0 문제
### 1.1. "input :" string을 사용하여 입력값 비교 위치 추측
![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-05-07-basic0_1.png)
- input 과정을 끝내면 실행 과정은
```
lea rcx,qword ptr ss:[rsp+20]   // rcx로 입력 string의 시작주소 로드
```
- test eax, eax 이후 correct, wrong으로 분기되는 것으로 보아 chall0.7FF74B6E1000 함수가 입력값이 비교되는 위치일 것으로 추측됨
### 1.2. 입력값 비교 함수
![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-05-07-basic0_2.png)
```
lea rdx,qword ptr ds:[7FF74B6E2220]     // 00007FF74B6E2220:"Compar3_the_str1ng"
mov rcx,qword ptr ss:[rsp+40]           // [rsp+40]:"입력한 string"
call <JMP.&strcmp>                      
```
- "Compar3_the_str1ng"와 입력값을 비교
- strcmp의 결과로, 두 파라미터가 다를 경우 eax에는 1이 저장
- eax에 1이 있을 경우 다음의 명령어를 실행   
  ```
  mov dword ptr ss:[rsp+20],0             
  mov eax,dword ptr ss:[rsp+20]           
  ```
  - rsp+20을 통해 eax에 0을 저장
  - 두 파라미터가 같을 경우 위 과정을 거쳐 eax에 1이 저장
### 1.3. 이후 correct와 wrong으로의 분기
- eax에 1이 있어야 분기가 되지않고 "correct" string을 포함한 명령어로 제어가 이동
- 즉, "Compar3_the_str1ng"와 입력값이 같아야 correct쪽으로 제어가 이동하므로 해당 string이 flag!



# 2. rev-basic-1 문제
### 2.1. "input :" string을 사용하여 입력값 비교 위치 추측
 - rev-basic-0 문제와 동일
### 2.2. 입력값 비교 함수
![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-05-07-basic1_1.png)
- rsp+8에 '입력한 string'의 시작 주소를 저장하고 이를 rcx에 로드
- 이후 한 글자(character)씩 비교
  ```
  mov eax,1                               
  imul rax,rax,0                          
  mov rcx,qword ptr ss:[rsp+8]            // [rsp+8]:"입력한 string"
  movzx eax,byte ptr ds:[rcx+rax]         
  cmp eax,43                              // 43:'C'
  ```
  - 이후 "rcx+rax"에서 rax의 값은 1씩 커짐
    - imul rax,rax, 0 에서의 상수값이 1씩 커지기 때문
- 위 과정을 반복하여 Compar3_the_ch4ract3r과 비교
### 2.3. 이후 correct와 wrong으로의 분기
- rev-basic-0 문제와 동일
- 즉, Compar3_the_ch4ract3r가 flag

# 3. rev-basic-2 문제
### 3.1. "input :" string을 사용하여 입력값 비교 위치 추측
 - rev-basic-0 문제와 동일
### 3.2. 입력값 비교 함수
![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-05-07-basic2_1.png)
- 현재 실행할 cmp명령어를 통해 다음 두가지를 비교
  - edx에 저장된 입력값의 첫번째 character
  - rcx에 저장된 주소가 가리키는 string에서 rcx에 더한 값만큼 뒤에 character
- rcx에 저장된 주소를 따라가보면 다음과 같은 string을 가리킴을 확인할 수 있음
  ![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-05-07-basic2_2.png)
- 즉, 위 과정을 반복하여 Comp4re_the_arr4y와 비교
### 3.3. 이후 correct와 wrong으로의 분기
- rev-basic-0 문제와 동일
- 즉, Comp4re_the_arr4y가 flag
