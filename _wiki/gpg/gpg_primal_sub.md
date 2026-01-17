---
layout  : wiki
title   : GPG 에서의 primal keys와 sub keys 
summary : 
date    : 2026-01-17 19:17:15 +0900
updated : 2026-01-17 21:27:36 +0900
tag     : 
toc     : true
public  : true
parent  : [[/gpg]]
latex   : false
resource: 4d6a0b7b-ffe5-4feb-aa69-04bcd8875580
---
* TOC
{:toc}

# OpenPGP(GnuPG)에서 공개키/비밀키와 primary key/subkey의 구조와 운용


## 1. 일반적인 public key (공개키)와 private key (비밀키)

public-key cryptography 는 기본적으로 **두 종류의 키**를 가짐.

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
# 비밀키로 서명. hello.txt.gpg 파일 생성.
gpg --sign hello.txt

# 공개키로 서명 검증. 검증 결과가 출력됨.
gpg --verify hello.txt.gpg
```

## 2. GPG(OpenPGP) 의 경우엔 한 단계 더 나눔.

GPG(OpenPGP)는 위의 공개키 / 비밀키 구조를 그대로 유지하면서,
각각을 다시 다음과 같이 **구조적으로 분해**한다.

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

![](https://github.com/user-attachments/assets/0b8f074b-e4b2-4a65-84a8-76043e7d1d9e){style="display: block; width:500px"}

* `pub` → primary key
* `sub` → subkey
* `[SC]`, `[E]`, `[S]` : 역할 플래그
* `[expieres: 2029-01-16] : 유효기간.


## 4. primary key의 의미와 역할

### 4.1 identity의 기준

**primary key**는 다음 질문에 답하는 key임.

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

primary key는 **signing(서명)** 도 수행 가능함.

```bash
gpg --local-user 9F3A7C2D8B4E1A90 --sign message.txt
```

* 이는 **identity-level signing** 임.
* 가능은 하지만,
* 실제로 사용되는 경우는 드물다.


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

이 경우 GnuPG는 자동으로
`[S]` 역할을 가진 signing subkey를 선택하여 사용.

### 5.3 Encryption subkey 사용 예

```bash
gpg --encrypt -r user@example.com secret.txt
```

이때 사용되는 key는
`[E]` 역할을 가진 encryption subkey 임.


## 6. public key 는 보통 “통째로” 배포되고 사용됨

**public key는 대부분 다음을 모두 포함하고 있음.

* public primary key
* 모든 public subkey
* 인증 정보 (UID, email)


### 예제: 공개키 배포

```bash
gpg --export --armor user@example.com > pubkey.asc
```

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

이는 **정상적인 운용 절차**이다.

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

### 9.1 remove (제거)

subkey를 목록에서 제거하는 작업이다.
공개적으로 “무효”를 선언하지는 않고, 
이미 공개된 경우에 문제를 일으키니 사용하지 않는게 나음.

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

* compromised
* superseded
* no longer used


## 10. 변경 후 공개키 재배포

subkey 추가,교체,폐기 후에는
해당 공개키를 저장하여 배포하거나
다시 key server에 배포해야 한다.

```bash
gpg --export --armor <KEY-ID> pubkey.asc
```

아래는 key server 업로드:

```bash
gpg --export --armor <KEY-ID> | curl -T - https://keys.openpgp.org
```
