<script src="https://unpkg.com/lunr/lunr.js"></script>
<script>
(async function () {
  const results = document.getElementById("results");
  const box = document.getElementById("searchBox");

  try {
    const res = await fetch("{{ '/search.json' | relative_url }}");
    if (!res.ok) throw new Error("search.json fetch failed: " + res.status);

    const docs = await res.json();
    if (!Array.isArray(docs)) throw new Error("search.json is not an array");

    const idx = lunr(function () {
      this.ref("url");
      this.field("title");
      this.field("content");
      docs.forEach(d => this.add(d));
    });

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

    const params = new URLSearchParams(window.location.search);
    const query = params.get("searchString");

    if (query) {
      box.value = query;
      doSearch(query);
    }

    box.addEventListener("input", () => doSearch(box.value));

  } catch (e) {
    results.innerHTML = "<li>Search error: " + String(e) + "</li>";
    console.error(e);
  }
})();
</script>

