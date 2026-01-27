---
layout: default
title: Search
permalink: /search/
---

<div id="searchStatus" class="search-status"></div>

<div class="search-pager">
  <button id="btnPrev" type="button">Prev</button>
  <span id="pageInfo"></span>
  <button id="btnNext" type="button">Next</button>
</div>

<ul id="results" class="search-results"></ul>

<style>
.search-status{
  padding: 10px 12px;
  margin: 10px 0 12px 0;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #f7f7f7;
  font-size: 0.95em;
}

.search-pager{
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 0 0 14px 0;
}

.search-pager button{
  padding: 6px 10px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #fff;
  cursor: pointer;
}

.search-pager button:disabled{
  opacity: 0.5;
  cursor: default;
}

.search-results{
  margin: 0;
  padding-left: 1.2em; /* 블릿 유지 */
}

.search-results li{
  margin: 10px 0;
}

.search-excerpt{
  margin-top: 6px;
  padding: 8px 10px;
  border: 1px solid #e3e3e3;
  border-radius: 8px;
  background: #fbfbfb;
  font-size: 0.9em;
  line-height: 1.5;
}

.search-excerpt mark{
  padding: 0 2px;
}
</style>

<script src="https://unpkg.com/lunr/lunr.js"></script>
<script>
(async function () {
  const status = document.getElementById("searchStatus");
  const results = document.getElementById("results");
  const pageInfo = document.getElementById("pageInfo");
  const btnPrev = document.getElementById("btnPrev");
  const btnNext = document.getElementById("btnNext");
  const box = document.getElementById("searchBox");

  const pageSize = 30;
  let currentPage = 1;
  let allItems = [];
  let currentQueryRaw = "";
  let docs = [];

  function escapeHtml(s) {
    return (s || "")
      .replaceAll("&", "&amp;")
      .replaceAll("<", "&lt;")
      .replaceAll(">", "&gt;")
      .replaceAll('"', "&quot;")
      .replaceAll("'", "&#039;");
  }

  function escapeRegExp(s) {
    return (s || "").replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
  }

  function buildExcerpt(content, q_norm, radius = 90) {
    const text = (content || "");
    const lower = text.toLowerCase();
    const i = lower.indexOf(q_norm);

    if (i === -1) {
      const head = text.slice(0, 180);
      return escapeHtml(head) + (text.length > 180 ? "..." : "");
    }

    const start = Math.max(0, i - radius);
    const end = Math.min(text.length, i + q_norm.length + radius);

    const prefix = start > 0 ? "..." : "";
    const suffix = end < text.length ? "..." : "";

    const snippet = text.slice(start, end);
    const safe = escapeHtml(snippet);

    const re = new RegExp(escapeRegExp(q_norm), "ig");
    const highlighted = safe.replace(re, m => `<mark>${m}</mark>`);

    return prefix + highlighted + suffix;
  }

  // Text Fragment 점프(지원 브라우저에서 해당 텍스트로 스크롤/하이라이트)
  function buildJumpUrl(pageUrl, q_raw) {
    const q = (q_raw || "").trim();
    if (!q) return pageUrl;
    return pageUrl + "#:~:text=" + encodeURIComponent(q);
  }

  function updatePager(totalHits) {
    const totalPages = Math.max(1, Math.ceil(totalHits / pageSize));
    currentPage = Math.min(Math.max(1, currentPage), totalPages);

    pageInfo.textContent = `Page ${currentPage} / ${totalPages}`;
    btnPrev.disabled = currentPage <= 1;
    btnNext.disabled = currentPage >= totalPages;
  }

  function renderPage() {
    const totalHits = allItems.length;
    const totalPages = Math.max(1, Math.ceil(totalHits / pageSize));
    updatePager(totalHits);

    if (totalHits === 0) {
      status.textContent = `검색 결과 0개 - query: ${currentQueryRaw}`;
      results.innerHTML = "<li>다른 검색어를 사용하세요.</li>";
      return;
    }

    status.textContent = `검색 결과 ${totalHits}개 (30개씩 페이징) - query: ${currentQueryRaw}`;

    const q_norm = (currentQueryRaw || "").trim().toLowerCase();
    const start = (currentPage - 1) * pageSize;
    const end = Math.min(totalHits, start + pageSize);
    const pageItems = allItems.slice(start, end);

    results.innerHTML = pageItems.map(i => {
      const d = docs.find(x => x.url === i.ref);
      const title = escapeHtml(d.title || d.url);
      const excerpt = buildExcerpt(d.content || "", q_norm, 90);
      const jumpUrl = buildJumpUrl(d.url, currentQueryRaw);

      return `
        <li>
          <a href="${jumpUrl}">${title}</a>
          <div class="search-excerpt">${excerpt}</div>
        </li>
      `;
    }).join("");
  }

  try {
    status.textContent = "검색 인덱스 로딩 중";

    const url = "{{ '/search.json' | relative_url }}";
    const res = await fetch(url);
    if (!res.ok) throw new Error("search.json fetch failed: " + res.status);

    docs = await res.json();
    status.textContent = `문서 ${docs.length}개 로딩 완료`;

    const idx = lunr(function () {
      this.ref("url");
      this.field("title");
      this.field("content");

      // 대소문자 무시: 인덱스에는 소문자로 저장
      docs.forEach(d => {
        this.add({
          url: d.url,
          title: (d.title || "").toLowerCase(),
          content: (d.content || "").toLowerCase()
        });
      });
    });

    function doSearch(q) {
      const q_raw = (q || "").trim();
      const q_norm = q_raw.toLowerCase();

      currentQueryRaw = q_raw;

      if (!q_norm) {
        status.textContent = "검색어를 입력하세요.";
        allItems = [];
        currentPage = 1;
        updatePager(0);
        results.innerHTML = "";
        return;
      }

      allItems = idx.search(q_norm);
      currentPage = 1;
      renderPage();
    }

    // 버튼 이벤트
    btnPrev.addEventListener("click", () => {
      if (currentPage > 1) {
        currentPage -= 1;
        renderPage();
        window.scrollTo({ top: 0, behavior: "smooth" });
      }
    });

    btnNext.addEventListener("click", () => {
      const totalPages = Math.max(1, Math.ceil(allItems.length / pageSize));
      if (currentPage < totalPages) {
        currentPage += 1;
        renderPage();
        window.scrollTo({ top: 0, behavior: "smooth" });
      }
    });

    // URL 파라미터로 초기 검색
    const params = new URLSearchParams(window.location.search);
    const query = params.get("searchString");

    if (query) {
      if (box) box.value = query;
      doSearch(query);
    } else {
      status.textContent = "검색어가 없습니다. 상단 검색창에 입력하세요.";
      allItems = [];
      currentPage = 1;
      updatePager(0);
      results.innerHTML = "";
    }

    if (box) box.addEventListener("input", () => doSearch(box.value));

  } catch (e) {
    console.error(e);
    status.textContent = "Search error: " + String(e);
    results.innerHTML = "";
  }
})();
</script>

