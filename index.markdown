---
layout: default
title: Blog
---

<section class="showcase-hero">
  <h1>Welcome To Limaomao Blog</h1>
  <p>记录代码、学习和有趣发现</p>
</section>

<section class="showcase-controls">
  <div class="filter-tabs" id="filter-tabs">
    <button class="filter-tab is-active" data-filter="all" type="button">All Posts</button>
    {% assign sorted_categories = site.categories | sort %}
    {% for category in sorted_categories %}
      <button class="filter-tab" data-filter="{{ category[0] | downcase }}" type="button">{{ category[0] }}</button>
    {% endfor %}
  </div>
  <div class="search-box">
    <input id="post-search" type="search" placeholder="Search posts" />
  </div>
</section>

<section class="post-grid" id="post-grid">
  {% for post in site.posts %}
    <article class="post-card"
      data-title="{{ post.title | downcase | escape }}"
      data-categories="{{ post.categories | join: ' ' | downcase | escape }}"
      data-excerpt="{{ post.excerpt | strip_html | downcase | escape }}">
      <a class="post-card-link" href="{{ post.url | relative_url }}">
        {% if post.image %}
          <img class="post-card-cover" src="{{ post.image | relative_url }}" alt="{{ post.title }}">
        {% else %}
          <div class="post-card-cover placeholder"></div>
        {% endif %}
        <div class="post-card-body">
          <h3>{{ post.title }}</h3>
          <p>{{ post.excerpt | strip_html | truncate: 110 }}</p>
          <div class="post-card-meta">
            <span>{{ post.date | date: "%Y-%m-%d" }}</span>
            {% if post.categories and post.categories.size > 0 %}
              <span>{{ post.categories | first }}</span>
            {% else %}
              <span>uncategorized</span>
            {% endif %}
          </div>
        </div>
      </a>
    </article>
  {% endfor %}
</section>

<script>
  (function () {
    var cards = Array.prototype.slice.call(document.querySelectorAll(".post-card"));
    var tabs = Array.prototype.slice.call(document.querySelectorAll(".filter-tab"));
    var searchInput = document.getElementById("post-search");
    var activeFilter = "all";

    function applyFilter() {
      var q = (searchInput.value || "").toLowerCase().trim();
      cards.forEach(function (card) {
        var title = card.getAttribute("data-title") || "";
        var categories = card.getAttribute("data-categories") || "";
        var excerpt = card.getAttribute("data-excerpt") || "";
        var matchesFilter = activeFilter === "all" || categories.indexOf(activeFilter) !== -1;
        var matchesSearch = !q || title.indexOf(q) !== -1 || categories.indexOf(q) !== -1 || excerpt.indexOf(q) !== -1;
        card.style.display = matchesFilter && matchesSearch ? "" : "none";
      });
    }

    tabs.forEach(function (tab) {
      tab.addEventListener("click", function () {
        tabs.forEach(function (btn) { btn.classList.remove("is-active"); });
        tab.classList.add("is-active");
        activeFilter = tab.getAttribute("data-filter") || "all";
        applyFilter();
      });
    });

    searchInput.addEventListener("input", applyFilter);
    applyFilter();
  })();
</script>
