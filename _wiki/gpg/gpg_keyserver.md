---
layout  : wiki
title   : Key Server - keys.openpgp.org 
summary : 
date    : 2026-01-17 15:37:50 +0900
updated : 2026-01-18 08:43:17 +0900
tag     : gpg
resource: 69/c864d7f69b47efade39f594306bddb
toc     : true
public  : true
parent  : [[/gpg]]
latex   : false
---
* TOC
{:toc}

---

# OpenPGP key-server와 keys.openpgp.org


## 1. key-server(key server)의 목적

**key-server(key server)** 는 OpenPGP 환경에서

* public keys(public primary key, public subkey)를 
* distribution (배포)하고 
* search (검색)하기 위한 서버.

key-server의 존재 목적은 다음과 같다.

1. public key를 중앙에서 공유할 수 있게 한다.
2. 특정 사용자의 public key를 쉽게 찾을 수 있게 한다.
3. public key의 변경 사항을 다른 사용자에게 전달한다.

중요한 점은 다음과 같다.

> key-server는 **public key만을 다루며**,
> private key(개인키)는 어떠한 경우에도 저장하거나 처리하지 않음.


## 2. OpenPGP key-server의 기본 특성

### 2.1 public key만 취급한다

key-server에 업로드(upload)되는 정보는 다음으로 제한됨.

* primary key
* subkey
* UID(User ID)
* 키 역할 플래그(key usage flags)
* 만료 정보(expire date)
* 폐기 정보(revocation)

private key 들은 key-server 가 다루는 범위에 포함되지 않음.


### 2.2 key-server는 "사람"이 아니라 "key"를 관리한다

key-server는 사용자 계정(account)을 기준으로 동작하지 않음.

대신 다음을 기준으로 key를 식별한다.

* **primary key fingerprint**

즉,

* email 주소는 식별 기준이 아니다.
* 사용자 이름은 식별 기준이 아니다.
* 오직 **키 자체**가 기준이 된다.


## 3. keys.openpgp.org의 설계 철학

**keys.openpgp.org**는 기존 OpenPGP key-server의 문제점을 해결하기 위해 설계됨.

### 3.1 기존 key-server의 문제점

전통적인 key-server(OpenPGP HKP 서버)는 다음과 같은 문제를 가졌다.

* 허위 UID(email) 등록이 매우 쉬움.
* 스팸 UID 대량 삽입 가능.
* key 정보 오염(key poisoning)이 발생하기 쉬움.
* key 삭제가 사실상 불가능.


### 3.2 keys.openpgp.org의 접근 방식

이를 개선하기 위해 keys.openpgp.org는 다음과 같은 정책을 채택:

1. **email 검증(email verification)** 을 통해 UID를 제한.
2. **Web of Trust 서명**을 서버에서 제거함.
3. 검증된 UID만 검색 가능하게 처리.

이로 인해 keys.openpgp.org는
* "신뢰 계산"이 아니라 
* "key 배포"에 집중하는 서버로 동작.


## 4. key-server에서의 key upload

### 4.1 Upload의 의미

key-server에 public key를 upload한다는 것은 다음을 의미:

> "현재 public key의 상태를 서버에 **등록(register)** 하거나 **갱신(update)** 한다"

이는 key 를 생성하는 행위가 아님.

만든 key 를 upload.


### 4.2 Upload되는 정보의 범위

Upload 시 서버에 전달되는 정보는 다음과 같음:

* primary key와 subkey 구조
* 각 키의 역할 플래그(`[SC]`, `[S]`, `[E]`, `[A]`)
* UID 목록
* 만료 정보
* 폐기 정보

이 정보는 이후 다른 사용자가 키를 받을 때 그대로 전달된다.


## 5. key-server에서의 email UID 검증

### 5.1 UID와 email의 의미

public key에는 **UID(User ID)** 가 포함되며, 일반적으로 다음 형태를 가짐.

```
이름 <email@example.com>
```

이 email 주소는 다음을 **주장(claim)** 하는 역할임.

> "이 key는 이 email 주소의 소유자의 key이다."

---

### 5.2 keys.openpgp.org의 email 검증

keys.openpgp.org는 이 주장을 그대로 신뢰하지 않음.

따라서 다음 절차를 수행하여 검증함:

1. public key에 포함된 email 주소로 인증 메일을 전송.
2. 해당 메일의 링크를 클릭해야 keyserver에 인증된 UID로 활성화시킴.

결과적으로:

* 인증된 `UID` => 검색 가능
* 인증되지 않은 `UID` => 서버에는 해당 public key는 존재하나 검색 불가


## 6. 키의 동일성 판단 기준

key-server에서 **같은 key인지 여부**는 다음 기준으로 판단된다.

* **primary key fingerprint**

이 기준에 따라 다음과 같이 처리된다.

* fingerprint가 같으면 => 같은 키의 업데이트(update) => 이메일 인증 처리 없음.
* fingerprint가 다르면 => 완전히 다른 키 => 추가를 위한 이메일 인증 처리 요구.

email 주소가 같더라도 fingerprint가 다르면 서로 다른 키이다.


## 7. key-server와 키 역할 플래그의 관계

key-server는 키의 **역할(role)** 을 해석하거나 변경하지 않음.

다만, public key에 포함된 **키 역할 플래그(key usage flags)** 를 그대로 저장/배포한다.

### 주요 역할 플래그

* **`S` (Sign, 서명)**
  서명 생성에 사용
* **`C` (Certify, 인증)**
  다른 키와 UID를 인증 (키와 UID가 본인 것임을 인증)
* **`E` (Encrypt, 암호화)**
  데이터 암호화
* **`A` (Authenticate, 인증)**
  로그인 및 인증 (자기가 자신을 증명)

특히,

* primary key는 보통 `[SC]` 역할을 가진다.
* subkey는 `[S]`, `[E]`, `[A]` 중 하나를 가진다.

key-server는 이 정보를 변경하지 않으며,
클라이언트가 이를 해석하여 사용한다.


## 8. key-server에서의 update

같은 키(primary key fingerprint 동일)를 다시 업로드하면,  
key-server는 이를 **업데이트(update)** 로 처리한다.

update를 통해 갱신되는 정보는 다음과 같음:

* subkey 추가 또는 변경
* 만료일 변경
* 폐기 정보 추가

key-server는 “과거 상태”로 되돌리는 기능(revert)을 제공하지 않는다.


## 9. revert(되돌리기)가 불가능한 이유

OpenPGP key-server는 **누적 모델(append-only model)** 을 따른다.

* 한 번 공개된 정보는 삭제되지 않는다.
* revocation (폐기정보)는 취소가 불가함.
* revert(되돌리기)는 개념적으로 아예 존재하지 않음.

key-server에서 가능한 조작은 다음으로 제한됨:

> 새로운 정보를 추가하여
> key 의 현재 상태를 변경하는 것


## 10. Revocation(폐기 선언)과 key-server

**revocation(폐기 선언)** 은 key-server에서 매우 중요한 정보이다.

* 폐기된 key는 서버에서 “무효”로 표시된다.
* key는 여전히 다운로드 가능하지만, 제대로 된 클라이언트에서 이의 사용을 거부하는 방식임.
* 이는 삭제가 아니라 **영구적인 신뢰 중단 선언** 에 해당함.


## 11. key-server에서의 만료(expire)

**expire date(만료일)** 정보는 key-server 는 key의 정보를 제출한 당시 그대로 배포됨.

* 만료된 key는 서버에서 삭제되지 않음.
* 클라이언트가 이를 해석하여 사용 여부를 결정하는 방식임.
* 만료일 변경은 업데이트를 통해 반영 가능함 (key-server 에 업데이트를 안하면 갱신 안 됨).


## 12. key-server 관점에서의 최종 정리

* key-server는 public key만을 다룬다.
* key-server는 사용자 계정을 관리하지 않음.
* key의 정체성은 primary key fingerprint로 결정됨.
* email UID는 검색 가능 여부를 제어할 뿐이다.
* key 역할 플래그는 서버가 해석하지 않고 그대로 배포.
* 같은 key 재업로드는 업데이트로 처리된다.
* revert(되돌리기)는 불가능하다.
* 폐기는 revoke(폐기 선언)로 표현된다.
* key-server는 신뢰를 판단하지 않고, 업데이트된 정보를 전달할 뿐임.


