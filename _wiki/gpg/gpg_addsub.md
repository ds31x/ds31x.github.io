---
layout  : wiki
title   : How to add new signing subkey
summary : 
date    : 2026-01-29 14:10:43 +0900
updated : 2026-01-29 14:22:12 +0900
tag     : 
resource: 94/E921DB95714274976F0DDB71D4603F
toc     : true
public  : true
parent  : [[/gpg]]
latex   : false
---
* TOC
{:toc}

# 새 signing subkey 추가 방법

(primary key 유지, 기존 subkey와 병행 또는 교체)

* 기존 `<signing_subkey>` 가

  * 유출되었거나
  * 장기간 사용되어 교체가 필요한 경우
* 만료 연장이 아니라 **새 서명 키로 전환**하려는 경우 임.

expiration 연장이 가능한 상황이라면 다음을 참고: [[/gpg/gpg_expire]]


## 1단계. primary key 편집 모드 진입

```bash
gpg --edit-key <primary_key>
```


## 2단계. 새 subkey 추가

GPG 프롬프트에서:

```text
gpg> addkey
```

선택 값:

* Key type: `(8) RSA (sign only)`
* Key size: `4096`
* Expiration: `1y` 또는 `2y`
* Email: 기존 `<email>` 그대로


## 3단계. 저장

```text
gpg> save
```

결과:

* 기존 subkey는 유지됨
* **새 signing subkey가 primary key 아래 추가됨**
* key ID / fingerprint는 **새로 생성됨**


## 4단계. 새 subkey ID 확인

```bash
gpg --list-secret-keys --keyid-format LONG <primary_key>
```

출력 예:

```text
sec   rsa4096/<primary_key>
ssb   rsa4096/<old_signing_subkey> [SC]
ssb   rsa4096/<new_signing_subkey> [S]
```


## 5단계. public key 재게시 (필수)

```bash
gpg --armor --export <primary_key>
```


## 6단계. GitHub에 반영

1. GitHub → **Settings → SSH and GPG keys**
2. **GPG keys**
3. 기존 키 유지
4. **갱신된 public key 추가**

* GitHub는 public key 안의 **모든 signing subkey를 자동 인식**
* 어떤 subkey를 쓸지는 **로컬 Git 설정**이 결정


## 7단계. keyserver(keys.openpgp.org)에 반영

```bash
gpg --send-keys <primary_key>
```

또는:

```bash
gpg --armor --export <primary_key> | \
  curl -T - https://keys.openpgp.org
```

* email 재검증 없음
* primary key 기준 검증 유지


## 8단계. 로컬 Git에서 새 subkey 사용하도록 전환

```bash
git config --global user.signingkey <new_signing_subkey>
git config --global commit.gpgsign true
```

### `<new_signing_subkey>` 값 얻는 방법

secret key 목록에서 subkey ID 확인

```text
gpg --list-secret-keys --keyid-format LONG
```

출력은 다음과 같은 형태임:

```text
sec   rsa4096/<primary_key>
uid           <email>
ssb   rsa4096/ABCDEF1234567890 [S]
ssb   rsa4096/1234567890ABCDEF [E]
```

* `[S]` 또는 `[SC]` 표시된 것이 signing subkey
* 슬래시(`/`) 뒤의 값이 subkey ID: 이 값이 `<new_signing_subkey>`
	* 위의 경우: `ABCDEF1234567890`

확인:

```bash
git config --global --get user.signingkey
```


## 9단계. 동작 확인

```bash
git commit -S -m "test signed commit with new subkey"
```

* 오류 없음
* GitHub에서 **Verified** 표시 확인

## 정리: 연장 vs 추가

| 구분          | expire 연장 | 새 subkey 추가 |
| ----------- | --------- | ----------- |
| key ID      | 유지        | 변경          |
| fingerprint | 유지        | 변경          |
| Git 설정 변경   | 불필요       | 필요          |
| 권장 상황       | 단순 만료     | 교체, 보안 재정비  |


## Summary

* **단순 만료 → expire 연장**
* **보안 교체 → 새 signing subkey 추가**
* 두 경우 모두
  **primary key 유지 + public key 재게시**는 필수
 
