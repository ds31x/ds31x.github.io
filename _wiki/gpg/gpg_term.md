---
layout  : wiki
title   : GPG 관련 용어
summary : 
date    : 2026-01-18 15:04:08 +0900
updated : 2026-01-18 16:07:39 +0900
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

Public-key cryptography (공개키 암호)는 항상 **키 쌍(pair)** 으로 구성된다.

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

이 공개키는 이후 signature verification 과 Authentication 에서 사용됨.


## 3. Identification / Identify (식별)

### 정의

* **Identification**: 주체가 자신이 누구인지 주장(claiming an identity)하는 단계
* **Identify**: 그 주장을 제시하는 행위 (to present that claim)

### 개념

이 단계에서는 아직 신뢰(trust)가 생성되지 않는다.

단지 “나는 이 공개키의 소유자다”라는 주장이 제시된다.

다음은 gpg가 관리하고 있는 key들의 리스트를 보여줌.

```bash
gpg --list-keys
```

## 4. Sign / Signature (서명)

### 정의

* **Sign**: 비밀키로 데이터에 서명하는 행위
* **Signature**: 생성된 서명 데이터

### 개념

서명(signing)은 public key cryptography에서 가장 기본적인 **암호학적 행위**이다.

* 서명은 서명된 대상(data)의 무결성(integrity)과 
* 해당 대상에 서명을 한 서명자 식별(정확히는 서명에 사용된 private key식별) 가능성의 근거를 제공
* 그 자체가 곧 인증(Authentication)은 아님.

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

검증은 서명이 (수학적으로) 올바른지 확인하는 절차이다.

성공 시 다음 사실만이 확정된다.

* “이 서명은 해당 공개키에 대응하는 비밀키로 생성되었다”

단, 이 단계에서는 **신원에 대한 해석이 포함되지 않는다**.

Verification does not determine:

* who the signer is
* whether the signer should be trusted
* whether the key is still acceptable for use

---

## 6. Authentication / Authenticate (인증)

### 정의

* **Authentication**: 검증 결과를 신원 확인으로 해석하는 절차

### 개념

Authentication은 다음 요소가 결합될 때 성립한다.

* Identification에서 제시된 신원(identity claim)
* 서명(Sign)과 검증(Verification) 결과
* “이 공개키는 이 사용자에게 속한다” 를 확인.

즉, Auttentication 은

* signing(서명)을 도구로 삼아서
* 접속한 자의 신원을 확인한 행위를 의미.


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
  * certification, signing, authentification, encryption


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

certificate 는 명사로 인증서 라고 생각할 것.

### 정의

* **Certificate**: 공개키와 신원을 연결한 신뢰 문서

### 개념

X.509 체계에서는 CA가 인증서를 발급하지만,
OpenPGP에서는 **Primary key가 서브키를 certify 하는 구조**가
인증서 체계의 역할을 부분적으로 대체함.


## 9. Validation / Validate (유효성 확인)

### 개념

Validation은 검증과 다르다.

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
7. Authentication (6번의 결과에 따라, 특정 public key의 소유자인지 신원확인)
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
