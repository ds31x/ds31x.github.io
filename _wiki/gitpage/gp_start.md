---
layout  : wiki
title   : Github.io 페이지 만들기
summary : 
date    : 2026-01-28 00:46:25 +0900
updated : 2026-01-28 01:27:15 +0900
tag     : bundle gem jekyll
resource: 2B/373DAF98724D3EA52998B0DBEFD534
toc     : true
public  : true
parent  : [[/gitpage]]
latex   : false
---
* TOC
{:toc}


# Minimal Mistakes로 GitHub.io 페이지 만들며 gem과 Bundler 이해하기


## 0. 이 튜토리얼의 목적

0. **package manager** 의 개념 이해.
1. markdown 과 정적 웹사이트 호스팅 에 대한 이해.
2. **gem과 Bundler의 역할과 차이를 이해**
3. 그 결과물로 **로컬과 GitHub.io에서 동시에 동작하는 사이트** 확보


## 1. 먼저 개념부터 정리 (반드시 알고 가야 함)

### 1.1 Markdown이란?

**Markdown**은 글을 쓰기 위한 간단한 문법이다.

```md
# 제목
## 소제목

- 목록
- 목록

**강조**
```

* 사람이 읽기 쉬움
* Jekyll이 Markdown을 HTML로 변환함

**우리는 글을 Markdown으로 쓰고, 이를 Jekyll 이 웹에서 서비스 가능한 HTML 로 만든다.**

**참고** : [Markdown 이란](https://dsaint31.me/mkdocs_site/CE/markdown_latex/markdown)

### 1.2 GitHub Pages (github.io)란?

**GitHub Pages**는 GitHub가 제공하는 **정적 웹사이트 호스팅 기능**이다.

* 저장소(repository)에 있는 파일을
* Jekyll로 빌드해서
* 웹사이트로 배포해 준다

형태는 두 가지가 있다.

* 사용자 사이트: `USERNAME.github.io`
* 프로젝트 사이트: `USERNAME.github.io/REPO_NAME`

### 1.3 Jekyll이란?

**Jekyll**은 **정적 사이트 생성기** 임.

* Markdown 파일
* 레이아웃 / 테마
* 설정 파일

을 읽어서 **HTML 파일 묶음(_site)**을 만들어 준다.

참고로, 
* GitHub Pages는 
* 내부적으로 Jekyll을 사용한다.

### 1.4 gem이란?

**gem**은 Ruby의 패키지 단위 를 의미하며 해당 패키지를 설치하는 프로그램을 가리키기도 함.

Ruby에서는 

* 라이브러리
* 도구
* 프레임워크

모두 gem 형태로 배포됨.

```bash
gem install jekyll
```

이 명령은:

* RubyGems 저장소에서
* jekyll 패키지를 다운로드하고
* 이를 현재 장비의 Ruby 환경에 설치

> 주의할 점은 
> **gem은 일종의 package manager라고 보다는 단순 설치 툴임.
> "설치" 를 위해 의존성 메타 데이터를 읽을 수 있으나,
> 프로젝트 단위의 의존성 충돌 해결 기능이나, 버전 충돌 검사, 환경격리 기능등은 제공하지 않음.

**참고:** [package manager란?](https://ds31x.tistory.com/335)


### 1.5 Bundler란?

**Bundler**는 gem을 "프로젝트 단위"로 관리하게 해주는 툴임.

* 설치를 직접하지는 않지만 (gem을 이용함), 
* 이를 제외한 package manager의 기능을 갖춤.

Bundler의 핵심 역할:

* 이 프로젝트에서
* 어떤 gem을
* 어떤 버전 조합으로 쓸지
* **고정하고 재현 가능하게 만드는 것**

이를 위해 Bundler는 두 파일을 사용한다.

| 파일             | 역할                  |
| -------------- | ------------------- |
| `Gemfile`      | 사람이 적는 "요구사항"       |
| `Gemfile.lock` | Bundler가 만든 "확정 답안" |


### 1.6 Minimal Mistakes란?

**Minimal Mistakes**는 Jekyll 테마다.

* 문서/강의 노트에 적합
* 검색, 카테고리, 태그, 아카이브 기본 제공
* GitHub Pages와 호환되도록 설계됨
* `remote_theme` 방식으로 사용 가능.


## 2. 오늘 실습의 핵심 구조

이 튜토리얼 전체의 동작을 이해하기 위한 구조임.

```
Gemfile        ← 사람이 작성
   ↓
bundle install ← Bundler
   ↓
Gemfile.lock   ← Bundler가 생성
   ↓
bundle exec    ← lock 기준으로 실행
```

이걸 이해하면 **모든 명령이 왜 필요한지** 설명할 수 있음.


## 3. Minimal Mistakes starter로 프로젝트 시작

### 3.1 GitHub에서 저장소 생성

* 템플릿: [`mmistakes/mm-github-pages-starter`](https://github.com/mmistakes/mm-github-pages-starter)
* “Use this template”로 새 저장소 생성
* 사용자 사이트면 저장소 이름을 `USERNAME.github.io`

<img src="https://github.com/user-attachments/assets/f23488ba-f621-4153-b1c1-2e0f073268fe" style="display: block; margin: 0 auto; width:300px" />

### 3.2 로컬로 clone

```bash
git clone <YOUR_REPO_URL>
cd <YOUR_REPO_DIR>
```

## 4. Ruby 실행 환경 준비 (예: conda)

ruby 기반의 jekyll 프로젝트를 수행하기 위해서 일종의 격리된 개발 환경을 만드는 게 유리함.

* S/W는 여러 기존의 라이브러리 등에 의존성을 가짐.
* Jekyll은 ruby, native extension, gem 버전에 매우 영향을 많이 받음.
* 시스템의 다른 S/W에도 영향을 주므로 격리된 환경에서 동작하게 하는게 좋음.

이를 위해 `conda`라는 툴을 사용.

```bash
conda create -n jekyllmm -c conda-forge -y \
  ruby=3.3 zlib libffi readline ncurses yaml \
  compilers make pkg-config \
  ca-certificates certifi
conda activate jekyllmm
```

* 아직 프로젝트 gem은 설치되지 않음
* Ruby 실행 환경만 준비한 상태
* 해당 실행 환경의 conda 환경을 사용하여 
* 시스템의 Ruby 와 분리됨.


## 5. gem 단계: Bundler 설치

```bash
gem install bundler -N
bundle -v
```

### 이 명령이 참고하는 것

* 프로젝트 파일을 **참고하지 않음**
* RubyGems 저장소만 사용

### 여기서 배우는 것

* Bundler도 gem이다
* gem은 “환경 단위 설치 도구”

### 주의할 점

이 단계에서 설치되는 Bundler는
현재 활성화된 conda 환경에 종속된 Ruby에 설치되며,
시스템 Ruby나 다른 conda 환경과는 분리된다.

conda 환경 자체가 이미 격리를 제공하므로,
`gem --user-install`과 같은 gem 로컬 설치 옵션은 사용하지 않는다.

> `--user-install` 의 경우, 사용자의 홈 디렉토리 기준으로 gem이 설치됨:  
> 즉, gem이 `~/.gem` 아래로 설치되므로 conda의 격리 효과가 없어지니 사용하면 안됨.


---

## 6. Bundler: 프로젝트 고립 설정

다음의 명령어는 반드시 프로젝트의 root 디렉토리에서 이루어져야 한다.

```bash
rm -rf .bundle vendor/bundle Gemfile.lock

bundle config set --local path "vendor/bundle"
bundle config set --local disable_shared_gems "true"
bundle config set --local clean "true"
```
* `path "vendor/bundle"` : gem이 설치될 위치 지정 (설정 안하면 시스템 gem경로 혹은 conda 환경의 gen에 설치)
* `disable_shared_gems "true"` : 시스템의 전역 gem을 사용하지 않도록 설정.
* `clean "true"` : `bundle install` 실행시 `Gemfile.lock`에 없는 gem을 자동으로 제거함 (실제 사용중인 gem만 남음).

### 이 명령이 만드는 것

* bundle이 사용하는 설정 파일 생성: `.bundle/config`
* 이후 gem은 `vendor/bundle`에만 설치됨

### 의미

* 전역 gem과 섞이지 않게 차단: `--local`
* 프로젝트 단위 환경 확보
* `--local` 은 해당 프로젝트에만 적용되는 설정이 됨.


## 7. Bundler: `bundle install`의 정확한 동작

```bash
bundle install --verbose
```

### `bundle install`이 참고하는 것

1. **Gemfile** 

   * 이 프로젝트가 원하는 gem 목록
2. **Gemfile.lock** (이미 있으면)

   * 이전에 확정된 버전 조합

### `bundle install`이 하는 일

1. Gemfile 읽기
2. 의존성 전체 계산(resolve)
3. 충돌 없는 버전 조합 결정
4. `Gemfile.lock` 생성 또는 갱신
5. gem 실제 설치

**이 단계가 Bundler의 존재 이유** 라고 할 수 있음: `Gemfile.lock`을 만들어냄.

* gem은 설치만 할 뿐, `Gemfile`을 아예 해석하지 않음.

## 8. 로컬 실행: `bundle exec`의 의미

```bash
bundle exec jekyll serve
```

### 참고: SSL connecton의 CRL 에러 발생시

> 2026.1.28 현재 Jekyll3.10.0에서  
> 위의 명령어 실행시 Ruby의 OpenSSL (Secure Socket Layer) 사용하면서 `unable to get certificate CRL`에러가 뜸.  
> **원인:**
> remote_theme 기능을 사용할 때 Jekyll은 외부(GitHub) 서버에 접속하여 테마 파일을 다운로드함.
> * 이때 시스템의 OpenSSL 설정이 엄격하거나,
> * 특정 네트워크 환경(방화벽, 프록시)에서 CRL 서버에 접근하지 못할 경우
> * certificate verify failed 오류를 뿜으며 중단됨.
> 현재 `curl`로는 되는데 conda환경의 Ruby 내에선 안 됨.
> 우선 장비에서 Jekyll을 돌릴 때는 remote_theme를 쓰지 않도록 우회해서 해결.  

에러는 다음과 같음:
```text
❯ bundle exec jekyll serve
To use retry middleware with Faraday v2.0+, install `faraday-retry` gem
Configuration file: /home/dsaint31/git/CE/_config.yml
            Source: /home/dsaint31/git/CE
       Destination: /home/dsaint31/git/CE/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
      Remote Theme: Using theme mmistakes/minimal-mistakes
jekyll 3.10.0 | Error:  SSL_connect returned=1 errno=0 peeraddr=20.200.245.246:443 state=error: certificate verify failed (unable to get certificate CRL)
```

해결은 `Gemfile` 에서 `minimal-mistakes-jekyll` gem을 추가하고
```text
source "https://rubygems.org"

gem "github-pages", group: :jekyll_plugins
gem "minimal-mistakes-jekyll" # added

gem "tzinfo-data"
gem "wdm", "~> 0.1.0" if Gem.win_platform?

# If you have any plugins, put them here!
group :jekyll_plugins do
  gem "jekyll-paginate"
  gem "jekyll-sitemap"
  gem "jekyll-gist"
  gem "jekyll-feed"
  gem "jemoji"
  gem "jekyll-include-cache"
  gem "jekyll-algolia"
end
```
이후 `bundle install`을 수행하여 `minimal-mistakes-jekyll` gem을 설치한다.

그리고 `_config.yml`을 복사하여 `_config_local.yml`을 만들고 remote_theme를 theme로 수정한다.
```yml
# Build settings
markdown: kramdown
theme: minimal-mistakes-jekyll
```
그리고, 다음과 같이 bundle로 장비에서 실행시 `_config_local.yml`을 지정하여 수행.
```
bundle exec jekyll serve --config _config_local.yml
```

이 경우 장비와 github.io 페이지 모두에서 제대로 동작하게 됨.

아니면, 다음처럼 시스템의 Cert를 사용하는 시도를 해볼 수 있음 (테스트 환경에선 실패)
```bash
# Ubuntu/Debian 기준
export SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt

# 그 후 다시 실행
bundle exec jekyll serve
```

> 버그가 수정되면 이 절은 무시해도 됨.

### 이 명령이 참고하는 것

* **Gemfile.lock**

### 의미

* 이 프로젝트에서 결정된 gem 조합만 로드하여 실행(`exec`).
* 전역 jekyll이 끼어들 수 없음

### 확인

```bash
bundle exec which jekyll
bundle exec jekyll -v
```
* 위의 명령어로 어디에 설치된 jekyll이 수행되었는지 확인 가능함.

브라우저에서 다음을 통해 jekyll의 정적 웹호스팅 결과를 확인 가능:

* `http://127.0.0.1:4000`

---

## 9. Markdown 실습: 글 하나 추가

다음은 `bash` 환경 기준임.

> 일반 에디터로  
> `_posts/2026-01-28-first-note.md` 파일을 생성해서  
> `cat ..` 아래 부터 `EOF`전까지의 내용을 입력하고 저장해도 됨  

참고: [bash 및 shell에 대한 글](https://ds31x.tistory.com/page/Shell-%EC%9A%94%EC%95%BD-%EC%A0%95%EB%A6%AC-bash-%EA%B8%B0%EC%A4%80)

```bash
mkdir -p _posts
cat > _posts/2026-01-28-first-note.md <<'EOF'
---
title: "첫 강의 노트"
categories: [lecture]
tags: [jekyll, bundler, gem]
---

## 오늘 배운 것

- gem과 Bundler의 역할 차이
- bundle install이 무엇을 참고하는지
EOF
```

* 저장 후 웹브라우저에서 새로고침하면 바로 반영됨


## 10. GitHub Pages 배포 (브랜치 기반)

[git이란](https://ds31x.tistory.com/271)

### 10.1 GitHub 설정

1. 저장소 → Settings
2. Pages
3. Source: **Deploy from a branch**
4. Branch: `main`
5. Folder: `/ (root)`
6. Save

참고: [remote theme로 github.io 페이지 만들기](https://dsaint31.github.io/github/git-page/)

### GitHub가 참고하는 것

* 선택한 브랜치의 Source
* GitHub 내부에서 Jekyll 빌드 수행


## 11. push

다음의 명령어로 로컬에 있는 코드를 github의 저장소로 업로드(`push`)

```bash
git add .
git commit -m "init minimal mistakes site"
git push
```

잠시 후:

* Settings → Pages에 URL 표시
* 해당 주소로 접속하면 로컬과 동일한 사이트 확인

현재 Github는 단순 password로는 push등의 저장소를 수정하지 못함:
* 일반적으로는 공개키 기반의 인증을 통해 push등을 수행하는 게 편함 (ssh이용)
* https를 이용하려면, PAT(Person Authentication Token) 방식을 사용하면 됨.

PAT로 push를 하려면 다음 URL참고: [PAT를 통한 authentication for GitHub](https://ds31x.tistory.com/603)


## 12. 최종 요약.

* **gem**
  * 패키지 설치 도구
  * 프로젝트 개념 없음
* **Bundler**
  * 프로젝트 단위 의존성 관리자
* **Gemfile**
  * 사람이 적는 요구사항
  * Bundler 가 읽는 지시사항임.
* **Gemfile.lock**
  * Bundler가 만든 확정된 답안
* **bundle install**
  * Gemfile을 읽고 lock을 만들며 설치
* **bundle exec**
  * lock 기준으로 실행 강제
* **GitHub Pages (브랜치 배포)**
  * 저장소 소스를 읽어 Jekyll 빌드 후 배포


