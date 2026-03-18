---
layout  : wiki
title   : Vimwiki commands - simple
summary : 
date    : 2026-03-04 17:29:52 +0900
updated : 2026-03-09 22:46:52 +0900
tag     : vimwiki
resource: BF/7568236B264AB7BED806F341F7DAD3
toc     : true
public  : true
parent  : [[/diary]]
latex   : false
---
* TOC
{:toc}

# Vimwiki 명령어 (simple) 

> vimwiki 는 기본적으로 diary를 쓸 수 있는 기능을 제공하는데...  
> gitpage 와 연동에서 이들을 제대로 활용하지 않고 있는터라 현재 거의 쓰고 있지 않음.
> 기본적으로 diary의 index 파일의 위치를 github page와 연동에서 쓰고 있지 않음.

오늘자 diary 생성

```vim
:VimwikiMakeDiaryNote
```

* `diary/2026-03-02.md` 파일이 생성됨.

---

Diary Index 열기

```vim
:VimwikiDiaryIndex
```

---

현재 파일을 참조하는 페이지 목록 

```vim
VimwikiBacklinks
```

---



