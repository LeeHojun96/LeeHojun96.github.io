---

layout : single
title:  "DLL Injection"
excerpt: "간단한 dll injection 실습"

categories:
  - Blog
tags:
  - [Windows, Hooking]

toc: true
toc_sticky: true

date: 2021-03-18
last_modified_at: 2021-03-19
---
<!--
원래 WinAPI Hooking 이후에 DX API Hooking을 하려고 했는데, 호출규약이 다르다는 것 말고는 큰 차이가 없어보여 계획을 변경했습니다.
6주차 과제는 "DLL Injection" 입니다.
임의의 타겟 프로세스에 원하는 모듈을 주입하시고 실제로 모듈이 잘 로딩되었는지 확인하는게 목적입니다. 모듈 로딩 여부는 프로세스 해커를 사용하거나 직접 만들었던 프로세스/모듈 정보를 열거하는 프로그램을 사용하시면 됩니다.

[필수1] CreateRemoteThread 함수를 사용하여 타겟 프로그램에 임의의 DLL을 인젝션합니다.
[선택1] 6주차 과제의 타겟 프로세스에 dll을 Injection하여 메모리를 패치함으로써 실행 흐름을 변경합니다.
[선택2] 선택과제를 하나 더 추가하면, 어떻게 DLL 인젝션을 방지하거나 탐지할 수 있을지 생각해보시면 좋을 것 같습니다

Link1 - API 후킹을 위한 Dll Injection 기법들: https://www.apriorit.com/dev-blog/679-windows-dll-injection-for-api-hooks
Link 2 - Dll Injection 개요: https://reversecore.com/m/38
Link 3 - CreateRemoteThread 함수를 사용한 Dll Injection: https://reversecore.com/m/40

dll injection은 다른 프로세스의 메모리에 데이터를 쓰거나 읽기 위한 목적으로 수행됩니다. 게임 핵이나 악성코드는 주로 메모리로부터 주요 정보를 빼오거나 실행 흐름, 데이터 등을 변조하기 위해 dll injection을 수행하며, 백신 또한 실시간 탐지 등을 목적으로 수행하는 경우가 많습니다.
-->

# 1. DLL injection 개념
- DLL injection이란
  - 다른 프로세스에 특정 DLL을 강제로 삽입하는 공격
  - DLL은 삽입된 프로세스의 메모리 접근권한을 가져 다양한 행위가 가능
- 응용 사례
  - 악성코드 : 브라우저 프로세스에 삽입되어 강제로 악성코드를 다운받는 공격, 백도어 설치, 키로깅 등의 공격 가능
  - API Hooking

# 2. DLL injection 과정

### 2.0. 개요

- 공격 대상이 되는 프로세스의 메모리 공간에 삽입하고자 하는 dll의 경로를 저장
- 강제로 LoadLibraryA()를 실행하게 만들어 저장된 경로의 dll을 inject : 이 과정에서 CreateRemoteThread()를 이용

### 2.1.  대상 프로세스 핸들 구하기 : OpenProcess()

- OpenProcess() api를 사용하여 공격 대상 프로세스의 핸들을 얻음
- 핸들은 추후 메모리 할당(VirtualAllocEx()) 및 메모리에 데이터 덮어쓰기(WriteProcessMemory()) api에 필요

### 2.2.  대상 프로세스 메모리에 inject할 DLL 경로 써주기 : VirtualAllocEx(), WriteProcessMemory()

- VirtualAllocEx() api를 사용해 공격 대상 프로세스에 inject할 dll 경로를 기록할 메모리를 확보
  - VirtualAllocEx()는 성공 시 할당한 메모리의 시작주소를 return
- WriteProcessMemory() api를 사용해 확보한 메모리에 dll 경로 쓰기
  - *32-bit 문자열 자료형으로 써야 함. 이유는 다음 단계에 설명*

### 2.3.  DLL 로드에 사용될 함수인 LoadLibraryA() api의 주소 구하기 : GetModuleHandle()

- LoadLibraryA() api는 kernel32.dll에 포함
- LoadLibraryA() api는 프로세스가 인자로 전달된 라이브러리를 로딩
  - *인자로 전달될 library file name은 32-bit 문자열 자료형이여 함*
- 추후 CreateRemoteThread()로 LoadLibraryA()를 실행시킬 때 이 api의 주소가 필요
- GetModuleHandle("kernel32.dll")로 프로세스 내 kernel32.dll 주소를 구할 수 있음   
  ***이 주소는 다른 프로세스에서도 항상 동일한 값***
****
**GetModuleHandle()로 얻은 kernel32.dll 주소가 항상 같은 이유**
- GetModuleHandle()로는 공격 대상 프로세스가 아닌 지금 실행하는 프로세스에 로딩된 kernel32.dll 주소를 얻는 것
- 이것이 유효한 이유는   
  (1) 모든 Windows 프로세스는 kernel32.dll을 로딩하고   
  (PE header를 조작해 IAT에서 kernel32.dll을 제거해도 loader가 강제로 로딩함)   
  (2) Windows OS는 이 라이브러리를 프로세스마다 같은 주소에 로딩하기 때문
    - MS에서는 OS 핵심 DLL 파일들의 ImageBase 값을 정리해뒀고 이 때문에 자신들끼리 겹치지 않고 따라서 dll relocation이 발생하지 않음  
****

### 2.4.  대상 프로세스에 스레드 실행시킴 : CreateRemoteThread()

- CreateRemoteThread()는 다른 프로세에게 스레드를 실행시켜 주는 api
```c++
HANDLE WINAPI CreateRemoteThread(
  __in   HANDLE                   hProcess,             // 프로세스 핸들
  __in   LPSECURITY_ATTRIBUTES    lpThreadAttributes,
  __in   SIZE_T                   dwStackSize,
  __in   LPTHREAD_START_ROUTINE   lpStartAddress,       // 스레드 함수 주소
  __in   LPVOID                   lpParameter,          // 스레드 파라미터 주소
  __in   DWORD                    dwCreationFlags,
  __out  LPDWORD                  lpThreadId
);
```
- 위 파라미터 중 스레드 함수 주소로 LoadLibraryA() api를 전달하고 스레드 파라미터 주소로 2단계에서 삽입한 dll의 주소를 전달
- 다음 코드가 실행되어 dll이 로딩됨
```c++
LoadLibraryA( ..., "./malicious_DLL.dll" , ...)
```
# 3. 탐지 및 방지법
- 모듈이 로딩되면 정상적인 모듈인지 snapshot을 찍어 확인
- loadLibary 같이 나한테 로딩되어있는 DLL 내 주요 api를 손상시켜놨다가 (memorywrite 등으로) 내가 사용할때만 복구시키고 사용    
- 파일 입출력할 때마다 실행되는 드라이버인 MiniFilter Driver가 접근하는 파일이 정상적인 파일인지 whitelist 등을 확인하는 방법
<!--
https://exchangeinfo.tistory.com/68
-->
# 4. 다른 injection 공격 : Manual Mapping
- 개념
  - LoadLibrary가 수행하는 모든 작업을 에뮬레이션하여 PE loader가 dll 적재하는 걸 에뮬레이션함
  - 즉, OS 몰래 dll을 어느 영역에 숨겨둘 수 있음
  - PE loader를 만들고 만든 pe loader를 실행시키는 쉘코드를 대상 프로세스에 인젝션하는 프로그램도 만들어야 함
- 주의점
  - 여러 dll이 로드되다가 숨겨진 dll이 있는 부분이 overwrite될 수도 있음
- 탐지법
  - 어디에 있든 모듈이 실행되면 쓰레드가 무조건 돌게 되는데 이 쓰레드의 시작점을 알 수 있음
  - 이 시작점이 정상적인 모듈영역에 있는지를 가지고 비정상 모듈인지를 판단
  - 해당 탐지법의 한계 : 일부 백신도 manual mapping되어 동작할 수 있ㄴ는데 백신을 불법프로그램으로 잘못 인식할 수 있음





# 5. DLL 코드
- 로드되면 메세지 박스를 띄움
```c++
#include "windows.h"
#include "stdio.h"

BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        MessageBoxA(NULL, "DLL injected!", "dll", MB_OK);
        break;
    }
    return TRUE;
}
```

# 6. dll injection 코드
```c++
#include <windows.h>
#include <stdio.h>
#include <tlhelp32.h>
#include <tchar.h>
#include <processthreadsapi.h>

#define lpDllPath "C:\\Users\\HJun\\source\\repos\\injectedDLL\\Debug\\injectedDLL.dll"

int main(void)
{
	HANDLE hProcessSnap;
	PROCESSENTRY32 pe32;

	HANDLE hModuleSnap = INVALID_HANDLE_VALUE;
	MODULEENTRY32 me32;

	// 1. 프로세스 정보 나열 : 이름, PID, base address
	hProcessSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

	if (hProcessSnap == INVALID_HANDLE_VALUE)
	{
		printf("func failed : CreateToolhelp32Snapshot() - process\n");
		return(-1);
	}

	pe32.dwSize = sizeof(PROCESSENTRY32); // 명세서에 따르면 설정을 해주지 않으면 Process32First() 함수 fail
	if (!Process32First(hProcessSnap, &pe32))
	{
		printf("func failed : Process32First()\n");
		CloseHandle(hProcessSnap);          // clean the snapshot object
		return(FALSE);
	}

	do {
		wprintf(L"PID : %d\t", pe32.th32ProcessID);
		wprintf(L"Name : %s\n", pe32.szExeFile);
	} while (Process32Next(hProcessSnap, &pe32));

	CloseHandle(hProcessSnap);

	// 2. 프로세스 선택
	int pid = 0;
	HANDLE hProcess;
	LPCVOID lpBaseAddress;
	BYTE  readBuffer[64] = { 0 };
	SIZE_T  nSize;
	//WCHAR lpDllPath[] = L"C:\\Users\\HJun\\source\\repos\\injectedDLL\\Debug\\injectedDLL.dll";


	HANDLE hThread;
	LPVOID pRemoteBuf;
	SIZE_T dwBufSize;
	HMODULE hMod;
	LPTHREAD_START_ROUTINE pThreadProc;

	while (true) {
		wprintf(L"Input injection target process' PID : \n");
		scanf_s("%d", &pid);
		if (pid == -1) {
			break;
		}
		hProcess = OpenProcess(PROCESS_VM_READ | PROCESS_VM_WRITE | PROCESS_VM_OPERATION, false, (DWORD)pid);
		if (!hProcess) {
			printf("func failed : OpenProcess()\n");
			break;
		}

	//2.2 프로세스 내 모듈 print

		me32.dwSize = sizeof(MODULEENTRY32);

		hModuleSnap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE | TH32CS_SNAPMODULE32, (DWORD)pid);
		if (hModuleSnap == INVALID_HANDLE_VALUE)
		{
			printf("func failed : CreateToolhelp32Snapshot() _ module\n");
			printf("%d\n", GetLastError());
			continue;
		}

		if (!Module32First(hModuleSnap, &me32))
		{
			printf("\tfunc failed : Module32First()\t");
			CloseHandle(hModuleSnap);          // clean the snapshot object
			printf("errcode = %d \n", GetLastError());
			continue;
		}
		int cnt = 1;
		do {
			printf("\t%d", cnt);
			cnt++;
			wprintf(L"\tModuleID : %d\t", me32.th32ModuleID);
			wprintf(L"Base address : 0x%X\t", me32.modBaseAddr);
			wprintf(L"Name : %s\n", me32.szModule);
		} while (Module32Next(hModuleSnap, &me32));


		// 3. 타겟 프로세스 내 메모리를 할당하고 DLL 경로 입력
		// 3.1. 타겟 프로세스 내 메모리 할당

		dwBufSize = strlen(lpDllPath) + sizeof(CHAR);
		pRemoteBuf = VirtualAllocEx(hProcess, NULL, dwBufSize, MEM_COMMIT, PAGE_READWRITE);
		if (!pRemoteBuf) {
			printf("func failed : VirtualAllocEx()\n");
			break;
		}
		// 3.2. 타겟 프로세스에 DLL 경로 입력
		if (!WriteProcessMemory(hProcess, pRemoteBuf, (LPVOID)lpDllPath, dwBufSize, NULL)) {
			printf("func failed : WriteProcessMemory()\n");
			break;
		}


		// 4. DLL 로드에 사용될 함수인 LoadLibraryA() api의 주소 구하기 : 어느 프로세스에서나 주소가 같음
		hMod = GetModuleHandle(L"kernel32.dll");
		if (!hMod) {
			printf("func failed : GetModuleHandle()\n");
			break;
		}
		pThreadProc = (LPTHREAD_START_ROUTINE)GetProcAddress(hMod, "LoadLibraryA");
		if (!pThreadProc) {
			printf("func failed : GetProcAddress()\n");
			break;
		}

		hThread = CreateRemoteThread(hProcess, NULL, 0, pThreadProc, pRemoteBuf, 0, NULL);
		if (!hThread) {
			printf("func failed : CreateRemoteThread()\n");
			break;
		}

		WaitForSingleObject(hThread, INFINITE);
		printf("\n");

		// 5. DLL 삽입되었는지 확인
		hModuleSnap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, (DWORD)pid);
		if (hModuleSnap == INVALID_HANDLE_VALUE)
		{
			printf("func failed : CreateToolhelp32Snapshot() _ module\n");
			continue;
		}

		if (!Module32First(hModuleSnap, &me32))
		{
			printf("\tfunc failed : Module32First()\t");
			CloseHandle(hModuleSnap);          // clean the snapshot object
			printf("errcode = %d \n", GetLastError());
			continue;
		}
		cnt = 1;
		do {
			printf("\t%d", cnt);
			cnt++;
			wprintf(L"\tModuleID : %d\t", me32.th32ModuleID);
			wprintf(L"Base address : 0x%X\t", me32.modBaseAddr);
			wprintf(L"Name : %s\n", me32.szModule);
		} while (Module32Next(hModuleSnap, &me32));

		// 6. 2로 돌아가기
		CloseHandle(hThread);
		CloseHandle(hProcess);
		break;
	}
	return 0;
}
```
