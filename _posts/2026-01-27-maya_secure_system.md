---
layout: article
title: "maya_secure_system 악성코드 분석 보고서"
excerpt: "maya_secure_system 바이러스는 자기 자신을 Scene 파일을 통해 복제하는 Maya 바이러스로 userSetup.py를 변조, 악성코드를 설치한 뒤 C2 서버에 하드웨어 정보를 전송하고 RCE를 수행하는 악성코드이다."
date: 2026-01-27 00:42 +0900
author: SystemaHacker
categories: [Malware-Analysis]
tags: [Malware-Analysis, Maya, Malware, Security]
permalink: /malware-analysis/maya_secure_system
---

## 개요
`maya_secure_system 바이러스`는 자기 자신을 Scene 파일을 통해 복제하는 Maya 바이러스로 `userSetup.py`를 변조, 악성코드를 설치한 뒤 C2 서버에 하드웨어 정보를 전송하고 [RCE](https://www.cloudflare.com/ko-kr/learning/security/what-is-remote-code-execution/)를 수행하는 난독화된 악성코드이다.

난독화된 악성코드의 원본 샘플과 복화화된 코드를 보고 싶다면 [Github 저장소](https://github.com/SystemaHacker/Deobfuscated-maya_secure_system.py)에 업로드해놓았다.   
파일 목록은 다음과 같다.   

| File | Description |
|------|------------|
| [maya_secure_system.py](https://github.com/SystemaHacker/Deobfuscated-maya_secure_system.py/blob/main/maya_secure_system.py) | Original obfuscated malware sample (**DANGEROUS — DO NOT RUN**) |
| [maya_secure_system.deob.py](https://github.com/SystemaHacker/Deobfuscated-maya_secure_system.py/blob/main/maya_secure_system.deob.py) | Deobfuscated malware code for analysis |

---
## 침투 방식
출처가 불분명한 Maya scene 파일 열람을 통한 악성코드 감염

---
## 침해 지표
1. `%userprofile%/Documents/maya/[마야버전]/` 경로의 `userSetup.py` 변조 및 `[maya_secure_system.py]` 파일 생성.   
2. 이후 열람한 Maya scene 파일의 script node에 복제 스크립트인 `maya_secure_system_scriptNode`를 삽입    

---
## maya_secure_system.py 분석
### 난독화
1. 심볼 난독화   
	난독화된 import, 함수, 변수 심볼을 VSCode의 `Rename Symbol` (F2) 기능을 이용하여 복원하였다.
	![난독화된 import](/assets/malware-analysis/maya_secure_system/obfuscatedImports.png)   

2. 실행 순서 난독화   
	`if g_iStatus == 1:` 실행 이후 `g_iStatus`를 `2`로 설정하여 다음 loop에서는 `if g_iStatus == 2:`에 해당하는 코드가 실행되고 그다음에는 `if g_iStatus == 3:`... 1842번 반복한다.   
	`while` 루프 내 모든 `if` 블럭을 실행 순서에 맞게 정렬한 후 `if` 구문을 제거하는 방법으로 복원하였다.   
	![난독화된 실행순서](/assets/malware-analysis/maya_secure_system/obfuscatedCodeflow.png)

3. 더미코드   
	실제로 사용되지 않은 변수들을 제거하였다.
	![더미코드](/assets/malware-analysis/maya_secure_system/dummyCodes.png)

4. 암호화된 String과 주석   
	악성코드를 Maya로 실행한 후 암호화에 사용되는 2개의 `key` 값을 메모리에서 읽는 방법으로 동적 분석하였다.   
	![암호화된 String과 주석](/assets/malware-analysis/maya_secure_system/obfuscatedStrings.png)

### 기능
1. `userSetup.py` 변조 및 `maya_secure_system.py` 설치   
	`maya_secure_system.py`를 설치하고 `userSetup.py`에서 이를 불러오는 코드를 삽입하여 매번 마야 실행과 함께 악성코드가 작동하도록 변조
	![userSetup 변조](/assets/malware-analysis/maya_secure_system/infectUserSetup.png)

2. Heartbeat    
	5분에 한 번씩 C2 서버에 생존신고
	![Heartbeat sending every 5 minutes](/assets/malware-analysis/maya_secure_system/heartbeat.png)

3. [RCE](https://www.cloudflare.com/ko-kr/learning/security/what-is-remote-code-execution/)   
	Heartbeat 응답으로부터 Custom script를 받아 실행하는 모습   
	![난독화된 실행순서](/assets/malware-analysis/maya_secure_system/heartbeatRce.png)
	![난독화된 실행순서](/assets/malware-analysis/maya_secure_system/rce.png)

	Custom script 실행 결과를 서버에 보고하는 모습   
	![난독화된 실행순서](/assets/malware-analysis/maya_secure_system/rceReport.png)

4. Scene 파일 감염   
	현재 열려있는 scene 파일의 script node에 복제 스크립트인 `maya_secure_system_scriptNode`를 삽입   
	이후 다른 PC에서 해당 scene 파일을 열람하면 똑같은 방법으로 감염된다.

	![Scene 파일 감염](/assets/malware-analysis/maya_secure_system/infectScene.png)

---
## 대응 및 복구 방안
1. `%userprofile%/Documents/maya/[마야버전]/` 경로 내 `userSetup.py`, `maya_secure_system.py` 파일 제거

2. 감염된 scene 파일 내 script node를 점검하고 `maya_secure_system_scriptNode`를 제거