---
layout  : wiki
title   : GPG 에서의 primal keys와 sub keys 
summary : 
date    : 2026-01-17 21:37:51 +0900
updated : 2026-01-29 16:20:36 +0900
tag     : gpg
resource: 34/5126287A1C4E9A8B0874BFDE1CA262
toc     : true
public  : true
parent  : [[/gpg]]
latex   : false
---
* TOC
{:toc}

# OpenPGP(GnuPG)에서 공개키/비밀키와 primary key/subkey의 구조와 운용


## 1. 일반적인 public key (공개키)와 private key (비밀키)

public-key cryptography 는 기본적으로 **두 종류의 키** 를 가짐.

* **public key (공개키)**
	* 외부에 배포되는 키로,
	* 서명 검증(verification)과 암호화(encryption)에 사용.

* **private key (비밀키)**
	* 소유자만 보관하는 키로,
	* 서명 생성(signing)과 복호화(decryption)에 사용.

> private key로 서명한 것은 public key로 검증할 수 있고,  
> public key로 암호화한 것은 private key로 풀 수 있음.


### 예제: signing 과 certification 

```bash
# 비밀키로 서명: hello.txt.gpg 파일 생성.
gpg --sign hello.txt

# 공개키로 서명 검증: 검증 결과가 출력됨.
gpg --verify hello.txt.gpg

# 검증과 파일의 내용을 확인.
gpg --decrypt hello.txt.gpg
```

* 뒤에 나오지만, 여기서 사용된 키들은 서명용 subkey임.
* `--sign`의 결과물은 원래의 파일의 내용과 함께 서명정보가 들어감.
* 서명과 내용을 분리한 형태로 처리할 수도 있음: `gpg --detach-sign hello.txt`
* 이 경우엔 검증하려면 `hello.txt`도 같이 필요: `gpg --verify hello.txt.sig hello.txt`
	* 서명이 attach된 경우와 달리, `hello.txt`의 내용이 바뀌면 검증시 `BAD signature ...`라고 뜸.

## 2. GPG(OpenPGP) 의 경우엔 한 단계 더 나눔.

GPG(OpenPGP)는 위의 공개키 / 비밀키 구조를 그대로 유지하면서,
각각을 다시 다음과 같이 **구조적으로 구분함**.

* **primary key**
* **subkey**

이는 누구의 key 인지 와 어떤 역할을 하는지를 구분함.

> 이 key 묶음은 누구의 것인가?
> 이 key 묶음은 어떤 역할을 수행하는가?


## 3. OpenPGP에서 "하나의 key 묶음"의 실제 구조

OpenPGP에서 "하나의 key 묶음"는 다음과 같은 **실제 구조**를 가짐.

```
primary key
  ├─ subkey (signing)
  ├─ subkey (encryption)
  └─ subkey (authentication)
```

이 구조는

* 공개키 쪽에도 존재하고
* 비밀키 쪽에도 존재함.

결국 4개의 키 종류가 있음:
* private primary key
* private subkey
* public primary key
* public subkey

### 예제: key 묶음 구성 확인

```bash
gpg --list-keys --keyid-format=long
```

출력 예:

<img src="/resource/34/5126287A1C4E9A8B0874BFDE1CA262/0b8f074b-e4b2-4a65-84a8-76043e7d1d9e.png" style="display: block; width:500px" />

* `pub` → primary key
* `sub` → subkey
* `[SC]`, `[E]`, `[S]` : 역할 플래그
* `[expieres: 2029-01-16]` : 유효기간.


## 4. primary key의 의미와 역할

### 4.1 identity의 기준

**primary key**는 다음 질문에 답에 해당하는 key임.

> 이 키 묶음은 누구의 것인가?

* key server 등에서 key 묶음을 식별을 primary key 의 fingerprint로 식별.
	* 참고로 key server에선 public key만 보관됨.
* 비 식별자료인 UID(name, email)가 primary key에 연결됨


결국 fingerprint가 해당 OpenPGP identity의 기준임.


### 4.2 Certification (C, 인증)

primary key의 핵심 역할은 **certification(인증)** 임.

* UID가 누구를 나타내는지 인증
* subkey가 누구에게 속하는지 인증
* subkey가 어떤 역할을 수행하는지 보증

### 예제: subkey 추가 시 해당 subkey에 자동 인증

```bash
gpg --edit-key 9F3A7C2D8B4E1A90
gpg> addkey
gpg> save
```

생성된 subkey는
primary key에 의해 자동으로 certification된다.

* subkey들과 UID가 누구의 것인지 certification을 해주는 역할.


### 4.3 Signing (S, 서명)

primary key는 이론적으로 signing(S) capability를 가질 수 있으나,
실제로는 identity(UID) 및 subkey에 대한 **certify(C) 서명에만 사용된다.
데이터 서명(signing)은 signing subkey가 담당하는 것이 OpenPGP 및 GnuPG의 표준 운영 모델이다.
* 즉, primary key는 signing 권한을 가질 수 있으나 ,
* GnuPG에서는 실제 데이터 서명 시 signing subkey가 자동으로 사용됨.
    * 사용자가 primary key를 명시하더라도, 이는 identity 선택에 가깝고 실제 서명 연산은 subkey에서 수행.
 
```bash
gpg --local-user <KEYID> --sign message.txt
```

* 이는 **identity-level signing** 임: 연결된 email주소도 가능.
* 여전히 서명용 subkey 사용됨.


## 5. subkey의 의미와 실제 사용

### 5.1 action(행위)을 수행하는 key가 subkey임.

**subkey**에는 어떤 행위에 사용되는지가 할당됨.

* subkey를 통해 
* 연관된 identity 가 무엇을 하는지를 알 수 있음.

subkey는 항상 **구체적인 다음의 역할**을 가짐.

* `S` : signing 
* `E` : encryption
* `A` : authentication


### 5.2 Signing subkey 사용 예

```bash
gpg --sign commit.txt
```
* `commit.txt` 의 내용에 서명이 첨부된 `commit.txt.gpg`가 생성됨.
* `cat commit.txt.gpg`로 수행시 파일 내용은 보임 (`--encrypt`와 `--sign`의 차이).

이 경우 GnuPG는 자동으로
`[S]` 역할을 가진 signing subkey를 선택하여 사용.

### 5.3 Encryption subkey 사용 예

```bash
gpg --encrypt -r user@example.com secret.txt
```

이때 사용되는 key는
`[E]` 역할을 가진 encryption subkey 임.

> 일반적으로 OpenPGP 메시지는 서명(`--sign`)과 암호화(`--encrypt`)를 함께 수행한다.
> 메시지는 수신자의 공개키(encryption subkey)로 암호화되며,  
> 동시에 송신자의 비밀키(signing subkey)를 사용하여 서명된다

```bash
gpg --encrypt --sign -r dsaint31@naver.com hello.txt
```

명시적인 버전은 다음과 같음:
```bash
gpg --encrypt --sign -u <KEYID> -r dsaint31@naver.com secret.txt
```

## 6. public key 는 보통 “통째로” 배포되고 사용됨

**public key는 대부분 다음을 모두 포함하고 있음.

* public primary key
* 모든 public subkey
* 인증 정보 (UID, email)


### 예제: 공개키 배포

```bash
gpg --export --armor user@example.com > pubkey.asc
```
* 생성된 `pubkey.asc` 자체를 배포.

subkey 만에 대한 public key 를 따로 추출하여 사용하는 경우는 별로 없음.


## 7. 기존 키에 subkey를 추가 가능

OpenPGP 설계의 핵심은 다음 문장이다.

> primary key는 identity로 유지되고,
> subkey는 이후에 추가·교체·제거·폐기될 수 있음.

기존 키에 subkey를 추가할 수 있음.

### 예제: subkey 추가

```bash
gpg --edit-key <KEY_ID>
gpg> addkey  # 원하는 subkey를 고르면 됨. 그만두려면 <C-c>
gpg> save
```

## 8. subkey 교체(replace)

subkey 교체는 다음 두 단계의 조합이다.

1. 새 subkey 추가
2. 기존 subkey를 더 이상 사용하지 않도록 처리

이는 **정상적인 운용 절차** 임.

### 예제: 교체 흐름

```bash
gpg --edit-key <KEY-ID>
gpg> addkey  # 암호화용(E) 또는 서명용(S) subkey 생성 등
gpg> key <n> # <n> 은 지울 키의 번호(선택되면 앞에 *가 붙음)
gpg> expire  # 만료일을 과거 또는 매우 짧게 설정.
gpg> save
```
* `<n>` : 은 primary key에 0이 붙고 순서대로 1,2로 증가.
* `expire` 대신 `delkey`를 사용하여 지울 수 있으나 공개되어 있는 경우 문제가 발생하므로 비추천.


## 9. subkey remove(제거)와 revocation(폐기)

`delkey` : "없었던 것처럼 치우기" (remove)
* 로컬 관리 목적
* 테스트용 subkey 정리 등에 사용됨.
* 아직 배포하지 않은 subkey 제거

`revkey` : "사용하면 안 된다고 알리기" (revocation)
* 공개적 무효화 선언
* 이미 배포된 subkey 폐기
* 타인이 명시적으로 invalid로 인식

### 9.1 remove (제거)

subkey를 목록에서 제거하는 작업이다.

* 이는 로컬 keyring에서 subkey를 제거하는 작업.
* 공개적으로 해당 subkey의 무효를 선언하지 않음,
* 다른 사용자의 keyring이나 keyserver에는 전혀 영향을 주지 않음 (여전히 외부에선 유효함).
* 이미 배포된 subkey를 더 이상 사용하지 않도록 알릴 필요가 있는 경우에는 `revoke`를 사용하는 것이 적절함

```text
gpg --edit-key <KEY-ID>
gpg> key 1
gpg> delkey
gpg> save
```

### 9.2 revocation (폐기)

subkey 폐기는 다음 의미를 가짐.

> 이 subkey는 더 이상 신뢰해서는 안 된다.

이는 한번 폐기된 key는 이를 **되돌릴 수 없음.**

```text
gpg --edit-key <KEY-ID>
gpg> key 1
gpg> revkey
gpg> save
```

사유(reason) 를 선택해야 함:

* compromised (손상됨·유출됨)
* superseded (대체됨)
* no longer used


## 10. 변경 후 공개키 재배포

subkey 추가,교체,폐기 후에는  
해당 공개키를 저장하여 배포하거나  
다시 key server에 배포해야 한다.

```bash
gpg --export --armor <KEY-ID> pubkey.asc
```

**아래는 key server 업로드하는 방법임:&&

[[/gpg/gpg_ks_setting]]{dirmngr} 을 우회하는 방법 (`keys.openpgp.org`만 가능)
```bash
gpg --export --armor <KEY-ID> | curl -T - https://keys.openpgp.org
```

정식 방법.
```bash
gpg --keyserver hkps://keys.openpgp.org --send-keys <KEY-ID>
```
