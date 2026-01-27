---
layout: default
title: Search
permalink: /search/
---

<ul id="results"></ul>

<script src="https://unpkg.com/lunr/lunr.js"></script>
<script>
(async function () {
  const res = await fetch("{{ '/search.json' | relative_url }}");
  const docs = await res.json();

  const idx = lunr(function () {
    this.ref("url");
    this.field("title");
    this.field("content");
    docs.forEach(d => this.add(d));
  });

  const box = document.getElementById("searchBox");
  const results = document.getElementById("results");

  function render(items) {
    results.innerHTML = items.slice(0, 30).map(i => {
      const d = docs.find(x => x.url === i.ref);
      return `<li><a href="${d.url}">${d.title}</a></li>`;
    }).join("");
  }

  function doSearch(q) {
    q = (q || "").trim();
    if (!q) { results.innerHTML = ""; return; }
    render(idx.search(q));
  }

  // URL 파라미터 searchString 읽기
  const params = new URLSearchParams(window.location.search);
  const query = params.get("searchString");

  if (query) {
    box.value = query;
    doSearch(query);
  }

  box.addEventListener("input", () => doSearch(box.value));
})();
</script>

