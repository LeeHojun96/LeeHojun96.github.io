# 1. main 함수
시작 명령어 : CALL main 
어떤 함수들이 사전 사후에 call되는지 : stub
__scrt_common_main_seh+0FA 에서 call됨

# 2. 함수 시작과 끝(main 등)
(1) caller
스택으로 argument 전달 
argument로 string 전달 
```assembly
PUSH OFFSET 00D33000	# ASCII "init value : %d"
```
(2) callee
```assembly
PUSH EBP	# 스택에 base pointer 백업
MOV EBP, ESP	# 스택 맨위 주소를 새로운 base point로 지정
PUSH ECX	# call한 함수의 위치 백업??
...
ADD ESP 8 	# 스택 포인터 초기화
XOR EAX, EAX 	# return 값 0을 EAX을 통해 전달
MOV ESP, EBP 	# 호출 전 stack pointer 값으로 복구
POP EBP 	# 스택에 백업해둔 base pointer값을 EBP로 복구
RETN
```

# 3.변수의 선언/할당/사용 (문자열, 포인터, 배열 등)
- int
```assembly
MOV DWORD PTR SS:[LOCAL.1], 0A 		# 스택에 초기화 값 10을 PUSH, LOCAL.1 = EBP-4
```
위 명령어가 PUSH와 다른 점?
- int * 
```assembly
LEA EAX,[LOCAL.2]		# LOCAL.2(선언한 변수)의 주소를 EAX로 
MOV DWORD PTR SS:[LOCAL.3],EAX  # EAX를 스택에 저장
```

# 4. PE의 image base와 주소가 다른 이유
ASLR 적용? 설정하는 법? 원리?
preferred base address가 이미 선점되어 있을 경우? 확인법? 다른 주소를 찾는 로직
