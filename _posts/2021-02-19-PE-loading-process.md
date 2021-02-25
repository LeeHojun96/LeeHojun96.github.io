---
layout : single
title: "PE loading process"
excerpt: "PE 파일이 메모리로 로딩되는 과정을 간단하게 정리"
toc: true
toc_sticky: true
 
date: 2021-02-19
last_modified_at: 2021-02-25
---


# 1. PE 파일 구조 분석

1) PE 파일을 ReadFile로 읽기
PE 구조의 이미지 사이즈를 구하고, rwx한 메모리 할당하기 위해서

# 2. 메모리에 파일의 섹션들 매핑

2) 프로세스를 위한 VAS(Virtual Address Space) 마련하고 excutable 모듈을 disk에서 VAS로 매핑
PE 구조의 이미지 사이즈를 구하고, rwx한 메모리 할당
application space를 0으로 채우기

3) 이미지를 preferred base address에 로드 시도 후 section들을 메모리에 매핑
loader는 section table을 보고 section의 RVA를 base address에 더한 주소에 각 section을 매핑

4) ImageBase에서 로드한 preferred base address가 load address랑 다르면 base relocation이 발생 
=> 해당 주소를 base address로 사용하지 못하는 상황의 경우
매핑 이후 relocation을 하는 이유?
load하려고 try한 단계에서 바로 하지 체크하지 못한 이유?

# 3. DLL 처리

4) import table이 확인하여 DLL을 프로세스의 주소 공간으로 매핑

5) 로더는 DLL의 export section을 검사하고 IAT는 내부값이 실제 import된 함수의 주소로 변경됨
symbol이 없으면 loader는 오류 출력

# 4. 파일 실행

6) 모든 모듈이 로드되면 실행 플로우는 entry point로 
