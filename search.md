---
layout: default
title: Search
permalink: /search/
---

<ul id="results"></ul>

<script src="https://unpkg.com/lunr/lunr.js"></script>
<script>
(async function () {
  try {
    const results = document.getElementById("results");
    const box = document.getElementById("searchBox");

    // 스크립트 실행 확인
    results.innerHTML = "<li>search.js loaded</li>";

    const url = "{{ '/search.json' | relative_url }}";
    const res = await fetch(url);
    results.innerHTML += `<li>fetch ${url} -> ${res.status}</li>`;

    const docs = await res.json();
    results.innerHTML += `<li>docs loaded: ${docs.length}</li>`;

    const idx = lunr(function () {
      this.ref("url");
      this.field("title");
      this.field("content");
      docs.forEach(d => this.add(d));
    });
    results.innerHTML += "<li>lunr index built</li>";

    function render(items, q) {
      if (!items.length) {
        results.innerHTML = `<li>0 results for: ${q}</li><li>try: MathJax, ROOT, latex, vim</li>`;
        return;
      }
      results.innerHTML =
        `<li>hits: ${items.length} for: ${q}</li>` +
        items.slice(0, 30).map(i => {
          const d = docs.find(x => x.url === i.ref);
          return `<li><a href="${d.url}">${d.title}</a></li>`;
        }).join("");
    }

    function doSearch(q) {
      q = (q || "").trim();
      if (!q) { results.innerHTML = "<li>type a query</li>"; return; }
      render(idx.search(q), q);
    }

    const params = new URLSearchParams(window.location.search);
    const query = params.get("searchString");

    if (query) {
      box.value = query;
      doSearch(query);
    } else {
      results.innerHTML += "<li>no query param</li>";
    }

    box.addEventListener("input", () => doSearch(box.value));
  } catch (e) {
    // results가 없을 때도 표시되게 최소한 alert
    console.error(e);
    alert("Search error: " + e);
  }
})();
</script>

