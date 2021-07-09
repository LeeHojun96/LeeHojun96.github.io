---

layout : single
title:  "안티 디버깅 (1) - 개념 & API 사용"
excerpt: "안티 디버깅 개념과 API를 사용한 안티디버깅 수법"

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
# 2. 안티 디버깅 기법 - API 활용
### 2.1. IsDebuggerPresent
##### - 기능 및 원리 -
- PEB 구조의 디버깅 상태값(BeingDebugged 필드)을 확인하여 해당 프로세스가 디버깅되고 있는지를 확인
  - PEB(Process Environment Block) : 유저 레벨에서 프로세스 정보를 담고 있는 구조체   
  (cf. EPROCESS : 커널 레벨에서 프로세스 정보를 담고 있는 구조체)   
    ![PEB](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-06-21-PEB.png)
    - PEB는 FS 레지스터를 통해 접근 가능 : TEB의 0x30가 PEB를 가리킴
    - FS 레지스터 : TEB를 가리킴
    - TEB : 현 프로세스에서 유저 레벨의 스레드 정보를 담는 구조체   
    ![TEB](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-06-21-TEB.png)
  - 따라서, 커널 모드의 디버거는 탐지할 수 없음
##### - 코드 -
- API syntax
  ``` C
  BOOL IsDebuggerPresent();       
  // parameter 없음
  // return 값
  //  - 0 아닌 값 : 현재 프로세스가 디버깅되고 있을 때
  //  - 0 : 현재 프로세스가 디버깅되고 있지 않을 때
  ```
- 어셈블리 코드
  ```
  mov eax, dword ptr fs:[18h]   // TEB.self(TEB 주소)를 eax로
  mov eax, dword ptr [eax+30h]  // TEB.PEB(PEB 주소)를 eax
  movzx eax, byte ptr [eax+2]   // PEB.BeingDebugged : 디버깅 중이면 1, 아니면 0
  ret                           // PEB.BeingDebugged 값을 eax에 담고 리턴
  ```
###### - 우회법 -
- IsDebuggerPresent 함수의 리턴값이 저장되는 eax에 0을 저장
  - 함수 자체를 mov, eax, 0으로 변경 : API Hooking(Trampoline 등)으로 api 변경
- PEB.BeingDebugged 값을 0으로 변경
  - 직접 어셈블리로 구현한 인라인 함수로 우회 가능


### 2.2. CheckRemoteDebuggerPresent
##### - 기능 및 원리 -
- 다른 프로세스가 디버깅 중인지 판별
- 내부적으로 NtQueryInformationProcess라는 api를 사용했었으나...)  
  - 이 API는 향후 대체되거나 사용할 수 없을 수 있음(출처 : [CheckRemoteDebuggerPresent Remark](https://docs.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-checkremotedebuggerpresent#remarks))
- 내부적으로 IsDebuggerPresent API를 사용(출처 : [CheckRemoteDebuggerPresent Remark](https://docs.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-checkremotedebuggerpresent#remarks))
  - NtQueryInformationProcess가 향후 대체될 수 있어서 IsDebuggerPresent 사용하도록 바뀐 것 같으나 확인은 해보지 못함

##### - 코드 -
- API syntax
  ``` C
  BOOL CheckRemoteDebuggerPresent(
    HANDLE hProcess,
    PBOOL  pbDebuggerPresent
  );     
  // parameter
  //  - hProcess : 디버깅 여부를 체크할 프로세스
  //  - pbDebuggerPresent : 디버깅 되고 있으면 True, 아니면 False
  // return 값
  //  - 0 아닌 값 : 현재 프로세스가 디버깅되고 있을 때
  //  - 0 : 함수 실패 시 반환값. GetLastError로 error info를 위해
  ```

##### - 우회법 -
- CheckRemoteDebuggerPresent 호출 시 두번째 파라미터 pbDebuggerPresent 값에 강제로 False 할당
- 내부에서 사용되는 API 우회
  - NtQueryInformationProcess 경우 : 아래 NtQueryInformationProcess에서 자세히 설명
  - IsDebuggerPresent 경우 : 위에서 언급한대로

### 2.3. NtQueryInformationProcess
##### - 기능 및 원리 -
- CheckRemoteDebuggerPresent에서 잠깐 언급된대로 프로세스에 관한 정보를 검색할 수 있는 윈도우 native API
##### - 코드 -
- API syntax
  ```
  __kernel_entry NTSTATUS NtQueryInformationProcess(
  HANDLE           ProcessHandle,             // 정보를 얻어올 프로세스의 핸들
  PROCESSINFOCLASS ProcessInformationClass,   // 프로세스의 정보를 반환할 타입
  PVOID            ProcessInformation,        // 위 타입에 해당하는 결과값을 반환할 버퍼 주소
  ULONG            ProcessInformationLength,  // 위에서 사용된 버퍼의 크기
  PULONG           ReturnLength               // 요청된 정보의 크기를 반환할 포인터
  );
  ```
  - ProcessInformationClass 값 : PEB 구조체의 포인터, 프로세스에 사용하는 디버거의 포트넘버, 64비트 환경에서 실행중인지 확인, 프로세스의 이름, 프로세스가 가리키는 subsystem type 값 등을 반환할 수 있음. define된 상수값으로 어떤 것을 버퍼에 저장할지 설정 가능
  - ProcessInformationClass에 따라 ProcessInformation 값이 의미하는 바가 달라짐
##### - 우회법 -
- class로 ProcessDebugPort(=0x07) 전달 시 : 프로세스에 사용하는 디버거의 포트넘버인 ProcessDebugPort(=0x07)를 ProcessInformationClass 값으로 지정할 경우 디버깅 중이면 ProcessInformation에 0이 아닌 값이, 디버깅 중이 아니라면 ProcessInformation에 0이 아닌 포트값이 저장됨. 따라서, ProcessInformation에 0을 덮어씌우면 됨.
- class로 ProcessDebugFlags(=0x1f) 전달 시 : 문서화되지 않은 값인 ProcessDebugFlags를 class로 전달 시 EPROCESS Flags의 NoDebugInherit Bit Flag 값을 반환받으므로 디버깅 중이면 0, 디버깅 중이 아니라면 1을 반환. 따라서, ProcessInformation에 1을 덮어씌우면 됨.


### 2.3. FindWindow
##### - 기능 및 원리 -
- Caption이나 class name으로 윈도우를 찾는 함수로 해당 윈도우의 핸들을 반환함
##### - 코드 -
- API syntax
```
  HWND WINAPI FindWindow(
    _In_opt_ LPCTSTR lpClassName,   // 찾고자 하는 class의 이름
    _In_opt_ LPCTSTR lpWindowName,  // 찾고자 하는 window의 이름
  );
  // return : 해당 윈도우 및 class를 찾을 경우 그 핸들을 반환, 못 찾을 경우 0 반환
```
##### - 우회법 -
- FindWindow로 전달되는 인자를 존재하지 않는 이름으로 수정
- FindWindow 반환 값을 0으로 수정

### 2.4. OutputDebugString
##### - 기능 및 원리 -
- 디버거에 출력할 문자열을 전달하는데 사용되는 함수
- SetLastError에서 설정된 값을 GetLastError가 동작하면서 확인
- 디버거가 동작 중이면 설정 값 변화 없음
- 디버거가 동작 중이지 않으면 2(지정된 파일을 찾을 수 없습니다) 리턴
##### - 코드 -
- API syntax
```
  void WINAPI OutputDebugString(
    _In_opt_ LPCTSTR lpOutputString,   // 디버거에 출력할 문자열 버퍼
  );
```
##### - 우회법 -
- GetLastError() 함수 호출 후 검사할 때 SetLastError() 값과 다르게 설정 

# 출처
- [EVERNICK & RUINA 블로그의 Windows 안티-디버깅](https://ruinick.tistory.com/category/Reversing/Anti%20Debugging)
- [night-Ohl 블로그의 IsDebuggerPresent, 우회방법](https://nightohl.tistory.com/entry/%EC%95%88%ED%8B%B0%EB%A6%AC%EB%B2%84%EC%8B%B1-IsDebuggerPresent)
