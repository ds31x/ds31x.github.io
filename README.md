# ds31x's blog

A personal blog and knowledge base built with Jekyll.

- **Owner**: ds31x
- **Email**: dsaint31@gmail.com
- **GitHub**: [https://github.com/ds31x](https://github.com/ds31x)

---

## 원본과의 차이점 및 주요 특징 (Differences & Key Features)

이 프로젝트는 [johngrib/johngrib-jekyll-skeleton](https://github.com/johngrib/johngrib-jekyll-skeleton)을 기반으로 함.

원본의 대부분의 기능들을 대부분 유지하며, 다음과 같은 극소수의 부분이 수정되었음.

1.  **`rg` (ripgrep)으로의 전환 (Migration to `rg` (ripgrep))**
    - `tool/save-images.sh`와 같은 일부 도구 스크립트에서 기존에 사용되던 `ag`(The Silver Searcher)가 더 빠른 성능의 `rg`(ripgrep)으로 대체되었음: (`rg` 설치가 필요).

2.  **`start.sh` 스크립트 수정 (Modifications to `start.sh` Script)**
    - 로컬 개발 환경의 편의성을 높이기 위해 헬퍼 스크립트의 기능이 일부 수정됨.

---

## 설치 (Installation)

### 1. 사전 요구사항 (Prerequisites)

- **Ruby**: You need to have Ruby installed. You can check the recommended version at [GitHub Pages Dependency versions](https://pages.github.com/versions/).
- **Node.js**: Required to run the metadata generation script.
- **(EN) `rg` (ripgrep)**: Used in some automation tools, replacing `ag`.

### 2. 의존성 설치 (Dependencies)

```bash
# Install Ruby dependencies from Gemfile
bundle install

# Install Node.js dependencies from package.json
npm install
```

### 3. Git Hooks 

새 글을 커밋할 때 자동으로 메타데이터 생성 및 이미지 로컬화를 수행하는 `pre-commit` 훅을 설정.

```bash
cp tool/pre-commit .git/hooks/
```

---

## 로컬 서버 실행 (Running the Server)

### `start.sh` 스크립트 사용 

- **Docker로 실행:**
  ```bash
  ./start.sh docker
  ```

- **Watch 모드로 실행:**
  ```bash
  ./start.sh watch
  ```

- **백그라운드로 실행:**
  ```bash
  # 서버 시작. 로그는 .localhost.log 에 저장.
  ./start.sh back

  # 백그라운드 서버 종료
  ./start.sh kill
  ```

### Using Bundler Directly

```bash
# 1. Generate metadata
./generateData.js

# 2. Serve the site
bundle exec jekyll serve
```

The site will be available at `http://localhost:4000`.

---

## 프로젝트 구조 및 주요 스크립트 (Project Structure & Key Scripts)

### 주요 폴더 (Main Folders)
- **`_config.yml`**: The main Jekyll configuration file.
- **`_data/`**: Data files used by Jekyll at build time.
- **`_includes/`, `_layouts/`, `_sass/`**: Folders defining the Jekyll theme and structure.
- **`_posts/`, `_wiki/`**: Core content directories for blog posts and wiki documents.
- **`data/`**: `generateData.js`에 의해 생성된 JSON 메타데이터가 저장되는 곳. 사이트 내 검색, 태그 기능에 사용. 
- **`tool/`**: `pre-commit` hook, 이미지 저장 스크립트 등 자동화 및 보조 도구들이 위치. 
- **`resource/`**: `save-images.sh` 스크립트에 의해 외부 이미지가 다운로드되는 폴더. 

### Git Hooks 및 주요 스크립트 동작 방식 (Git Hooks & Key Scripts Workflow)

이 프로젝트는 `pre-commit` Git hook을 통해 커밋 시점의 작업을 자동화.

**`pre-commit` 실행 순서:**
1.  **`./generateData.js` 실행:**
    - 모든 마크다운 파일(`_wiki`, `_posts`)의 Front Matter를 파싱하여 사이트 검색 및 태그 기능에 필요한 JSON 메타데이터를 `data` 폴더에 생성. `package.json`에 명시된 `yamljs` 라이브러리를 사용.
2.  **`./tool/save-images.sh` 실행:**
    - 스테이징된 마크다운 파일에 포함된 외부 이미지 URL을 `rg`로 찾아 로컬 `resource` 폴더로 다운로드하고, 파일 내 경로를 수정.
3.  **변경사항 자동 스테이징 (`git add`):**
    - 위 과정에서 생성된 메타데이터(`data`, `_data`)와 다운로드된 이미지, 수정된 마크다운 파일을 현재 커밋에 자동으로 추가.

---

## 콘텐츠 작성 (Content Creation)

### 새 카테고리 만들기 (Creating a New Category)

`/_wiki/` 디렉터리에 새 파일을 생성하고 (예: `/_wiki/my-category.md`), 아래와 같이 front matter를 작성합니다. `layout`은 `category`여야 함.

```markdown
---
layout  : category
title   : 제목을 입력합니다.
summary : 
date    : 2022-10-06 00:00:00 +0900
updated : 2022-10-06 00:00:00 +0900
tag     : 
toc     : true
public  : true
parent  : index
latex   : false
---

* TOC
{:toc}
```

### 새 글 작성하기 (Creating a New Post)

`/_wiki/` 또는 카테고리 하위 디렉터리에 마크다운 파일을 생성하고 (예: `/_wiki/my-category/my-post.md`), 아래와 같이 front matter를 작성합니다. `layout`은 `wiki`여야 하며, `parent`는 상위 카테고리 이름이어야 함.

```markdown
---
layout  : wiki
title   : 제목을 적습니다
date    : 2022-10-08 11:23:00 +0900
updated : 2022-10-08 11:23:00 +0900
tag     : 
toc     : true
public  : true
parent  : category-name
latex   : false
---

* TOC
{:toc}

내용을 적습니다.
```

