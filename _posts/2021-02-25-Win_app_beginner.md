---

layout : single
title:  "Win Application 기본"
excerpt: "기본적인 Win app 개발 샘플 정리"

categories:
  - Blog
tags:
  - [Windows]

toc: true
toc_sticky: true

date: 2021-02-25
last_modified_at: 2021-02-26
---
<!--
과제
4주차 과제는 Windows 데스크톱 앱 개발로 5주차 과제에 선행하여 필요합니다.
- [필수 1] 직접 프로젝트를 생성하거나 샘플을 다운받아서 Windows 데스크톱 앱을 만듭니다. (Link 1, 2)
- [필수 2] 소스 코드를 설명합니다. 주석을 달아도 되고 소스코드와 별도로 각 요소들을 설명하는 글을 작성해도 좋습니다. 함수 정보는 Microsoft Docs에서 검색해주세요.
- [선택 1] Windows의 창에 대해 배우고 이를 응용하여 그림을 그리거나 창을 닫기 전 알림을 띄우는 등의 코드를 작성해봅니다.
- [선택 2] 작성한 Windows 데스크톱 앱을 x86-32 디버거(Visual Studio 내장 디버거, x64dbg 등)를 사용하여 소스코드와 비교하며 관찰합니다.
- [선택 3] Windows의 클래스, 프로시저, 메시지 등에 대해 공부합니다. (Link 4)

Link 1 - 최소한의 Windows 데스크톱 프로그램 작성: https://docs.microsoft.com/en-us/windows/win32/learnwin32/your-first-windows-program
Link 2 - Windows Hello World 샘플 다운로드: https://docs.microsoft.com/en-us/windows/win32/learnwin32/windows-hello-world-sample
Link 3 - Win32 API 문서 (기능 기준): https://docs.microsoft.com/en-us/windows/win32/apiindex/windows-api-list
Link 4 - Windows 개요: https://docs.microsoft.com/en-us/windows/win32/winmsg/windows
-->
# 0. 개요

## 1) 목표

- 가장 기본적인 Windows application의 소스코드 분석

## 2) 분석 대상

- 분석할 샘플 프로그램 소스

  - minimal Windows desktop program : https://docs.microsoft.com/en-us/windows/win32/learnwin32/your-first-windows-program

- 샘플 프로그램 기능 : 사용자가 종료 전까지 빈 윈도우 창을 띄움.

# 1. 사용할 문자셋(character set) 정의     

```cpp
#ifndef UNICODE
#define UNICODE
#endif

#include <windows.h>
```

## 1) Windows 스타일 자료형이 정의되어 있는 'windows.h'

  - CHAR, WCHAR, LPSTR 등이 정의되어 있음

## 2) 매크로 UNICODE와 같이 사용하여 WBCS의 지원하는 'windows.h'

- WBCS(Wide Byte Character Set) : 모든 문자를 2바이트로 표현하는 방식. 보통 유니코드 기반으로 함.
- MBCS(Multi Byte Character Set) : 다양한 바이트 수를 사용해 문자를 표현하는 방식. 보통 아스키코드에서 정의하고 있는 문자들은 1바이트로, 아스키코드에서 정의하지 않는 다른 문자들은 2바이트로 표현.

- windows.h 헤더에는 WBCS와 MBCS를 동시에 수용하는 형태의 프로그램을 위해 매크로가 정의되어 있음.   
=> 매크로 UNICODE가 정의되었는가 아닌가에 따라 코드가 다른 형태로 치환됨.

  - TCHAR, LPTSTR 등의 T가 붙은 자료형이 매크로 UNICODE가 정의되었는가 아닌가에 따라 다르게 바뀜.   
  단, T자료형은 tchar.h은 windows.h에 포함되어 있지 않으므로 따로 include되어야 함.
  ```cpp
  #ifdef UNICODE
    typedef WCHAR TCHAR;
    typedef LPWSTR LPTSTR;
  #else
    typedef CHAR TCHAR;
    typedef LPSTR LPTSTR;
  #endif
  ```
  - string의 형태가 매크로 UNICODE가 정의되었는가 아닌가에 따라 다르게 바뀜
  ```cpp
  #ifdef UNICODE
    typedef WCHAR TCHAR;
    typedef LPWSTR LPTSTR;
  #else
    typedef CHAR TCHAR;
    typedef LPSTR LPTSTR;
  #endif
  ```  


# 2. main 함수 부분

## 1) 전체 main 함수 형태

```cpp
int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, PWSTR pCmdLine, int nCmdShow)
{
  ...
}
```

- wWinMain()
  - WinMain()은 윈도우 어플리케이션의 시작 함수로 콘솔 애플리케이션의 시작 함수 main() 같은 것.
  - wWinMain()은 wmain()과 유사한 것으로 WBCS 방식(ex. 유니코드 방식)을 지원하는 형태   
  = main 함수에 전달되는 인자를 유니코드 기반으로 구성함(ex. L"test.exe")
  - 프로그램의 *entry point*

## 2) main 함수 내 세부 단계

#### (1) window class 생성 단계

```cpp
  // Register the window class.
    const wchar_t CLASS_NAME[]  = L"Sample Window Class";

    WNDCLASS wc = { };

    wc.lpfnWndProc   = WindowProc;
    wc.hInstance     = hInstance;
    wc.lpszClassName = CLASS_NAME;

    RegisterClass(&wc);
```

- Window class 개념

  - 만들어질 윈도우 창의 여러 특성들을 정의하는 구조체   
  ex) 버튼의 수, 메시지 처리 함수, 스타일 등  
  - WNDCLASS 구조체 인스턴스로 구현.
  - Window class는 C++에서의 class와 다른 개념으로 OS에서 사용되는 데이터 구조체.

- WNDCLASS 구조체

  ```cpp
  typedef struct tagWNDCLASSA {
  UINT      style;
  WNDPROC   lpfnWndProc;
  int       cbClsExtra;
  int       cbWndExtra;
  HINSTANCE hInstance;
  HICON     hIcon;
  HCURSOR   hCursor;
  HBRUSH    hbrBackground;
  LPCSTR    lpszMenuName;
  LPCSTR    lpszClassName;
  } WNDCLASSA, *PWNDCLASSA, *NPWNDCLASSA, *LPWNDCLASSA;
  ```

  - 샘플 소스에 제시된 멤버변수 설명

    - lpfnWndProc : 윈도우의 메세지 처리 함수 지정. 메세지가 발생할 때마다 이 멤버가 지정하는 함수가 호출되며 이 함수가 모든 메시지를 처리
    - hInstance : 이 윈도우 클래스를 등록하는 '프로그램의 번호'이며 WinMain의 인수로 전달된 hInstance값을 그대로 대입하면 됨.   
    OS는 이 번호를 통해 어떤 프로그램에서 등록했는지 기억해두었다가 프로그램이 종료되면 이 클래스의 등록을 취소함.    
    - lpszClassName : 윈도우 클래스의 이름을 문자열로 정의. 여기서 지정한 이름은 후에 CreateWindow 함수에 전달하여 사용함.

- RegisterClass 함수로 생성한 window class 등록
RegisterClass 함수의 인수로 WNDCLASS 구조체의 주소를 전달. '이런 특성을 가진 윈도우를 앞으로 사용하겠다'는 등록 과정이며 OS는 이 윈도우 클래스의 특성을 기억해둠.

#### (2) 윈도우 창 생성 단계

```cpp
    // Create the window.

    HWND hwnd = CreateWindowEx(
        0,                              // Optional window styles.
        CLASS_NAME,                     // Window class
        L"Learn to Program Windows",    // Window text
        WS_OVERLAPPEDWINDOW,            // Window style

        // Size and position
        CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT,

        NULL,       // Parent window    
        NULL,       // Menu
        hInstance,  // Instance handle
        NULL        // Additional application data
        );
```

- CreateWindowEx 함수로 윈도우 창을 생성하고 그 핸들을 변수 hwnd에 저장
- CreateWindowEx()

  ```cpp
  HWND CreateWindowExA(
  DWORD     dwExStyle,
  LPCSTR    lpClassName,
  LPCSTR    lpWindowName,
  DWORD     dwStyle,
  int       X,
  int       Y,
  int       nWidth,
  int       nHeight,
  HWND      hWndParent,
  HMENU     hMenu,
  HINSTANCE hInstance,
  LPVOID    lpParam
  );
  ```
  - dwExStyle : 창을 투명하게 한다던가하는 '스타일에 대한 옵션'을 부여할 수 있음. 0을 전달할 경우 default 스타일로 설정됨
  - lpClassName : RegisterClass 함수로 등록한 window class를 전달.
  - lpWindowName : 윈도우 창의 타이틀 바에 들어갈 문자열.
  - dwStyle : 윈도우 창의 스타일을 정함.    
  ex) 수직 수평 스크롤바 유무, 팝업 윈도우 스타일 등
  - X, Y : 윈도우 창의 초기 위치에 대한 설정
  - nWidth, nHeight : 윈도우의 크기에 대한 설정
  - hWndParent : pop-up 윈도우를 위한 옵션으로 이 창을 생성한 부모 윈도우의 핸들값을 전달.
  - hMenu : 메뉴에 대한 설정값.
  - hInstance : main 함수에서 전달받은 '프로그램 번호'로 window class에서 사용된 것과 동일.
  - lpParam : 원하는 데이터를 파라미터로 넘길 때 사용됨. 추후에 상술

```cpp

    if (hwnd == NULL)
    {
        return 0;
    }

    ShowWindow(hwnd, nCmdShow);
```

- 오류가 발생해 생성한 윈도우 창의 핸들값이 NULL일 경우 프로그램 종료
- 핸들이 정상적으로 전달됐으면 ShowWindow()로 윈도우 창을 사용자에게 보여줌
  - nCmdShow : 윈도우의 최대화, 최소화할 때 사용되는 값으로 main 함수의 인자를 통해 전달됨.

#### (3) 메세지 루프 부분

```cpp
    // Run the message loop.

    MSG msg = { };
    while (GetMessage(&msg, NULL, 0, 0))
    {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    return 0;
```

- window message 개념

  - 언제 발생할지 예측할 수 없는 유저와 OS로부터의 이벤트에 대응하기 위해 등장.   
  ex) 유저의 마우스 클릭, 윈도우 OS의 sleep 모드 등의 이벤트
  - windows OS는 다양한 이벤트마다 대응하는 메세지를 상수값으로 정의함.   
  ex)

    ```cpp
    #define WM_LBUTTONDOWN    0x0201  # 마우스 좌클릭에 해당하는 메세지 코드
    ```

- window message 관리

  - MSG 구조체로 관리됨
  - OS는 윈도우를 만든 쓰레드마다 메세지 큐를 만들어 관리함.

- GetMessage()

  - 앞서 말한 메세지 큐에서 하나의 메세지를 POP하여 첫번째 파라미터 msg에 저장하는 함수

- TranslateMessage(&msg);

  - 키보드 입력과 관련된 함수로 key stroke을 character로 바꿔줌
  - DispatchMessage() 이전에 call됨

- DispatchMessage(&msg);

  - OS로 하여 window procedure를 call하도록 함.
  - 위에 있는 WindowProc를 콜하는 것

# 3. Window procedure

```cpp
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch (uMsg)
    {
    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;

    case WM_PAINT:
        {
            PAINTSTRUCT ps;
            HDC hdc = BeginPaint(hwnd, &ps);



            FillRect(hdc, &ps.rcPaint, (HBRUSH) (COLOR_WINDOW+1));

            EndPaint(hwnd, &ps);
        }
        return 0;

    }
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}
```

## 1) WindowProc() 구조

- 파라미터

  - hwnd : 윈도우의 핸들
  - uMsg : 전달받을 메세지 코드
  - wParam과 lParam : 메세지 코드에 따라 의미가 달라지는, 추가적인 데이터들

## 2) WindowProc() 내용

- 위 DispatchMessage() 함수로 call될 window procedure를 정의
- switch 문

  - 입력받은 메세지 코드에 따라 분기됨
  - WM_DESTROY

    - 윈도우가 파괴(destroy)되었을 때 전달되는 메세지 코드
    - PostQuitMessage() 함수 : 시스템에게 쓰레드가 종료 요청을 보냈음을 알려주는 함수. 보통 VM_DESTROY와 같이 쓰임

  - WM_PAINT

    - 윈도우에서의 paint 의미 : 윈도우 창 안에서 무언가를 보여주는 것
    - 윈도우를 paint하라고 보내는 메세지 코드 :    
    예시)   

      - 윈도우 창을 만들면서 OS는 이 창을 paint하라고 사용자에게 WM_PAINT 메세지를 보냄. 즉, 윈도우를 show하려면 사용자는 최소 한번 이상의 WM_PAINT 메세지를 받게 되는 것.   
      - 창을 업데이트하면서 창의 일부를 다시 repaint하라고 해당 메세지 코드를 보내기도 함.
      - 다른 창에 가려져있던 일부가 드러나게 되는 경우 update하기 위.

    - BeginPaint()와 EndPaint()
      painting 연산의 시작과 끝을 선언. 이 사이에서 모든 painting 작업이 이뤄져야 함.
    - FillRect(hdc, &ps.rcPaint, (HBRUSH) (COLOR_WINDOW+1));

      - HBRUSH 구조체라는 브러쉬로 사각형을 색칠하는 함수
      - rcPaint는 PAINTSTRUCT의 멤버변수로 사각형의 좌표에 대한 정보
      - 위 코드의 경우 update된 전체 영역을 rcPaint로 넘김
        ```cpp
        PAINTSTRUCT ps;   //멤버변수 rcPaint의 초기화값이 창의 좌표로 추정
        HDC hdc = BeginPaint(hwnd, &ps);
        FillRect(hdc, &ps.rcPaint, (HBRUSH) (COLOR_WINDOW+1));
        ```

- DefWindowProc() 부분

  - default message handler
  - 따로 처리하지 않을 몇몇 메세지들을 이 함수로 넘겨 디폴트대로 처리함.

# 4. 정리

## 1) 윈도우 프로그래밍의 기본 형태

  - 윈도우 클래스 지정 (WNDCLASS)
  - 윈도우 클래스 등록 (RegisterClass)
  - 윈도우 생성 및 업데이트 (CreateWindow 및 UpdateWindow)
  - 메시지 루프 (GetMessage 루프)
  - 메시지 처리함수 작성 (WindowProc)

## 2) 참조
https://docs.microsoft.com/en-us/windows/win32/learnwin32/creating-a-window
https://docs.microsoft.com/en-us/windows/win32/learnwin32/window-messages
