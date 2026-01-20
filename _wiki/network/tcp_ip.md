---
layout  : wiki
title   : TCP/IP
summary : 
date    : 2026-01-18 19:04:11 +0900
updated : 2026-01-18 21:42:32 +0900
tag     : TCP
resource: E2/C0249A7C97400FA78527FA2A10712C
toc     : true
public  : true
parent  : [[/network]]
latex   : false
---
* TOC
{:toc}

# TCP/IP

> **Transmission Control Protocol / Internet Protocol (TCP/IP)**는 Internet을 구성하는 핵심적인 **프로토콜 쌍([protocol](https://dsaint31.me/mkdocs_site/CE/ch06/ce06_2_01_history/#protocols) pair)** 임.

* TCP/IP는 Internet에 접속하고 통신하기 위해 사용되는 표준 프로토콜
* [Internet](https://dsaint31.me/mkdocs_site/CE/ch06/ce06_2_03_Internet/#internet)은 본질적으로 TCP/IP를 기반으로 구성된 거대한 네트워크라고 볼 수 있음.
* 이는 
	* [[/network/OSI]]의 **네트워크 계층(Layer 3)의 IP 프로토콜**과 
	* **전송 계층(Layer 4)의 TCP 프로토콜**을 함께 사용
	* 즉, IP 위에 TCP가 정의되는 구조를 가짐.

IP 프로토콜은 `192.168.31.34`와 같은 **IP Address**를 사용하고, **datagram이라 불리는 packet** 단위로 데이터를 전달하는 역할을 담당. 

반면 TCP 프로토콜은 이러한 packet들이 **순서대로, 빠짐없이 수신되도록 보장**함으로써 신뢰성 높은 데이터 송수신을 제공.

전 세계의 모든 컴퓨터와 기기가 Internet을 통해 서로 연결되어 통신하기 위해서는 

* 반드시 **TCP/IP라는 프로토콜 쌍을 사용해야 하며**, 
* 이를 위해 각 장비에는 고유한 **IP Address가 부여** 됨.

## IP 

Internet Protocol의 약어. 

데이터를 작은 패킷으로 나누어 목적지 주소(address 지정)를 붙이고, 여러 네트워크를 거쳐 최종 목적지까지 전달되도록 경로를 결정(routing)하는 네트워크 계층 프로토콜

OSI의 Network Layer(3계층) 에 속함: Transport Layer(4계층) 의 지시에 따라, 목적지 주소(IP Address)를 기준으로 데이터를 어디로 보낼지, 어떤 경로(Route)를 사용할지를 처리.

> 데이터를 어디 IP address로 어떤 route 로 보낼지를 담당.

* [IP Address 에 대한 자료](https://dsaint31.tistory.com/439)
* [IP 에 대한 자료들](https://dsaint31.tistory.com/tag/IP)

주의할 점은 인간은 IP address보다는 ***Domain Name*** 으로 목적지를 지정한다.
(이를 가능하게하는 Domain Name System은 응용계층에 속함)

* [Domain Name 과 Domain Name System](https://dsaint31.tistory.com/440)
 
## TCP 

[Transmission Control Protocol](https://dsaint31.me/mkdocs_site/CE/ch06/ce06_2_03_Internet/#transmission-control-protocolinternet-protocol-tcpip)의 약어

네트워크 상에서 데이터가 순서대로, 빠짐없이, 오류 없이 전달되도록 연결을 설정하고 흐름과 오류를 제어하는 전송(신뢰성 있는 전송) 계층 프로토콜.

OSI의 Transport Layer(4계층) 에 속함: **Network Layer**보다 상위 계층으로, 데이터를 어떻게 안전하고 정확하게 보낼지를 정의.

> 데이터를 어떻게 **잘** 보낼지를 담당.

Transport Layer 에는 TCP 와 UDP(User Datagram Protocol)가 유명.

### TCP vs. UDP

* TCP(Transmission Control Protocol): 
	* 데이터가 빠지지 않고 순서대로 도착해야 하는 경우에 주로 사용
	* 웹([HTTP/HTTPS](https://dsaint31.me/mkdocs_site/CE/ch06/ce06_2_03_Internet/#protocol)), 이메일, 파일 전송 등 대부분의 일반적인 인터넷 서비스에서 사용됨.
* UDP(User Datagram Protocol): 
	* 약간의 데이터 손실이 있어도 괜찮은 대신 지연이 최소화되어야 하는 경우에 주로 사용. 
	* 실시간 스트리밍, 화상 통화, 온라인 게임, [DNS](https://dsaint31.tistory.com/440) 등에서 널리 사용됨.


