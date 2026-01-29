---
layout  : wiki
title   : GPG Backup and Recovery
summary : 
date    : 2026-01-29 12:27:38 +0900
updated : 2026-01-29 13:00:42 +0900
tag     : gpg
resource: 45/CFE03D7C1F4F909012598C4D652F7C
toc     : true
public  : true
parent  : [[/gpg]]
latex   : false
---
* TOC
{:toc}

# GPG 키 백업과 복구 

본 문서는 GPG 키 백업 및 복구에 대한 문서임.

응용으로는 개인 장비 2대 이상에서 동일한 GPG identity를 운영할 때도 사용가능.

## 1. 기본 원칙

* GPG identity는 **하나의 primary key**로 정의함
* 여러 장비는 **동일한 key material을 공유**함
* 백업은 **형식과 확장자의 관례를 엄격히 준수**함


## 2. 백업 파일 형식과 확장자 관례

### 2.1 형식-확장자 매핑

| 대상                     | 형식            | 확장자    | 비고     |
| ---------------------- | ------------- | ------ | ------ |
| Private key 전체         | binary        | `.gpg` | 표준, 기본 |
| Public key             | ASCII armored | `.asc` | 배포용    |
| Revocation certificate | ASCII armored | `.asc` | 필수     |
| Ownertrust             | plain text    | `.txt` | 선택     |

### 2.2 규칙

* binary 파일에 `.asc` 사용 금지
* armored 출력에는 반드시 `.asc`
* private key는 기본적으로 binary 보관


## 3. Fingerprint 기반 파일명 규칙

### 3.1 Fingerprint 사용 이유

* Key ID(8 or 16 hex)는 충돌 가능성 존재
* Fingerprint(40 hex)는 전 세계적으로 유일 (SHA-1 기반)
* 파일 혼동 및 오배포 방지

### 3.2 파일명 규칙

Fingerprint의 **마지막 16자리**를 파일명에 포함하는 것이 권장되는 관례.

* 하위 16자리 사용이 관례임.

**예시: fingerprint**

```text
1234ABCD5678EFGH9012IJKL3456MNOP7890QRST
```

사용 부분

```text
3456MNOP7890QRST
```

추출 방법

```bash
gpg --with-colons --fingerprint <sec_keyid> \
  | awk -F: '/^fpr:/ {print substr($10, length($10)-15)}'
```

* `<sec_keyid>`는 `gpg --list-secret-keys --keyid-format long`에서 확인 가능
* 예: `sec   rsa4096/ABCD1234EFGH5678 2022-03-01 [SC]`

### 3.3 권장 백업 디렉토리 구조

```text
gpg-backup/
├── privatekey-full-3456MNOP7890QRST.gpg
├── publickey-3456MNOP7890QRST.asc
├── revoke-3456MNOP7890QRST.asc
└── ownertrust-3456MNOP7890QRST.txt
```

## 4. 백업 절차

### 4.1 Private key 전체 백업 (binary)

```bash
gpg --export-secret-keys \
    --export-options export-backup \
    ABCD1234EFGH5678 \
    > privatekey-full-3456MNOP7890QRST.gpg
```

### 4.2 Public key 백업 (ASCII armored)

```bash
gpg --armor \
    --export ABCD1234EFGH5678 \
    > publickey-3456MNOP7890QRST.asc
```

### 4.3 Revocation certificate 생성 (필수)

```bash
gpg --output revoke-3456MNOP7890QRST.asc \
    --gen-revoke ABCD1234EFGH5678
```

### 4.4 Ownertrust 백업 (선택)

```bash
gpg --export-ownertrust \
    > ownertrust-3456MNOP7890QRST.txt
```

## 5. 백업 파일을 암호화를 권장

### 5.1 왜 추가 암호화가 필요한가

* private key 자체는 passphrase로 보호됨
* 그러나 백업 파일 유출 시:
  * 오프라인 brute-force 시도 가능
  * 장기 보관 리스크 존재

따라서 **백업 파일 묶음 자체를 한 번 더 암호화**하는 것이 권장됨.


### 5.2 gpg 대칭 암호화 방식 (가장 흔한 방식)

```bash
tar czf gpg-backup-3456MNOP7890QRST.tar.gz gpg-backup/

gpg -c gpg-backup-3456MNOP7890QRST.tar.gz
```

결과

```text
gpg-backup-3456MNOP7890QRST.tar.gz.gpg
```

* 이 파일만 외부 저장 매체에 보관
* 원본 디렉토리는 즉시 삭제 권장
* 대칭(symmetric) 암호화 `-c` 이므로 passphrase만 있으면 됨.


### 5.3 보관 위치 관례

* 암호화된 USB
* 오프라인 외장 디스크
* 서로 다른 물리적 위치 2곳 이상

**비권장**

* 클라우드 단독 보관
* 이메일 전송
* 평문 파일 보관


## 6. GPG 복구

### 6.1 백업 파일 복호화

```bash
gpg gpg-backup-3456MNOP7890QRST.tar.gz.gpg
tar xzf gpg-backup-3456MNOP7890QRST.tar.gz
```

* symmetric encryption 된 상태이니 passphrase만 있으면 됨.


### 6.2 키 복구

```bash
gpg --import privatekey-full-3456MNOP7890QRST.gpg
gpg --import-ownertrust ownertrust-3456MNOP7890QRST.txt
```

확인

```bash
gpg --list-secret-keys --keyid-format long
```

이전과 출력이 동일해야 함.


## 7. 장비 분실 시 실제 대응 및 복구 순서

### 7.1 분실 직후 판단

| 상황                | 조치            |
| ----------------- | ------------- |
| 단순 분실, 유출 불확실     | 새 subkey 생성   |
| 유출 가능성 있음         | subkey revoke |
| primary key 유출 의심 | 전체 revoke     |


### 7.2 새 장비에서 복구 절차

1. 백업 파일 복호화
2. private key import
3. ownertrust import
4. 키 상태 확인

```bash
gpg --list-secret-keys
```

### 7.3 Subkey 교체가 필요한 경우

```bash
gpg --edit-key ABCD1234EFGH5678
gpg> addkey
gpg> save
```

이후

```bash
gpg --export ABCD1234EFGH5678 > publickey-new.asc
```

외부 서비스(GitHub, 서버 등)에 재배포


### 7.4 최악의 경우 - 전체 폐기

```bash
gpg --import revoke-3456MNOP7890QRST.asc
```

* 기존 identity 완전 종료
* 새 key pair 생성
* 새 fingerprint로 재배포

[[/gpg/gpg_revoke_key]]

## 8. 피해야할 사항들

* binary private key를 `.asc`로 저장
* revoke 파일을 온라인 저장소에 업로드
* primary key 없이 subkey 관리 시도
* 백업 파일을 암호화 없이 장기 보관


## 9. 최종 요약

* private key: binary + `.gpg`
* public, revoke: ASCII armored + `.asc`
* 파일명에 fingerprint 포함
* 백업 묶음은 **반드시 추가 암호화**
* 분실 대응은 subkey 단위부터
* primary 유출 시 identity 종료

