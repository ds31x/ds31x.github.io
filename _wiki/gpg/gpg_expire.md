---
layout  : wiki
title   : How to Extend an Expired GPG Signing Subkey 
summary : 
date    : 2026-01-29 13:42:29 +0900
updated : 2026-01-29 13:55:35 +0900
tag     : gpg
resource: 45/CF9945F98B407A998429B920896DDD
toc     : true
public  : true
parent  : [[/gpg]]
latex   : false
---
* TOC
{:toc}

# GPG signing subkey 만료(expired) 연장 방법

primary key 유지하면서 subkey 를 연장하여 사용

## 표기 규칙

* `<primary_key>` : primary key ID 또는 fingerprint
* `<signing_subkey>` : signing subkey ID
* `<email>` : key에 등록된 email

## 가정

* `<primary_key>` 는 유효
* GitHub, keyserver에 이미 검증되어 게시됨
* `<signing_subkey>` 만 expired 상태
* 목표는 **기존 subkey의 expiration 연장**

## 1단계. primary key 편집 모드 진입

```bash
gpg --edit-key <primary_key>
```
<img src="/resource/45/CF9945F98B407A998429B920896DDD/346c4dab-8aaa-4867-9b5b-6818c8e7d536.png" style="display: block; margin:0 auto; width: 400px" />


## 2단계. 연장할 signing subkey 선택

GPG 프롬프트에서:

```text
gpg> key <N>
```

* `<N>`은 signing subkey 번호 (맨 위가 `0`): 예) `1`, `2`
* `[S]` 또는 `[SC]` 표시된 subkey 선택
* 선택되면 `*` 표시 확인

예:

```text
ssb* rsa4096/<signing_subkey> [SC] [expired]
```

<img src="/resource/45/CF9945F98B407A998429B920896DDD/fd647331-7d13-4e1e-b62a-be8a5ccea56b.png" style="display: block; margin:0 auto; width: 400px" />


## 3단계. subkey 만료일 변경

```text
gpg> expire
```


<img src="/resource/45/CF9945F98B407A998429B920896DDD/926272c9-6839-4846-ad91-86d9c9dca8f0.png" style="display: block; margin:0 auto; width: 400px" />


입력 예:

* `1y`
* `2y`
* `YYYY-MM-DD`

주의 사항:

* **같은 subkey**, expiration만 변경됨
* key ID / fingerprint 유지

---

## 4단계. 변경 사항 저장

```text
gpg> save
```

---

## 5단계. 로컬 상태 확인

```bash
gpg --list-secret-keys --keyid-format LONG <primary_key>
```

* `[expired]` 표시 제거 확인
* 새 만료일 표시 확인

## 6단계. public key 재게시 (필수)

```bash
gpg --armor --export <primary_key>
```


## 7단계. GitHub에 반영 (수정본 - expire 연장 케이스)

로컬에서 갱신된 공개키 블록(export 결과) 생성

```bash
gpg --armor --export <primary_key> > <public_key.asc>
```

GigHub에 GPG 등록

1. GitHub → **Settings → SSH and GPG keys** → **GPG keys**
2. 기존에 등록된 동일 키(fingerprint 기준)를 **삭제(Delete)**
3. **New GPG key**에서 `<public_key.asc>` 내용을 붙여넣어 **재등록(Add)**

주의 사항

* GitHub는 등록된 GPG key를 **편집/갱신(update)** 하는 기능이 없음
* expiration 연장은 공개키 블록(패킷)이 변경되므로, **삭제 후 재등록**이 정석
* "기존 키 유지 + 갱신된 public key 추가"는 동일 fingerprint 중복으로 **추가가 불가능** 함.


## 8단계. keyserver(keys.openpgp.org)에 반영

```bash
gpg --send-keys <primary_key>
```

또는:

```bash
gpg --armor --export <primary_key> | \
  curl -T - https://keys.openpgp.org
```

* email 재검증 불필요
* 기존 검증 상태 유지

## 9단계. 로컬 Git 설정 확인

이미 `<signing_subkey>` 를 사용 중이면 변경 불필요.

```bash
git config --global --get user.signingkey
```


## 10단계. 동작 확인

```bash
git commit -S -m "test signed commit after expiration extension"
```

* 오류 없음
* GitHub에서 **Verified** 표시 확인

## 주의 사항

* expire 연장은 **private key 보유자만 가능**
* public key 재게시 없으면
  * GitHub
  * keyserver
    모두 변경 사항을 알 수 없음
* primary key는 건드릴 이유 없음

