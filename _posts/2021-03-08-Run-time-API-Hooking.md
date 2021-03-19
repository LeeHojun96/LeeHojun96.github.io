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
# 0. 개요
- Target.exe라는 샘플 프로그램의 메모리를 변조하거나 사용하는 api를 hooking하여 숨겨진 메세지 박스를 display하는 것이 목표
- Target.exe를 리버싱할 때는 pdb 등의 파일을 제거하고 수행하는 것을 권함
  => 이러한 파일에는 함수 이름 등 디버깅 정보가 포함되어 있는데 디버거 실행 시 이런 정보들이 출력되어 리버싱 연습에 부적합
# 1. Target.exe  분석
### 1.1. 프로그램 동작   
- 기본적으로 1~5까지의 입력을 받아 각 메뉴를 실행함
  ![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-03-08-success.png)
  - 1번 : 메세지 박스를 보여줌
  - 2번 : 메세지 박스의 내용물을 설정
    - display할 문자열을 입력받음
  - 3번 : 메세지 박스의 제목을 설정
    - display할 문자열을 입력받음
  - 4번 : 메세지 박스 내용물과 제목을 초기화
  - 5번 : 프로그램 나가기  

### 1.2. 프로그램 분석
- 가설을 세워 hidden message와 관련된 정보가 있을 것으로 추정되는 위치를 추측
  - 사용자가 입력할 수 있는 경우는 3가지
    - (1) 첫 메뉴 선택
    - (2) 2번 선택 후 display할 문자열 입력
    - (3) 3번 선택 후 display할 문자열 입력
  - 이 중 첫 메뉴 선택에서의 입력값과 히든 메세지와 관련이 있을 것이라 가정하고 리버싱을 진행해본다
  - 첫 메뉴들을 프린트하는 부분 찾기 : 출력된 string으로 찾기
  ![b](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-03-08-mainprint.png)   
  - fgetchar()으로 메뉴 선택 입력값을 받는데 인접한 곳에서 숨겨진 메세지 박스와 관련된 string을 찾을 수 있었다
  - Hidden message 확인    
  ![c](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-03-08-hiddencode.png)
    - fgetchar()로 입력된 값은 5와 2번 비교됨
    - 첫번째 분기(0x402D6A)에서 점프하고 두번째 분기(0x402D75)에서 점프하지 않을 시 히든 메세지가 출력되는 함수(0x402D81)가 call됨
- ASLR 미적용 : 몇번 재실행해봐도 명령어들의 주소는 바뀌지 않음
  * ASLR : 메모리에 로딩될 때 ImageBase를 랜덤하게 바꾸는 기술


# 2. 메모리 변조를 통한 숨겨진 메시지 박스 표시
- run-time 메모리 변조 프로그램을 만들기  
  - 실행되고 중인 Target.exe 프로세스의 메모리를 변조해 Target.exe에서 5를 입력했을 때 프로그램이 종료되지 않고 숨겨진 메세지가 나오도록 만들었다.
- Target.exe에서 변조할 부분   

  ```arm assembly
  jne target.402D71   // 주소 : 00402D6A  기계어 : 75 05
  ```

  앞서 말한 첫번째 분기(0x402D6A)에서 무조건 점프하도록 위 코드를 다음과 같이 수정   

  ```arm assembly
  jmp target.402D71   // 주소 : 00402D6A  기계어 : EB 05
  ```

- WriteProcessMemory()로 0x402D6A 주소의 명령어를 바꿈
*Target.exe 변조 프로그램*
![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-03-08-success2.png)
*변조된 Target.exe*
![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-03-08-success.png)

# 3. MessageBoxW 함수 후킹
- MessageBoxW 함수의 위치 및 코드
![a](https://github.com/LeeHojun96/LeeHojun96.github.io/blob/master/_posts/img/2021-03-08-messageboxw.png)

### 3.1. MessageBoxW를 후킹하여 call될 때마다 로그가 출력되도록 변조

### 3.2. MessageBoxW에 전달되는 인자 변조



# 3. Target.exe 변조 프로그램 소스 코드
```c++
#include <windows.h>
#include <stdio.h>
#include <string.h>
#include <tlhelp32.h>
#include <tchar.h>
#include <processthreadsapi.h>


int main(void) {
	// 1. target 프로세스의 PID 찾기. 없으면 다시 1번
	HANDLE hTarget, hProcessSnap;
	PROCESSENTRY32 pe32;
	WCHAR targetName[] = L"Target.exe";
	LPCVOID lpBaseAddress;
	BYTE  rwBuffer[64] = { 0 };
	SIZE_T  nSize;
	int refresh = 1;
	int targetNotExist;

	while (refresh) {
		hProcessSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

		if (hProcessSnap == INVALID_HANDLE_VALUE)
		{
			printf("func failed : CreateToolhelp32Snapshot()");
			return(-1);
		}

		pe32.dwSize = sizeof(PROCESSENTRY32); // 명세서에 따르면 설정을 해주지 않으면 Process32First() 함수 fail
		if (!Process32First(hProcessSnap, &pe32))
		{
			printf("func failed : Process32First()");
			CloseHandle(hProcessSnap);          // clean the snapshot object
			return(FALSE);
		}

	// 2. target 프로세스 핸들을 얻기

		do {
			wprintf(L"PID : %d\t", pe32.th32ProcessID);
			wprintf(L"Name : %s\n", pe32.szExeFile);

      // 프로세스 이름이 target.exe인 것 찾기 
			targetNotExist = wcscmp(pe32.szExeFile, targetName);
			if (targetNotExist) {
				continue;
			}
			else {
				hTarget = OpenProcess(PROCESS_VM_READ | PROCESS_VM_WRITE | PROCESS_VM_OPERATION, false, pe32.th32ProcessID);
				if (!hTarget) {
					printf("func failed : OpenProcess()");
					break;
				}
				nSize = 2;
				lpBaseAddress = (LPVOID)0x402D6A;

				if (!ReadProcessMemory(hTarget, lpBaseAddress, rwBuffer, nSize, NULL)) {
					printf("func failed : ReadProcessMemory()");
					break;
				}
				printf("before \nRead memory : ");
				for (int i = 0; i < nSize; i++) {
					printf("%02hhx ", ((BYTE*)rwBuffer)[i]);
				}
				printf("( = jne target.402D71)\n");

	// 3. 0x00402D6A 주소의 메모리 overwrite : jne를 jmp로

				rwBuffer[0] = 0xEB;		// jne -> jmp로 수정

				if (!WriteProcessMemory(hTarget, (LPVOID)lpBaseAddress, rwBuffer, nSize, NULL)) {
					printf("func failed : ReadProcessMemory()");
					break;
				}
				if (!ReadProcessMemory(hTarget, lpBaseAddress, rwBuffer, nSize, NULL)) {
					printf("func failed : ReadProcessMemory()");
					break;
				}
				printf("after \nRead memory : ");
				for (int i = 0; i < nSize; i++) {
					printf("%02hhx ", ((BYTE*)rwBuffer)[i]);
				}

				break;
			}

		} while (Process32Next(hProcessSnap, &pe32));

		printf("\n");
		if (targetNotExist) {
			printf("No Target.exe process.");
		}
		printf("Refresh? (input 1 or 0)\n");
		scanf_s("%d", &refresh);

		CloseHandle(hProcessSnap);
	}

	return 0;
}
```
# 4. Target.exe 소스코드
```c++
#include <Windows.h>
#include <iostream>
#include <string> // for using std::getline
#include <stdio.h>

// 함수의 전방 선언
void SetMessage(std::wstring *_pMessage);
void SetCaption(std::wstring *_pCaption);
void CallMessageBoxW(LPCWSTR _lpText, LPCWSTR _lpCaption);

int main()
{
    std::wstring defaultText = L"The message to be displayed";
    std::wstring defaultCaption = L"The dialog box title";

    std::wstring text = defaultText; // 표시할 메시지
    std::wstring caption = defaultCaption; // 대화상자의 제목

    while (true)
    {
        int input;

        printf("1: Displays a message box.\n");
        printf("2: Set the meesage to be displayed.\n");
        printf("3: Set the dialog box title.\n");
        printf("4: Restore the default message and title.\n");
        printf("5: Quit.\n");

        // 사용자로부터 메뉴 번호를 입력받습니다.
        printf("> ");
        scanf_s("%d", &input);
        while (getchar() != '\n'); // 입력 버퍼 비우기

        // 반복문 탈출
        if (5 == input)
            break;

        if (5 == input)
        {
            // 절대 실행되지 않는 숨겨진 코드
            // 컴파일러는 최적화 옵션에 따라 이 코드를 제거할 수 있습니다.
            CallMessageBoxW(L"Hidden message!", L"SUCCESS");
        }

        // 메뉴 선택
        switch (input)
        {
        case 1: // 메시지 박스 출력
            CallMessageBoxW(text.c_str(), caption.c_str());
            break;
        case 2: // 표시할 메시지 설정
            SetMessage(&text);
            break;
        case 3: // 대화상자 제목 설정
            SetCaption(&caption);
            break;
        case 4: // 메시지 및 제목 복원
            text = defaultText;
            caption = defaultCaption;
            break;
        default: // 잘못된 입력
            printf("[Error] Invalid input.\n");
        }

        printf("\n");
    }

    return 0;
}

void SetMessage(std::wstring *_pMessage)
{
    std::cout << "Input the message to be displayed\n> ";
    std::getline(std::wcin, *_pMessage); // 사용자로부터 표시할 메시지를 입력받습니다.
}

void SetCaption(std::wstring *_pCaption)
{
    std::cout << "Input the dialog box title\n> ";
    std::getline(std::wcin, *_pCaption); // 사용자로부터 대화상자 제목을 입력받습니다.
}

void CallMessageBoxW(LPCWSTR _lpText, LPCWSTR _lpCaption)
{
    HWND hWnd = NULL; // 만들 메시지 박스의 소유자 창에 대한 핸들로, NULL이면 소유자 창을 가지지 않습니다.
    UINT uType = MB_OK; // OK 버튼을 가진 메시지 박스입니다.

    MessageBoxW(hWnd, _lpText, _lpCaption, uType); // Wide character 문자열을 사용하여 메시지 박스를 표시합니다.
}
```
