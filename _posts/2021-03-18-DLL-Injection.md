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

# 0. 개요
- DLL injection
  - 대상 프로세스 핸들 구하기 : OpenProcess()
  - 대상 프로세스 메모리에 inject할 DLL 경로 써주기 : VirtualAllocEx(), WriteProcessMemory()
  - DLL 로드에 사용될 함수인 LoadLibraryA() api의 주소 구하기
  - 대상 프로세스에 스레드 실행시킴 : CreateRemoteThread()


# 방지법

# DLL 코드
```c++

#include "pch.h"
#include "stdio.h"
#include "windows.h"


DWORD WINAPI ThreadProc(LPVOID lParam)
{
    printf("Injection success \n");

    return 0;
}

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    HANDLE hThread = NULL;

    switch (fdwReason)
    {
    case DLL_PROCESS_ATTACH:
        hThread = CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
        CloseHandle(hThread);
        break;
    }

    return TRUE;
}


```

# dll injection 코드
```c++
#include <windows.h>
#include <stdio.h>
#include <tlhelp32.h>
#include <tchar.h>
#include <processthreadsapi.h>

#define DLL_PATH L"C:\Users\HJun\source\repos\injectedDLL\Debug\injectedDLL.dll"


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


	while (true) {
		wprintf(L"Input injection target process' PID : ");
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
		do {
			wprintf(L"\tModuleID : %d\t", me32.th32ModuleID);
			wprintf(L"Base address : 0x%X\t", me32.modBaseAddr);
			wprintf(L"Name : %s\n", me32.szModule);
		} while (Module32Next(hModuleSnap, &me32));


		// 3. 타겟 프로세스 내 메모리를 할당하고 DLL 경로 입력
		// 3.1. 타겟 프로세스 내 메모리 할당
		HANDLE hThread;
		LPVOID pRemoteBuf;
		SIZE_T dwBufSize;
		HMODULE hMod;
		LPTHREAD_START_ROUTINE pThreadProc;

		dwBufSize = lstrlenW(DLL_PATH) + sizeof(WCHAR);
		pRemoteBuf = VirtualAllocEx(hProcess, NULL, dwBufSize, MEM_COMMIT, PAGE_READWRITE);

		if (!pRemoteBuf) {
			printf("func failed : VirtualAllocEx()\n");
			break;
		}
		// 3.2. 타겟 프로세스에 DLL 경로 입력
		if (!WriteProcessMemory(hProcess, pRemoteBuf, (LPVOID)DLL_PATH, dwBufSize, NULL)) {
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


		// 5.
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
		do {
			wprintf(L"\tModuleID : %d\t", me32.th32ModuleID);
			wprintf(L"Base address : 0x%X\t", me32.modBaseAddr);
			wprintf(L"Name : %s\n", me32.szModule);
		} while (Module32Next(hModuleSnap, &me32));



		// 6. 2로 돌아가기
		CloseHandle(hThread);
		CloseHandle(hProcess);
	}
	return 0;
}
```
