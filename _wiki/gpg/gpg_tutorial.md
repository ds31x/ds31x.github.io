---
layout  : wiki
title   : GnuPG 사용법 익히기.
summary : 
date    : 2026-01-29 18:47:15 +0900
updated : 2026-01-29 19:41:05 +0900
tag     : gpg tutorial
resource: BA/163BC425D54955AF9F69BE7CA102F2
toc     : true
public  : true
parent  : [[/gpg]] 
latex   : false
---
* TOC
{:toc}


# GnuPG(gpg) 사용법 튜토리얼

아래 자료를 미리 읽어보길 권함:
[Cryptography: From Symmetric-Key Encryption to TLS](https://ds31x.tistory.com/613#2.-public-key-cryptography-%EA%B3%B5%EA%B0%9C%ED%82%A4-%EC%95%94%ED%98%B8)

## 1. GnuPG는 무엇을 하는 도구인가

GnuPG는 **공개키 암호(public-key cryptography)** 기반으로 하는 PGP의 구현물임.

주로 다음을 수행:

1. **서명(Sign)**
   * 이 파일이 **누가 만들었는지**
   * 중간에 **변조되지 않았는지** 증명
2. **암호화(Encrypt)**
   * 이 파일을 **특정 수신자만 읽게** 보호

이 둘은 목적이 다르며, **서명만 / 암호화만 / 둘 다** 상황에 따라 사용함.


## 2. 신뢰의 출발점: fingerprint

fingerprint는 **공개키 전체를 요약한 고정 식별자** 임:

* 공개키가 1비트라도 달라지면 fingerprint는 완전히 달라짐
* 동일 fingerprint를 갖는 다른 키를 만드는 것은 현실적으로 불가능
* 사람이 직접 비교할 수 있을 만큼 짧음 (40 hex 가 많이 사용됨)

즉,

> GnuPG에서
> 
> fingerprint를 확인했다는 애기는 
> "대응하는 공개키가 정말 이 사람의 것이다"를 확인했다는 뜻임.
>
> 사실 공개키와 identity의 바인딩은 GunPG만으론 해결이 안됨.


## 3. GPG는 이메일로 fingerprint 확인함

보안 전제는 다음과 같다.

> 공격자가
>
> * 공개키 전달 경로
> * 해당 이메일 계정
> 
> 이 둘을 동시에 장악하지 않는 한**
> fingerprint를 갖는 다른 공개키를 만들어 속이는 것은 현실적으로 불가능

> 공격자가
> 
> * 공개키 전달 경로
> * 해당 이메일 계정
> 
> 이 둘을 동시에 장악하지 않는 한,
> 사용자에게 거짓 공개키와 그에 대응하는 fingerprint를 함께 제시하여 속이는 것(MITM, Man-In-The-Middle)은
> 현실적으로 어렵다.

이 가정하에서 다음 방식은 **유효한 신원 확인**임:

* 공개키는 파일이나 키서버로 받음
* fingerprint는 **해당 이메일 주소의 소유자로부터 이메일 본문으로 별도 확인**

중요한 점은 **경로 분리 (channel sepration)** 임.

* 같은 이메일로 키와 fingerprint를 동시에 받아서는 안 됨
* 키는 키서버에서 받고, fingerprint는 이메일로 받은 뒤 서로 일치하는지 확인

이는 전통적인 Web of Trust에서
직접 대면하여 fingerprint를 확인하던 절차를,
이미 신뢰된 이메일 채널을 이용해 구현한 방식임.


## 4. 튜토리얼에서 사용하는 더미 식별자 

* 나: `User One <me@me.test>`
* 상대: `User Two <other@other.test>`

### 상대 공개키 fingerprint (기준값)

```
AAAA BBBB CCCC DDDD EEEE FFFF 1111 2222 3333 4444
```

* 읽기 쉽게 중간에 공백이 들어가나,
* 식별자로 쓸 경우엔 공백없이 입력해야 함.

### 상대 KEYID (long, fingerprint에서 파생)

```
1111222233334444
```

> KEYID는 fingerprint **마지막 16 hex**를 사용함.

참고로,  
최신 OpenPGP(v5)에 256bit fingerprint(64 hex 문자) 도 정의되어 있으나,  
기본 출력에서는 160bit(v4) 가 일반적임.


## 5. gpg 설치 확인

```bash
gpg --version
```

출력은 다음과 같은 형태임"

```test
gpg (GnuPG) 2.4.9
libgcrypt 1.11.2
```
* gpg가 정상 동작 중
* 아래 모든 설명은 이 버전 기준


## 6. 내 키 생성 (처음 한 번만)

### 왜 필요한가

* **서명**을 하려면 내 **비밀키**
* 다른 사람이 나에게 **암호화**하려면 내 **공개키**가 필요함

### 명령

```bash
gpg --full-generate-key
```

### 선택

* 키 종류: `RSA and RSA` (primary key와 sub key 모두 RSA 알고리즘 사용)
* 키 길이: `4096`
* 만료: 필요에 따라 설정
* 이름: 이 문서에선 `User One` 이라고 가정함.
* 이메일: 역시 `me@me.test` 라고 가정함.
* passphrase 는 기억할 수 있는 것으로 설정.

### 생성 완료

다음의 형태의 출력이 나옴.

```text
gpg: key 9999888877776666 marked as ultimately trusted
gpg: revocation certificate stored as \
~/.gnupg/openpgp-revocs.d/9999888877776666.rev
```

## 7. 키 구조 확인

```bash
gpg --list-secret-keys
```

```text
sec   rsa4096 2026-01-01 [SC]
      9999888877776666
uid           [ultimate] User One <me@me.test>
ssb   rsa4096 2026-01-01 [E]
```

설명:

* `sec [SC]`
  * primary key
  * identity와 certify의 기준
* `ssb [E]`
  * encryption subkey
  * 실제 암호화에 사용
* 실무에서는 **서명·암호화 모두 subkey 중심**


## 8. 내 공개키 내보내기

```bash
gpg --armor --export me@me.test > me_pubkey.asc
```
* `--armor`: 텍스트 형태(ASCII armor)
* 상대에게 전달할 공개키
* 자유롭게 배포 가능

---

## 9. 상대 공개키 가져오기

```bash
gpg --import other_pubkey.asc
```
* 키서버를 통해 가져올 수 있음.

출력은 다음과 같음:
```text
gpg: key 1111222233334444: public key "User Two <other@other.test>" imported
```
* 공개키를 **가져왔을 뿐**
* 아직 신뢰하지 않음

공개키는 `hkps://keys.openpgp.org` 에서 email주소로 검색후 가져올 수 있음
```
gpg --keyserver hkps://keys.openpgp.org --search-keys other@other.test
```
* 이후 여러 키 후보를 보여주고, 번호로 선택 가능함.
* 다운로드와 import가 모두 이루어짐.

다음은 long `<KEYID>`로 다운로드 하는 방법임:

```text
gpg --keyserver hkps://keys.openpgp.org --recv-keys 1111222233334444
```


## 10. fingerprint 확인 (이메일로 받은 fingerprint와 확인)

```bash
gpg --fingerprint other@other.test
```

```text
pub   rsa4096 2025-06-01 [SC]
      AAAA BBBB CCCC DDDD EEEE FFFF 1111 2222 3333 4444
uid   User Two <other@other.test>
```

설명:

* 위 fingerprint는
* 대응하는 이메일을 통해 해당 소유주에게 fingerprint를 받은 후
* 일치하는지를 확인
* 일치하면 해당 공개키를 해당 이메일 소유주의 것으로 신뢰 가능

> 이메일 소유주가 내가 생각하는 실제 그 사람인지는 여전히 모름....
> 이메일을 잘 못 알고 있거나, 도용된 이메일 계정이라면...


## 11. 파일 서명 (무결성, Integrity)

Integrity(무결성) 는 정보가 전송·저장 과정에서 변경되지 않았음을 보장함을 의미.

```bash
gpg --detach-sign hello.txt
```
* `hello.txt.sig` 서명 파일이 생김.

검증:

```bash
gpg --verify hello.txt.sig hello.txt
```
* 서명 파일과 대응하는 파일을 이용하여 verification.


검증 결과는 다음과 같은 형태임:
```text
gpg: Good signature from "User One <me@me.test>" [ultimate]
```

설명:

* 파일이 변조되지 않았음
* 작성자 확인 가능

## 12. 파일 암호화 (기밀성,Confidentiality)

Confidentiality(기밀성) 는 정보가 허가된 수신자만 접근할 수 있도록 보호하는 것을 의미.

```bash
gpg --encrypt -r other@other.test secret.txt
```
* 수신자 `-r`로 지정.
* 해당 이메일의 소유자의 비밀키로만 복호화 가능.

복호화:

```bash
gpg --decrypt secret.txt.gpg
```

결과는 다음과 같음.
```text
gpg: encrypted with rsa4096 key, ID 1111222233334444
```

## 13. 서명 + 암호화

encryption은 단독으로 하기보다는 서명과 같이 하는게 일반적임.

* 누가 보냈는지 `--sign`
* 해당하는 사람만 읽을 수 있게 `--encrypt -r`

```bash
gpg --encrypt --sign -r other@other.test secret.txt
```

복호화:

```bash
gpg --decrypt secret.txt.gpg
```

결과는 다음과 같음:
```text
gpg: encrypted with rsa4096 key, ID 1111222233334444
gpg: Good signature from "User One <me@me.test>" [ultimate]
```

---

---

## 14. 상대 키에 서명 (Web of Trust)

WOT의 핵심 기능.

```text
gpg --edit-key other@other.test
gpg> sign
gpg> save
```

설명:

* "나는 이 공개키가 이 사람의 것임을 확인했다"는 선언
* 해당 공개키의 "키–신원(binding)" 에 대한 보증을 한다는 의미.
* 이는 키서버로 전파된다 (내가 보증한 사실을 다른 이들도 알게 됨)

> 이는 해당 인물을 신뢰(trust)하거나,  
> 그 인물이 서명한 다른 키까지 자동으로 신뢰함을 의미하지는 않음.

## 15. ownertrust 설정

"이 사람이 다른 사람의 키를 서명할 때, 그 판단을 얼마나 믿을 것인가" 인지를 설정.

```text
gpg --edit-key other@other.test
gpg> trust
```

* 3: marginal (부분적으로만 신뢰. 다른 이들의 marginal trust가 모여야 유효해짐)
* 4: full (타인에게 줄 수 있는 최대의 신뢰. 이 사람이 서명한 키는 신뢰 경로에 놓임)
* 5: ultimate (내 키만 가능)

가장 높은 신뢰인 `ultimage`는 내 키에만 가능함 (내가 그렇게 믿기로 한 것임)


이는 로컬 설정임(키서버로 전달되지 않음)

> 공개키에 서명(sign)하는 것은 해당 키의 신원을 보증하는 행위이고,
> ownertrust 설정은 그 키 소유자가 다른 키를 보증할 때
> 그 판단을 얼마나 신뢰할지를 로컬에서 결정하는 로컬 설정(나만의 설정)이다.

## 16. 내가 한 서명 공개

```bash
gpg --keyserver keys.openpgp.org --send-keys 9999888877776666
```

설명:

* **내 KEYID**
* 내가 한 key-signing이 함께 키서버를 통해 전파됨


## 17. 주의사항

* `.gpg`, `.sig`는 바이너리
* `cat`, `vi`로 열지 말 것


이후 잘못 하여 터미널의 모든 글자가  깨지면 다음을 수행하면 이후 터미널 글자 제대로 보임:

```bash
reset
```


