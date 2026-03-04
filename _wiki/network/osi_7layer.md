---
layout  : wiki
title   : Open System Interconnection (OSI) 7-layer
summary : 
date    : 2026-01-18 20:59:09 +0900
updated : 2026-01-18 22:24:29 +0900
tag     : OSI network
resource: 51/4EC8DFE6BD4BE2AE1156696B9450AB
toc     : true
public  : true
parent  : [[/network]]
latex   : false
---
* TOC
{:toc}

# OSI

Open System Interconnection 의 약어.

OSI 7계층 모델은 네트워크 아키텍처의 표준 [protocol](https://dsaint31.me/mkdocs_site/CE/ch06/ce06_2_01_history/#protocols)임.

## Architecture

![]()

```
┌─────────────────────────────┐
│ 7. Application Layer        │  ← 사용자·응용 프로그램
├─────────────────────────────┤
│ 6. Presentation Layer       │  ← 데이터 표현 형식 변환
├─────────────────────────────┤
│ 5. Session Layer            │  ← 통신 세션 관리
├─────────────────────────────┤
│ 4. Transport Layer          │  ← 종단 간 전송 특성
├─────────────────────────────┤
│ 3. Network Layer            │  ← IP 기반 경로 선택
├─────────────────────────────┤
│ 2. Data Link Layer          │  ← 프레임 전달 (LAN)
├─────────────────────────────┤
│ 1. Physical Layer           │  ← 전기·무선 신호
└─────────────────────────────┘
```

* 위로 갈수록 추상적 개념, 아래로 갈수록 물리적 구현에 가까움
* 실제 데이터 전송 시
	* 송신 측: 7 → 1 계층 순서로 내려가며 캡슐화(encapsulation)
	* 수신 측: 1 → 7 계층 순서로 올라가며 역캡슐화(decapsulation)
* 각 계층은 바로 아래 계층의 서비스만 사용하며, 내부 구현은 숨겨짐


## Example

### 7. 응용 계층 (Application Layer)

게임 클라이언트 자체가 동작하는 계층으로, **수신된 데이터를 의미 있는 게임 명령으로 해석하고 게임 로직을 실행**.

예를 들어 "챔피언이 Q 스킬을 사용했다", "좌표 (x, y)로 이동한다"는 데이터를 이해해 **상태를 계산하고 화면 갱신을 지시**.


### 6. 표현 계층 (Presentation Layer)

응용 계층이 이해할 수 있도록 **데이터의 표현 형식만 변환**하는 계층.

네트워크로 받은 바이트 스트림을 구조화된 데이터로 복원(역직렬화)하거나, **인코딩 변환·압축 해제·암호화 해제**를 수행.


### 5. 세션 계층 (Session Layer)

클라이언트와 게임 서버 사이의 **논리적인 연결 상태를 관리**.

게임 시작부터 종료까지의 세션을 유지하고, 일시적인 연결 끊김 시 **재접속하여 같은 게임 흐름을 이어갈 수 있게** 함.


### 4. 전송 계층 (Transport Layer)

[[/network/tcp_ip#TCP]] 에서 TCP가 Transport Layer (Layer 4)임.

클라이언트와 서버 간 데이터 전달의 **전송 방식과 특성**을 담당: 데이터 전송단위가 segment임.

LoL과 같은 게임에서는 반응성이 매우 중요하므로 **주로 UDP 기반 전송**을 사용하여 지연을 최소화.


### 3. 네트워크 계층 (Network Layer)

[[/network/tcp_ip#IP]] 에서 IP 가 Network Layer (Layer 3)임.

내 PC에서 라이엇 게임 서버까지 **[IP address](https://dsaint31.tistory.com/439)를 기준으로 packet이 이동할 경로를 결정**: 데이터 전송 단위가 packet(=IP datagram)임.

어느 서버로, 어떤 [라우터](https://dsaint31.tistory.com/449)들을 거쳐 갈지가 이 계층에서 결정됨.

### 2. 데이터 링크 계층 (Data Link Layer)

PC와 [공유기](https://dsaint31.tistory.com/226#%EA%B3%B5%EC%9C%A0%EA%B8%B0%20(NAT%20%EC%9E%A5%EB%B9%84)-1-3), 공유기와 ISP 장비 사이에서 **frame 단위 통신과 오류 검출**을 담당: 데이터 전송단위가 frame.

참고로 frame의 payload가 바로 IP packet으로 packet을 **encapsulation** 하고 있음:

```text
[ Destination MAC ]
[ Source MAC      ]
[ Type            ]
[ Payload (IP packet) ]
[ FCS (오류 검출) ]
```

* FCS: Frame Check Sequence: 오류 검출용 필드로 32bit [CRC (Cyclic Redundancy Check)](https://dsaint31.me/mkdocs_site/CE/ch03_seq/ce03_04_ecc_memory/?h=crc#checksum-and-crc)를 사용.

같은 네트워크 구간([LAN](https://dsaint31.me/mkdocs_site/CE/ch06/ce06_2_01_history/#lan-vs-wan))에서의 안정적인 전달이 이 계층의 역할임.

[MAC address](https://dsaint31.tistory.com/227)를 이용하는 [Switch](https://dsaint31.tistory.com/226#Switch-1-1)가 여기에 해당.

### 1. 물리 계층 (Physical Layer)

유선 LAN, Wi-Fi, 광케이블을 통해 **전기적·무선 신호가 실제로 전달**되는 단계: 데이터 전송단위가 1bit.

케이블 불량이나 무선 간섭은 바로 지연이나 끊김으로 나타남.  

모든 연결될 호스트에 같은 전기신호를 보내는 [Hub](https://dsaint31.tistory.com/226#Hub-1) 가 여기에 해당.

## 같이 보면 좋은 자료들

* [Hug, Switch, Router, 공유기](https://dsaint31.tistory.com/226)
* [MAC Address](https://dsaint31.tistory.com/227)


