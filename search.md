---
layout: default
title: Search
permalink: /search/
---

<div id="searchStatus" class="search-status"></div>
<ul id="results" class="search-results"></ul>

<style>
/* 검색 요약/상태 박스 */
.search-status{
  padding: 10px 12px;
  margin: 10px 0 14px 0;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #f7f7f7;
  font-size: 0.95em;
}

/* 결과 리스트는 블릿 유지 (기본 ul 스타일 유지) */
.search-results{
  margin: 0;
  padding-left: 1.2em;
}
.search-results li{
  margin: 6px 0;
}
</style>

<script src="https://unpkg.com/lunr/lunr.js"></script>
<script>
(async function () {
  try {
    const status = document.getElementById("searchStatus");
    const results = document.getElementById("results");
    const box = document.getElementById("searchBox");

    // 초기 상태
    status.textContent = "search.js loaded";

    const url = "{{ '/search.json' | relative_url }}";
    const res = await fetch(url);
    status.textContent = `fetch ${url} -> ${res.status}`;

    const docs = await res.json();
    status.textContent = `docs loaded: ${docs.length}`;

    const idx = lunr(function () {
      this.ref("url");
      this.field("title");
      this.field("content");

      docs.forEach(d => {
        this.add({
          url: d.url,
          title: (d.title || "").toLowerCase(),
          content: (d.content || "").toLowerCase()
        });
      });
    });

    status.textContent = "lunr index built";

    function render(items, q_raw) {
      if (!items.length) {
        status.textContent = `검색 결과: 0개 (query: ${q_raw})`;
        results.innerHTML = "<li>다른 검색어를 사용하세요.</li>";
        return;
      }

      status.textContent = `검색 결과: ${items.length}개 (상위 30개 표시) - query: ${q_raw}`;

      results.innerHTML = items.slice(0, 30).map(i => {
        const d = docs.find(x => x.url === i.ref);
        return `<li><a href="${d.url}">${d.title}</a></li>`;
      }).join("");
    }

    function doSearch(q) {
      const q_raw = (q || "").trim();
      const q_norm = q_raw.toLowerCase();

      if (!q_norm) {
        status.textContent = "검색어를 입력하세요.";
        results.innerHTML = "";
        return;
      }

      render(idx.search(q_norm), q_raw);
    }

    const params = new URLSearchParams(window.location.search);
    const query = params.get("searchString");

    if (query) {
      if (box) box.value = query;
      doSearch(query);
    } else {
      status.textContent = "검색어가 없습니다. 상단 검색창에 입력하세요.";
      results.innerHTML = "";
    }

    if (box) box.addEventListener("input", () => doSearch(box.value));

  } catch (e) {
    console.error(e);
    const status = document.getElementById("searchStatus");
    if (status) status.textContent = "Search error: " + String(e);
    alert("Search error: " + e);
  }
})();
</script>


