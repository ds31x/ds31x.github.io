---
layout  : wiki
title   : keyserver 지정하기 
summary : 
date    : 2026-01-29 15:31:37 +0900
updated : 2026-01-29 15:39:35 +0900
tag     : gpg keyserver
resource: 0B/53CEE8B3B14A6592CEB0EAE89C487C
toc     : true
public  : true
parent  : [[/gpg/]]
latex   : false
---
* TOC
{:toc}

# GnuPG에서 `keys.openpgp.org` 키서버 사용 설정

## 목적

* OpenPGP 공개키를 `keys.openpgp.org` 키서버에 게시 및 조회
* 키서버 접속 경로를 명시적으로 통제
* 최신 GnuPG(2.x) 구조(dirmngr, keybox 기반)에서 예측 가능한 동작 확보

> Debian과 Ubuntu에선 `hkps://keyserver.ubuntu.com`이 기본으로 되어 있음.  
> 이는 과거 sks계열과 호환되나 append-only 방식이고 일반 개발자에겐 그리...  
> `hkps://keys.openpgp.org` 로 바꾸는 게 나음.


## GnuPG 구성 요소와 역할

최신 GnuPG는 기능별로 구성 요소가 분리되어 있다.

* **gpg**
  * 키 생성, 서명, 검증 수행
  * 키 메타데이터 처리 정책 결정
* **dirmngr**
  * 키서버 통신 전담
  * hkps(HTTPS) 연결 및 TLS 검증 처리
* **keybox**
  * 공개키 및 메타데이터 저장 구조
  * 기존 keyring 방식은 사용되지 않음

키서버와의 실제 네트워크 접속은 **`dirmngr`가 수행**하며,
키 내부 메타데이터를 어떻게 해석할지는 **gpg의 정책**에 의해 결정된다.


## 키서버 기본 설정 (접속 대상 지정)

실제로 접속할 keyserver URL은 **dirmngr 설정 파일**에 지정한다.

> 초기엔 없으므로, 만들어줘야 함.

### 설정 위치

```
~/.gnupg/dirmngr.conf
```

### 설정 내용

```conf
keyserver hkps://keys.openpgp.org
```

의미:

* dirmngr가 사용할 **기본 keyserver를 명시적으로 고정**
* 키 조회(`--recv-keys`), 전송(`--send-keys`) 시 이 서버가 사용됨

권한 설정:

```bash
chmod 600 ~/.gnupg/dirmngr.conf
```

설정 반영:

```bash
gpgconf --kill dirmngr
gpgconf --launch dirmngr
```

설정 검증:

```bash
dirmngr --gpgconf-test
```

동작하고 있는 확인:

```bash
gpg-connect-agent 'GETINFO version' /bye

# D 2.4.3
# OK
```


## 참고: 키 내부 메타데이터로 인한 접속 경로 변경 문제

OpenPGP 공개키에는 다음과 같은 **선택적 메타데이터**가 포함될 수도 있음.

```
Preferred-Keyserver: hkp://example.com
```

이 필드는 키를 생성하거나 배포한 주체가
"이 키는 특정 keyserver를 사용하라"는 **권고함** 을 의미.

GnuPG는 기본 동작에서, 

* 키 내부에 이같은 정보가 존재하면
* 사용자가 설정한 keyserver가 있더라도 
* **이를 참조하여 다른 서버로 접속**할 수 있음.

이로 인해 다음과 같은 문제가 발생할 수 있음:

* 의도하지 않은 keyserver로의 접속
* 네트워크 경로의 비예측성 증가
* 문서/조직 기준과 다른 서버 사용


### 해결책: 키 메타데이터에 의한 접속 변경을 차단하는 설정

위와 같은 상황을 방지해야 하는 경우,
키 내부의 `Preferred-Keyserver` 정보를 **무시하도록 강제하는 정책 옵션**을 사용할 수 있음.

#### 설정 위치

이 옵션은 **gpg의 정책 옵션**이므로 다음 파일에 설정:

```
~/.gnupg/gpg.conf
```

#### 설정 내용

```conf
keyserver-options no-honor-keyserver-url
```

의미:

* OpenPGP 키 내부의 `Preferred-Keyserver` 메타데이터를 전혀 참조하지 않음
* 모든 키 조회·전송 요청은
    * `dirmngr.conf`에 지정된 keyserver
    * 또는 명령행에서 명시한 `--keyserver` 옵션만 사용

주의:

* 일부 환경에서는 이 옵션을 dirmngr 설정 파일에 넣을 경우 오류가 발생할 수 있으므로
  **반드시 `gpg.conf`에만 설정**한다.
* `keys.openpgp.org` 단일 운영 환경에서는 필수 옵션은 아니며,
  접속 경로 통제가 필요한 경우에만 사용한다.


## 동작 확인

```bash
gpg --send-keys <SHORT_KEYID 또는 LONG_KEYID>
```

출력 예:

```
gpg: sending key XXXXXXXX to hkps://keys.openpgp.org
```

표시된 URL이 `keys.openpgp.org`이면 설정이 정상 적용된 것임.


## 정리

* 실제 keyserver URL 지정
    * `~/.gnupg/dirmngr.conf`
* 키 내부 메타데이터 처리 정책
    * `~/.gnupg/gpg.conf`
* 접속 대상과 정책을 **역할별로 분리**하여 설정


