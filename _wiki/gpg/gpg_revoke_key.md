---
layout  : wiki
title   : How to revoke GPG Key
summary : 
date    : 2026-01-29 12:50:17 +0900
updated : 2026-01-29 15:58:39 +0900
tag     : 
resource: 81/1ADD009DDD47C7B71193930DD120D2
toc     : true
public  : true
parent  : [[/gpg]]
latex   : false
---
* TOC
{:toc}

# gpg 키 전체 폐기

Primary key 유출이 의심되거나, 신뢰 체인을 더 이상 유지할 수 없는 경우 **기존 GPG identity를 완전히 폐기**한다.

## 1 Revocation certificate import

```bash
gpg --import revoke-3456MNOP7890QRST.asc
```
* 해당 fingerprint에 대응하는 key를 **영구적으로 폐기(revoked)** 상태로 표시
* 로컬 키링 기준에서 identity 종료 선언


## 2 폐기된 공개키 재배포 (키서버 반영)

* Revocation은 **내 로컬에만 import하면 아무 의미가 없음**.
* 반드시 외부에 공개되어야 함.

### 1) 키서버로 revoked key 업로드

```bash
gpg --keyserver hkps://keys.openpgp.org --send-keys ABCD1234EFGH5678
```

* 기존 public key + revoke 정보가 함께 전송됨
* 키서버에 등록된 해당 key는 **revoked 상태로 표시**

**주의**

* 이 단계가 누락되면 외부에서는 여전히 **"유효한 키"** 로 인식함


### 2) 주요 키서버 반영 관례

GnuPG 기본 설정 기준으로 다음이 대상이 됨.

* `keys.openpgp.org`

**확인**

```bash
gpg --keyserver hkps://keys.openpgp.org --send-keys ABCD1234EFGH5678
```


## 3 키서버 특이 사항 (중요)

* **keys.openpgp.org**
  * UID(email) 검증 중심
  * revoke는 정상 반영됨
  * revoked 상태가 명확히 표시됨

* 과거 SKS 계열
  * 네트워크 불안정
  * 현재는 보조적 의미

관례적으로

> **revocation은 반드시 `keys.openpgp.org`에 반영**


## 4 외부 서비스 재배포 (필수)

키서버만으로는 충분하지 않음.
다음 위치에 **폐기 사실 또는 새 키**를 명시적으로 반영해야 함.

* GitHub / GitLab
  * GPG key 삭제
  * revoked key 제거
  * 새 key fingerprint 등록
* 개인 웹사이트
* 이메일 서명
* 문서, 논문, 명함 등에 기재된 fingerprint

원칙

> “키서버 + 직접 배포”가 동시에 이뤄져야 폐기가 완결됨


## 5 새 key pair 생성

```bash
gpg --full-generate-key
```

* 기존 key와 **완전히 무관한 새 identity**
* fingerprint 절대 재사용 불가


## 6 새 fingerprint 재배포


```bash
gpg --armor --export NEWKEYID > publickey-new.asc
```
* 새 public key export


```bash
gpg --keyserver hkps://keys.openpgp.org --send-keys NEWKEYID
```
* 키서버 업로드

* 외부 서비스 등록 (GitHub, 서버, 이메일 등)


## 7 문서화 관례 (권장)

기존 fingerprint가 공개된 문서가 있다면 다음과 같이 처리함.

> “이 fingerprint는 202X-XX-XX부로 폐기되었으며, 새 fingerprint는 XXXX이다.”

이는 **장기 신뢰 관리**에서 매우 중요함.


## 핵심 체크리스트

* [ ] revoke.asc import
* [ ] 키서버로 revoked key 전송
* [ ] GitHub 등 외부 서비스 정리
* [ ] 새 key 생성
* [ ] 새 fingerprint 재배포


