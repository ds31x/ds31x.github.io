<script src="https://unpkg.com/lunr/lunr.js"></script>
<script>
(async function () {
  const results = document.getElementById("results");
  const box = document.getElementById("searchBox");

  // 0) 스크립트가 실행되는지부터 화면에 표시
  results.innerHTML = "<li>search.js loaded</li>";

  try {
    // 1) search.json 로드
    const url = "{{ '/search.json' | relative_url }}";
    const res = await fetch(url);
    results.innerHTML = `<li>fetch ${url} -> ${res.status}</li>`;

    const docs = await res.json();
    results.innerHTML += `<li>docs loaded: ${docs.length}</li>`;

    // 2) Lunr 인덱스 생성
    const idx = lunr(function () {
      this.ref("url");
      this.field("title");
      this.field("content");
      docs.forEach(d => this.add(d));
    });
    results.innerHTML += "<li>lunr index built</li>";

    function render(items, q) {
      if (!items.length) {
        results.innerHTML = `<li>0 results for: ${q}</li>`;
        results.innerHTML += `<li>try: MathJax, ROOT, latex, vim</li>`;
        return;
      }
      results.innerHTML = `<li>hits: ${items.length} for: ${q}</li>` +
        items.slice(0, 30).map(i => {
          const d = docs.find(x => x.url === i.ref);
          return `<li><a href="${d.url}">${d.title}</a></li>`;
        }).join("");
    }

    function doSearch(q) {
      q = (q || "").trim();
      if (!q) {
        results.innerHTML = "<li>type a query</li>";
        return;
      }
      const hits = idx.search(q);
      render(hits, q);
    }

    // 3) URL 파라미터로 검색 수행
    const params = new URLSearchParams(window.location.search);
    const query = params.get("searchString");

    if (query) {
      box.value = query;
      doSearch(query);
    } else {
      results.innerHTML += "<li>no query param</li>";
    }

    // 4) 입력 중 실시간 검색
    box.addEventListener("input", () => doSearch(box.value));

  } catch (e) {
    results.innerHTML = `<li>Search error: ${String(e)}</li>`;
  }
})();
</script>

