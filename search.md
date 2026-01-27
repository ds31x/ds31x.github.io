---
layout: default
title: Search
permalink: /search/
---

<div id="searchStatus" class="search-status"></div>

<div class="search-pager">
  <button id="btnPrev" type="button">Prev</button>
  <div id="pageNumbers" class="page-numbers"></div>
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
  flex-wrap: wrap;
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

.page-numbers{
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
}

.page-num{
  padding: 6px 10px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #fff;
  cursor: pointer;
  user-select: none;
}

.page-num.active{
  font-weight: 700;
  background: #efefef;
}

.page-ellipsis{
  padding: 0 6px;
  color: #666;
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

<script>
  // ===== Search settings from Jekyll (_config.yml) =====
  const SEARCH_PAGE_SIZE = {{ site.search.page_size | default: 30 }};
  const SEARCH_DEBOUNCE_MS = {{ site.search.debounce_ms | default: 200 }};
</script>

<script src="https://unpkg.com/lunr/lunr.js"></script>
<script>
(async function () {
  const status = document.getElementById("searchStatus");
  const results = document.getElementById("results");
  const btnPrev = document.getElementById("btnPrev");
  const btnNext = document.getElementById("btnNext");
  const pageNumbers = document.getElementById("pageNumbers");
  const box = document.getElementById("searchBox");

  const pageSize = SEARCH_PAGE_SIZE;
  const debounceMs = SEARCH_DEBOUNCE_MS;

  let currentPage = 1;
  let allItems = [];
  let currentQueryRaw = "";
  let docs = [];
  let idx = null;

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

  function buildJumpUrl(pageUrl, q_raw) {
    const q = (q_raw || "").trim();
    if (!q) return pageUrl;
    return pageUrl + "#:~:text=" + encodeURIComponent(q);
  }

  function getTotalPages() {
    return Math.max(1, Math.ceil(allItems.length / pageSize));
  }

  function clampPage(p) {
    const tp = getTotalPages();
    return Math.min(Math.max(1, p), tp);
  }

  function setUrlParams(queryRaw, page, push=true) {
    const u = new URL(window.location.href);

    if (queryRaw && queryRaw.trim()) u.searchParams.set("searchString", queryRaw.trim());
    else u.searchParams.delete("searchString");

    if (page && page > 1) u.searchParams.set("page", String(page));
    else u.searchParams.delete("page");

    if (push) history.pushState({ searchString: queryRaw, page }, "", u.toString());
    else history.replaceState({ searchString: queryRaw, page }, "", u.toString());
  }

  function readUrlParams() {
    const params = new URLSearchParams(window.location.search);
    const q = params.get("searchString") || "";
    const p = parseInt(params.get("page") || "1", 10);
    return { q, p: (Number.isFinite(p) && p > 0) ? p : 1 };
  }

  function buildPageWindow(current, total, maxButtons=10) {
    if (total <= maxButtons) {
      return { pages: Array.from({length: total}, (_,i)=>i+1), leftEllipsis:false, rightEllipsis:false };
    }

    const half = Math.floor(maxButtons / 2);
    let start = current - half;
    let end = current + (maxButtons - half - 1);

    if (start < 1) { start = 1; end = start + maxButtons - 1; }
    if (end > total) { end = total; start = end - maxButtons + 1; }

    const pages = [];
    for (let i = start; i <= end; i++) pages.push(i);

    return { pages, leftEllipsis: start > 1, rightEllipsis: end < total, start, end };
  }

  function gotoPage(p, pushUrl=true) {
    currentPage = clampPage(p);
    setUrlParams(currentQueryRaw, currentPage, pushUrl);
    renderPage();
    window.scrollTo({ top: 0, behavior: "smooth" });
  }

  function renderPager() {
    const totalPages = getTotalPages();
    currentPage = clampPage(currentPage);

    btnPrev.disabled = currentPage <= 1;
    btnNext.disabled = currentPage >= totalPages;

    pageNumbers.innerHTML = "";

    const win = buildPageWindow(currentPage, totalPages, 10);

    if (win.leftEllipsis) {
      const b1 = document.createElement("span");
      b1.className = "page-num";
      b1.textContent = "1";
      b1.addEventListener("click", () => gotoPage(1, true));
      pageNumbers.appendChild(b1);

      const ell = document.createElement("span");
      ell.className = "page-ellipsis";
      ell.textContent = "...";
      pageNumbers.appendChild(ell);
    }

    win.pages.forEach(p => {
      const b = document.createElement("span");
      b.className = "page-num" + (p === currentPage ? " active" : "");
      b.textContent = String(p);
      if (p !== currentPage) b.addEventListener("click", () => gotoPage(p, true));
      pageNumbers.appendChild(b);
    });

    if (win.rightEllipsis) {
      const ell = document.createElement("span");
      ell.className = "page-ellipsis";
      ell.textContent = "...";
      pageNumbers.appendChild(ell);

      const bt = document.createElement("span");
      bt.className = "page-num";
      bt.textContent = String(totalPages);
      bt.addEventListener("click", () => gotoPage(totalPages, true));
      pageNumbers.appendChild(bt);
    }
  }

  function renderPage() {
    const totalHits = allItems.length;

    renderPager();

    if (!currentQueryRaw.trim()) {
      status.textContent = "검색어가 없습니다. 상단 검색창에 입력하세요.";
      results.innerHTML = "";
      return;
    }

    if (totalHits === 0) {
      status.textContent = `검색 결과 0개 - query: ${currentQueryRaw}`;
      results.innerHTML = "<li>다른 검색어를 사용하세요.</li>";
      return;
    }

    status.textContent = `검색 결과 ${totalHits}개 (${pageSize}개씩 페이징) - query: ${currentQueryRaw}`;

    const q_norm = currentQueryRaw.trim().toLowerCase();
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

  function runSearch(queryRaw, pushUrl=false) {
    const q_raw = (queryRaw || "").trim();
    const q_norm = q_raw.toLowerCase();

    currentQueryRaw = q_raw;

    if (!q_norm) {
      allItems = [];
      currentPage = 1;
      setUrlParams("", 1, pushUrl);
      renderPage();
      return;
    }

    allItems = idx.search(q_norm);
    currentPage = 1;

    setUrlParams(q_raw, currentPage, pushUrl);
    renderPage();
    window.scrollTo({ top: 0, behavior: "smooth" });
  }

  function debounce(fn, wait) {
    let t = null;
    return function (...args) {
      if (t) clearTimeout(t);
      t = setTimeout(() => fn.apply(this, args), wait);
    };
  }

  const onInputDebounced = debounce(() => {
    runSearch(box ? box.value : "", true);
  }, debounceMs);

  try {
    status.textContent = "검색 인덱스 로딩 중";

    const url = "{{ '/search.json' | relative_url }}";
    const res = await fetch(url);
    if (!res.ok) throw new Error("search.json fetch failed: " + res.status);

    docs = await res.json();
    status.textContent = `문서 ${docs.length}개 로딩 완료`;

    idx = lunr(function () {
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

    // 초기 URL 반영
    const { q, p } = readUrlParams();
    if (box) box.value = q;

    if (q.trim()) {
      allItems = idx.search(q.trim().toLowerCase());
      currentQueryRaw = q.trim();
      currentPage = clampPage(p);
      setUrlParams(currentQueryRaw, currentPage, false);
      renderPage();
    } else {
      currentQueryRaw = "";
      allItems = [];
      currentPage = 1;
      setUrlParams("", 1, false);
      renderPage();
    }

    // Prev/Next
    btnPrev.addEventListener("click", () => {
      if (currentPage > 1) gotoPage(currentPage - 1, true);
    });
    btnNext.addEventListener("click", () => {
      const tp = getTotalPages();
      if (currentPage < tp) gotoPage(currentPage + 1, true);
    });

    // 입력 debounce
    if (box) box.addEventListener("input", onInputDebounced);

    // 뒤로가기/앞으로가기
    window.addEventListener("popstate", () => {
      const { q, p } = readUrlParams();
      if (box) box.value = q;

      if (!q.trim()) {
        currentQueryRaw = "";
        allItems = [];
        currentPage = 1;
        renderPage();
        return;
      }

      currentQueryRaw = q.trim();
      allItems = idx.search(currentQueryRaw.toLowerCase());
      currentPage = clampPage(p);
      renderPage();
      window.scrollTo({ top: 0, behavior: "smooth" });
    });

  } catch (e) {
    console.error(e);
    status.textContent = "Search error: " + String(e);
    results.innerHTML = "";
  }
})();
</script>

