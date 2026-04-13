---
layout  : wiki
title   : GPG 관련 용어
summary : 
date    : 2026-01-18 15:04:08 +0900
updated : 2026-01-28 14:49:09 +0900
tag     : gpg
resource: 17/8DC5A0BEDB4C8783BC8168B5ABD928
toc     : true
public  : true
parent  : [[/gpg]]
latex   : false
---
* TOC
{:toc}

# Authentication(인증), Authorization(인가), Public Keys (공개키), Certificates (인증서) 개념 정리

(GPG 명령어 예제 포함, Primary Key–Subkey 구조 반영)

## 1. Public Key와 Private Key

### 정의

* **Public Key**: 외부에 공개되는 키
* **Private Key**: 소유자만 보관하는 키

### 기본 원리

Public-key cryptography (공개키 암호)는 항상 **키 쌍(key pair)** 으로 구성된다.

| 기능    | 사용되는 키 |
| ----- | ------ |
| Encryption(암호화)   | 공개키    |
| Decryption(복호화)   | 비밀키    |
| Signature Creation | 비밀키    |
| Signature Verification (서명 검증) | 공개키    |

* Priavate key는 외부로 노출되어서는 안 되며, 
* Public key는 배포될 수 있다.

### GPG 예제: 키 쌍 생성

```bash
gpg --full-generate-key
```
* private key는 signing 과 decrpytion에 사용됨.
* 대응되는 public key는 verification과 encryption에 사용됨.

## 2. 사전 단계: 공개키 배포(Distribution)

Signature verification은 상대가 **public key를 알고 있을 때만** 성립.

* 즉, 먼저 public key를 배포하거나 상대방에 전달하는 단계가 필요: key-server 사용([[/gpg/gpg_keyserver]]
* 상대방의 public key를 얻기 위해 [[/gpg/gpg_keyserver]]를 이용할 수 있음.

다음은 public key를 export하는 명령어임.

```bash
gpg --export --armor user@example.com > public.asc
```

* `*.asc` 는 ascii를 의미함.
* armor란 텍스트 기반 통신을 위해 ascii 문자들로 이루어진 base64 인코딩으로 사용함을 의미.

이 공개키는 이후 signature verification 과 [Authentication](#6-authentication--authenticate-인증) 에서 사용됨.


## 3. Identification / Identify (식별)

### 정의

* **Identification**: 주체가 **자신이 누구인지 주장(claiming an identity)**하는 단계
* **Identify**: 그 주장을 제시하는 행위 (to present that claim)

### 개념

이 단계에서는 아직 신뢰(trust)가 생성되지 않는다.

단지 "나는 이 공개키의 소유자다" 라는 주장이 제시된다.

다음은 gpg가 관리하고 있는 key들의 리스트를 보여줌.

```bash
gpg --list-keys
```

* 관리하고 있는 key pair에서 private key를 통해 public key의 소유자임을 identification 함.
* 여기서 signature와 authentification이 관련됨.

## 4. Sign / Signature (서명)

### 정의

* **Sign**: 비밀키로 데이터에 서명하는 행위
* **Signature**: 생성된 서명 데이터

### 개념

서명(signing)은 public key cryptography에서 가장 기본적인 **암호학적 행위**이다.

* 서명은 서명된 대상(data)의 무결성(integrity)과 
* 해당 대상에 서명을 한 서명자 식별(정확히는 서명에 사용된 private key식별) 가능성의 근거를 제공
* 주의할 점은 **signing 자체가 곧 인증(Authentication)은 아니라는 점임**.

```bash
gpg --sign message.txt     
gpg --verify message.txt.gpg
```
* `--sign` 은 서명을 함 (실제로 서명과 암호화가 같이 진행됨)
* `--verify` 는 서명에 대한 verification을 수행.

### Signature vs. Encryption

| 구분        | 서명(Signature) | 암호화(Encryption) |
| --------- | ------------- | --------------- |
| 목적        | 무결성·인증        | 기밀성             |
| 공개키 연산 결과 | 유효성 판단        | 원문 복원           |
| 원본 데이터 복원 | 불가해도 상관없음 | 가능            |
| SSH에서 사용  | 서버 인증         | 사용 안 함          |

	  
## 5. Verification / Verify (검증)

### 개념
Verification은 제시된 **디지털 서명(Signature)이 수학적으로 올바른지** 기술적으로 확인하는 절차임.



### 성공 시 확정되는 사실
* **출처의 수학적 일치**: "이 서명은 해당 Public key와 쌍을 이루는 비밀키(private key or `SK`)로 생성되었다."
	* Mathematrical Correpondence. 
* **무결성(Integrity) 확인**: "서명된 데이터는 전송 과정에서 일절 변조되지 않았다."

### 검증이 결정하지 않는 것 (Non-determinants)
이 단계는 순수한 수학적 연산이며, 신원에 대한 해석(interpretation of identity)을 포함하지 않음.
* **서명자가 누구인가?** (비밀키 탈취 가능성)
* **서명자를 신뢰할 수 있는가?**
* **해당 키가 현재 유효한가?** (폐기 여부 미확인)


## 6. Authentication / Authenticate (인증)

### 정의
**Authentication**은 Verification(검증) 결과를 **Identification(신원 확인)**으로 해석하여, 행위자의 신원을 최종 확정하는 절차입니다.

### 성립 요건 (3요소의 결합)
Authentication은 다음 요소가 결합하며 모두 충족될 때 성립합니다.

1. **Identity Claim (신원 주장)**: 사용자가 자신의 정체성을 제시함. (예: "나는 홍길동(`hong@example.com`)이다.")
2. **Verification Success (검증 성공)**: 기술적 서명 검증 성공. (예: "홍길동의 공개키로 서명이 수학적으로 올바르게 풀린다.")
3. **Binding (연결 고리 확인)**: "이 공개키가 실제 홍길동(`hong@example.com`)의 것인가?"를 신뢰할 수 있는 수단으로 확인하는 과정.

### Binding 방식의 진화

| 방식 | 설명 | 특징 |
| :--- | :--- | :--- |
| **전통적 방식 (Web of Trust)** | 지인의 비밀키를 통한 signing 이나 오프라인 Fingerprint 대조를 통해 확인. | 비밀키 소유자와 실제 인물 간의 관계를 직접 확인. |
| **현대적 방식 (Identity Verification)** | `keys.openpgp.org` 처럼 이메일 인증을 통해 키 점유권 확인. | **제3자 서명(WoT)을 포기**하고 이메일-키 연결의 유효성만 보장. |
| **실용적 방식 (CA/Certificate)** | 신뢰할 수 있는 기관(CA, Certificate Authority)이 identity(신원)과 key(키)를 묶어 Certificate(인증서) 발행. | TLS/SSL 등 현재 대부분의 상용 환경에서 표준으로 사용됨. |

* CA가 발행하는 Certificate는 public credential (공적자격증명) 에 해당함.
* 이 공개키는 홍길동의 것이다라는 binding 을 증명하며, 이는 신뢰받는 CA의 보장에 의해 이루어짐.

> **주의**: 현대적인 키 서버 모델에서 이메일-키의 Binding을 확인해주더라도, 해당 이메일 자체가 사칭(예: `ceo@google.com`을 사칭한 계정)인지 검증하는 것은 여전히 **최종 이용자의 몫** 임.

즉, Auttentication 은

* **서명(Signing)을 도구로 삼아서**, 
* 접속하거나 데이터를 보낸 자의 **신원을 확정**하는 보안 행위를 의미


### 실무 예시: GitHub Pages에서 Remote Theme 사용 시의 인증 과정

Jekyll `remote_theme`를 사용할 때, 내 컴퓨터(Client)가 "테마 배포 서버"의 신원을 확인하는 과정은 다음과 같음.


1. Identity Claim (신원 주장)
	* "테마 배포 서버"가 "나는 특정 테마 패키지를 제공하는 공식 서버다"라고 claim.
	* 해당 서버는 공인된 CA(Certificate Authority) 의 비밀키로 서명된 "서버인증서"를 가짐.
2. CA Credential (신뢰의 뿌리: Trust Anchor)
	* 내 컴퓨터에는 `DigiCert` 등과 같은 공인된 최상위 **Root CA Certificates**가 설치되어 있어야 함.
	* **환경별 독립성**: 
		* OS가 관리하는 저장소 외에도, 
		* Python(`pip install --upgrade certifi`)이나 Conda(`conda install certifi`)는 
		* 각 환경 내에 독립적인 '최신 공인된 CA 명단(CS Bundle)'을 유지.
	* **갱신의 중요성**: 
		* 만약 환경 내 공인된 CA 명단이 구버전이라 
		* 최근 발행된 공인된 CA 정보를 포함하지 않는다면, 
		* 최신 서버 인증서에 대한 **Verification(검증)** 자체가 불가능해짐.
	* **역할**: 
		* 즉, 이 명단은 CA가 발행한 개별 **Certificate(인증서)들의 진위 여부를 판가름하는 절대적인 기준점** 임.
3. Binding 확인 (SSL/TLS Handshake)
	* Jekyll이 HTTPS로 접속하면, GitHub 서버는 자신의 **서버 인증서(Server Certificate)**를 보냄.
	* 내 컴퓨터는 이 인증서에 찍힌 CA의 디지털 서명을 내가 가진 **CA Public Key(CA Certificate)**와 수학적으로 대조.
4. Authentication 완료 (Final Act)
	* 서버 인증서가 내가 신뢰하는 CA의 비밀키로 서명되었음을 확인했다면, 
	* 이는 CA가 해당 서버를 "공식 GitHub 서버"라고 **보증(Binding)**하고 있음을 의미.
	* 이 수학적 증명을 통해 상대방의 신원을 확정하는 **Authentication(인증)**이 완료. 
	* 비로소 안전한 테마 다운로드가 시작됨.



## 7. Primary Key와 Subkey 구조 (GPG, OpenPGP)

보다 자세한 건 다음을 참고: [[/gpg/gpg_primal_sub]]

### 7.1 역할 구분

OpenPGP(GPG)에서는 Private and Public 외에도 Primary and Sub 의 구분이 존재.

* **Primary key**
  * 신원의 기준점 (Act as the identity anchor)
  * 사용자 ID에 대한 서명에 사용됨.
  * 서브키에 대한 **certification** 에 사용됨.
* **Subkey (서브키)**
  * 실제 사용 목적 담당 
  * 서명용, 암호화용 등으로 분리 가능
  * certification, signing, authentication, encryption


### 7.2 Primary 비밀키의 subkey들에 대한 certify 역할

Primary 비밀키는 서브키를 **certify(보증)** 한다.
여기서 certify의 의미는 다음과 같다.

* 암호화가 아님
* 일반 데이터 서명이 아님
* **“이 서브키는 나에 의해 승인되었다”는 키-바인딩 서명**
* 이 subkey는 이 primarykey에 대응되는 것이다.(속한 것이다)

구조는 다음과 같다.

```
User ID
 ↑  (certified by)
Primary Key
 ↑  (certifies)
Subkey
```

검증자(verifier)는 다음을 확인한다.

* 이 서브키가 **Primary key에 의해 certify 되었는가? 
* 수학적으로 올바른가?=위조되지 않않나?

이를 통해 서브키의 정당성(legitimate)이 확보된다.


### 7.3 설계 의의 (Design Rationale)

이 구조로 인해 다음이 가능해진다.

* Primary 키는 긴 기간동안 유지하면서 대응되는 서브키들은 보안을 위해 짧은 주기로 교체
* 필요에 따라 특정 용도의 서브키만 폐기할 수 있음.
* Primary 키를 오프라인에 보관!!

반대로,

* **Primary 비밀키가 유출되면 전체 신뢰가 붕괴됨**
* 따라서 Primary 키는 가장 강하게 보호해야 한다.


## 8. Certificate 

certificate 는 명사로 **인증서** 라고 생각할 것.

### 정의

* **Certificate**: 공개키와 신원을 연결한 신뢰 문서
* Certificates는 **Credential의 한 종류**
* 보통, 신뢰할 수 있는 제3자(CA, 인증 기관)가 발행한 디지털 문서를 의미함.

### 개념

X.509 체계에서는 CA가 인증서를 발급하지만,
OpenPGP에서는 **Primary key가 서브키를 certify 하는 구조**가
인증서 체계의 역할을 부분적으로 대체함.

* 구조: 공개 키(Public Key)와 발행 기관의 디지털 서명이 포함되어 있음.
* 용도: 주로 통신 암호화(HTTPS/SSL)나 신원 확인 시 "이 공개 키가 특정 소유자의 것이 맞음"을 공적으로 보증할 때 사용함.
* 특징: 비밀번호처럼 단순한 값이 아니라, 암호학적 검증이 가능한 구조화된 파일 형태임

### 8-1. Credentials

Credential (자격증명)은 사용자의 신원을 증명하기 위해 제시하는 모든 수단을 아우르는 상위 개념임.

* 범위: 넓은 개념으로, "나는 누구인가"를 입증하는 모든 데이터를 포함함.
* 구성 요소: ID/비밀번호, 생체 정보(지문, 안면 인식), OTP 번호, API Key, 그리고 Certificates까지 모두 Credential의 일종임.
* 특징: 시스템에 접근할 수 있는 '열쇠' 그 자체를 의미함.

## 9. Validation / Validate (유효성 확인)

### 개념

Validation은 verification(검증)과 다르다.

* Verification: signature가 수학적으로 올바른가?
* Validation: 사용해도 되는 상태인가?

만료 여부, 폐기 여부 등은 Validation의 영역이다.


## 10. Authorization / Authorize (인가)

### 정의

* **Authorization**: 인증된 주체에게 허용되는 행위를 결정
* 허용된 권한이 무엇인지를 확인하는 것.

### 개념

Authorization은 Authentication 이후에만 수행됨.
서명이나 키 자체는 권한을 의미하지 않는다.



## 11. Revocation / Revoke (폐기)

### 정의

* **Revocation**: 한때 유효했던 신뢰를 공식적으로 

```bash
gpg --gen-revoke KEYID > revoke.asc
gpg --import revoke.asc
```

폐기는 되돌림이 아니라
“앞으로 더 이상 이 키를 신뢰하지 않는다”는 선언임.

revocation은 한번 이루어지면 영구적임


## 12. 전체 개념 흐름과 각 단계의 역할

### 사전 단계

1. Key pair 생성
2. 공개키 배포 ( [[/gpg/gpg-keyserver]] )
3. Primary key에 의한 Subkey certification (일부는 key pair 생성에서 이루어짐)

### 실행 단계

4. Identification (주체가 내가 누구다라고 주장.)
5. Sign (주체가 비밀키로 데이터에 서명)
6. Verification (해당 서명이 올바른지 확인: sign이 해당 private key로 이루어짐을 확인)
7. Authentication (6번의 결과와 Binding 을 통해, 특정 public key의 소유자인지 신원확인: CA cert 또는 WoT 등이용)
8. Authorization (Authenticated user에게 허용할 권한을 결정)

### 사후 단계

9. Revocation (키 폐기): 이후 key server등에 알려야 함.


## 13. 핵심 정리 문장

* signing 은 암호학적 행위이다: private key를 통해 데이터를 처리.
* Authentication 은 signature verificaton result을 통해 identiy proof(신원 증명)으로 해석하는 절차이다.
* Primary 비밀키는 subkeys를 certify 한다: subkey에 대해 key-binding certificaton signatures를 생성 (=cerify).
* subkeys 는 실사용을 담당한다 (실제 사용되는 key로 보안 등의 이유로 짧은 주기별로 교체/폐기 됨).
* revocation 은 신뢰 관계를 공식적으로 종료한다.


```
* Signing is a cryptographic operation.
* Authentication is the process of interpreting verified signatures as proof of identity.
* The primary private key certifies subkeys.
* Subkeys perform operational cryptographic tasks.
* Revocation formally terminates a trust relationship.
```
