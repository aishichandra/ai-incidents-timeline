<script>
  import { onMount } from 'svelte';
  import { fade } from 'svelte/transition';
  import TimelineCard from './components/TimelineCard.svelte';
  import Tooltip from './components/Tooltip.svelte';

  let timelineData = [];
  let offset = null;
  let loading = false;
  let hasMore = true;

  let selectedCategories = [];
  let selectedPlatforms = [];
  let startDate = "";
  let endDate = "";
  let searchQuery = '';

  let dropdownOpen = false;
  let platformDropdownOpen = false;
  let dateFilterOpen = false;
  let showBackToTop = false;

  let sentinel;
  let observer;

  const categoryDefinitions = {
    "advertising product": "Development, launch, or changes in advertising products",
    "AI / automation": "Integration of artificial intelligence and automation technologies within platforms’ or publishers’ operations; noteworthy advancements in the broader AI landscape; partnerships between AI companies and publishers; news related to AI-generated content",
    "AI incidents" : "Instances of harms (or near-harms) related to AI and journalism",
    "algorithm": "Changes and updates made to the platform's algorithms, impacting how content is distributed, ranked, or presented",
    "alt-tech": "Updates related to social platforms popular among the alt-right, far-right and others who espouse extremism or fringe theories",
    "AR / VR": "Incorporation and advancement of Augmented Reality (AR) and Virtual Reality (VR) technologies within the platform's framework",
    "audience engagement": "User engagement metrics, features, or strategies",
    "bug / glitch / issue": "Technical glitches, bugs, or issues that impact the platform's functionality",
    "business": "Corporate activities such as acquisitions, rebrands, and strategic shifts, highlighting significant changes in the platform's business landscape",
    "content moderation": "Changes or controversies surrounding the platform's policies and practices for moderating content",
    "COVID-19": "Impact of the COVID-19 pandemic on the platform, including changes in user behavior, policies, or response strategies",
    "data / security": "Data breaches, security measures, or changes in data handling policies and practices",
    "elections / civic integrity": "Platform involvement in elections, including measures taken to prevent misinformation, political advertising policies, and responses to election-related events",
    "fediverse": "Updates related to decentralized social networking platforms that allow independent servers to communicate with each other through the ActivityPub protocol",
    "mis / disinformation": "Efforts and controversies surrounding the platform's attempts to combat or handle misinformation",
    "mission statement": "Updates or changes in the platform's mission and values",
    "newsletters": "Updates related to newsletter products or related audience engagement strategies",
    "partnership": "Partnerships within and between tech companies and news organizations (or lack thereof)",
    "philanthropy": "Charitable initiatives, donations, or philanthropic efforts undertaken by the platform",
    "platform as publisher": "Instances where platforms themselves engage in publishing content, blurring the lines between platform and publisher",
    "platform competition": "Competitor-related events, market dynamics, or strategic moves in response to competition",
    "platform personnel": "Changes in key personnel within the platform, such as CEOs, executives, or notable hires",
    "policy change": "Changes in terms of service, rules, or overall policies governing user behavior on the platform",
    "product": "Launches or updates to platform features, tools, or services",
    "regulation / legal dispute": "Legal challenges, regulatory scrutiny, or changes in legislation affecting the platform",
    "research / investigation": "Studies, investigations, or research initiatives undertaken by the platform or external parties related to its impact or practices",
    "revenue / funding": "Updates about platforms’ or publishers’ income or earnings"
  };


  const AIRTABLE_URL = "https://api.airtable.com/v0/appbtcs0TcBJOaflz/tbl8VV3Klj4Icfkb8";
  const AIRTABLE_HEADERS = {
    Authorization: "Bearer patUw9YjcnlCQoIZr.8ed788e6012e4870566558b401cc31a8304d0c360d7dcd2fefc69ad1e8f6e1af"
  };

  function extractUrl(input) {
    if (typeof input !== "string") return null;
    const markdownMatch = input.match(/\[.*?\]\((.*?)\)/);
    if (markdownMatch) return markdownMatch[1];
    const fallbackMatch = input.match(/https?:\/\/[^\s)]+/);
    return fallbackMatch ? fallbackMatch[0] : null;
  }

  function extractDomain(url) {
    try {
      if (!url || !url.startsWith("http")) return null;
      const parsedUrl = new URL(url);
      let hostname = parsedUrl.hostname;

      if (hostname === "web.archive.org") {
        const parts = parsedUrl.pathname.split("/").slice(3).join("/");
        const fullOriginalUrl = parts.startsWith("http") ? parts : `https://${parts}`;
        hostname = new URL(decodeURIComponent(fullOriginalUrl)).hostname;
      }

      return hostname.replace(/^www\./, "");
    } catch {
      return null;
    }
  }

  async function fetchNextBatch() {
    if (loading || !hasMore) return;
    loading = true;
    try {
      const url = new URL(AIRTABLE_URL);
      if (offset) url.searchParams.append("offset", offset);
      url.searchParams.append("view", "viwlZ0icRGkfLhux6");
      url.searchParams.append("pageSize", 100);

      const res = await fetch(url.toString(), { headers: AIRTABLE_HEADERS });

      if (!res.ok) {
        throw new Error(`Error fetching Airtable data: ${res.status}`);
      }

      const data = await res.json();

      const newRecords = data.records
        .filter(r => r.fields.date && r.fields.desc)
        .map(r => {
          const rawLink = r.fields.source_url_1;
          let links = [];

          if (Array.isArray(rawLink)) {
            links = rawLink.map(link => {
              const url = extractUrl(link);
              const domain = extractDomain(url);
              return url && domain ? { text: domain, url } : null;
            }).filter(Boolean);
          } else if (typeof rawLink === "string") {
            const url = extractUrl(rawLink);
            const domain = extractDomain(url);
            links = url && domain ? [{ text: domain, url }] : [];
          }

          return {
            id: r.id,
            date: r.fields.date || "Unknown date",
            platform: r.fields.platform || "Unknown platform",
            categories: r.fields.category || [],
            title: r.fields['AI incident title'] || "(Untitled)",
            description: r.fields.desc || "(No description provided)",
            links: links.map(link => ({
              text: link.text || "View Source",
              url: link.url || "#"
            }))
          };
        });

      timelineData = [...timelineData, ...newRecords];
      offset = data.offset;
      hasMore = Boolean(data.offset);
    } catch (err) {
      console.error("Error fetching batch:", err);
      hasMore = false;
    } finally {
      loading = false;
    }
  }

  async function fetchAllData() {
    offset = null;
    hasMore = true;
    timelineData = [];
    while (hasMore) {
      await fetchNextBatch();
    }
  }

  function filterByCategories(data, selected) {
    if (!selected.length) return data;
    return data.filter(item => item.categories.some(cat => selected.includes(cat)));
  }

  function filterByPlatforms(data, selected) {
    if (!selected.length) return data;
    return data.filter(item => {
      const platforms = Array.isArray(item.platform) ? item.platform : [item.platform];
      return platforms.some(p => selected.includes(p));
    });
  }

  function filterByDateRange(data, start, end) {
    if (!start && !end) return data;

    const startTime = start ? new Date(start).getTime() : -Infinity;
    const endTime = end ? new Date(new Date(end).setHours(23, 59, 59, 999)).getTime() : Infinity;

    return data.filter(item => {
      const itemTime = new Date(item.date).getTime();
      return itemTime >= startTime && itemTime <= endTime;
    });
  }

  function filterBySearch(data, query) {
    if (!query) return data;
    const searchTerm = query.toLowerCase();
    return data.filter(item =>
      item.description.toLowerCase().includes(searchTerm) ||
      item.title.toLowerCase().includes(searchTerm)
    );
  }

  function groupByMonth(data) {
    const groups = {};
    data.forEach(item => {
      const dateObj = new Date(item.date);
      const key = `${dateObj.toLocaleString('default', { month: 'long' })} ${dateObj.getFullYear()}`;
      groups[key] = groups[key] || [];
      groups[key].push(item);
    });

    const sortedGroups = {};
    Object.keys(groups)
      .sort((a, b) => new Date(b) - new Date(a))
      .forEach(key => {
        sortedGroups[key] = groups[key];
      });

    return sortedGroups;
  }

  function handleClickOutside(event) {
    if (!event.target.closest('.dropdown')) {
      dropdownOpen = false;
      platformDropdownOpen = false;
      dateFilterOpen = false;
    }
  }

  function scrollToTop() {
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }

  function exportToCSV(data) {
    const headers = ['Date', 'Title', 'Platform', 'Categories', 'Description', 'Sources'];

    const rows = data.map(item => [
      item.date,
      item.title,
      Array.isArray(item.platform) ? item.platform.join('; ') : item.platform,
      item.categories.join('; '),
      item.description,
      item.links.map(link => link.url).join('; ')
    ]);

    const csv = [
      headers.join(','),
      ...rows.map(row =>
        row.map(cell =>
          `"${String(cell).replace(/"/g, '""')}"`
        ).join(',')
      )
    ].join('\n');

    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const link = document.createElement('a');
    const url = URL.createObjectURL(blob);
    link.setAttribute('href', url);
    link.setAttribute('download', `timeline-export-${new Date().toISOString().split('T')[0]}.csv`);
    link.style.display = 'none';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }

  onMount(() => {
    fetchAllData();
    document.addEventListener('click', handleClickOutside);
    window.addEventListener('scroll', () => {
      showBackToTop = window.scrollY > 300;
    });

    observer = new IntersectionObserver(entries => {
      entries.forEach(entry => {
        if (entry.isIntersecting && hasMore && !loading) {
          fetchNextBatch();
        }
      });
    });

    if (sentinel) {
      observer.observe(sentinel);
    }

    return () => {
      document.removeEventListener('click', handleClickOutside);
      if (observer && sentinel) observer.unobserve(sentinel);
    };
  });

  $: uniqueCategories = Array.from(new Set(timelineData.flatMap(item => item.categories)))
    .sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()));
  $: uniquePlatforms = Array.from(new Set(timelineData.flatMap(item =>
    Array.isArray(item.platform) ? item.platform : [item.platform]
  ))).sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()));

  $: filteredData = filterBySearch(
    filterByDateRange(
      filterByCategories(
        filterByPlatforms(timelineData, selectedPlatforms),
        selectedCategories
      ),
      startDate,
      endDate
    ),
    searchQuery
  );

  $: groupedData = groupByMonth(filteredData);
</script>

<div class="article">
  <div class="header">
    <div class="title-container">
      <div class="title-row">
        <div class="logo">
          <img src="Tow Center Logo.webp" alt="Tow Center Logo" />
        </div>
        <span id="title-style">Tow Center for Digital Journalism's</span>
      </div>
      <h1 class="article-title">Platforms and Publishers Timeline</h1>
    </div>
  </div>

  <p class="article-intro">
    The timeline below, compiled by the Tow Center for Digital Journalism <a href="https://towcenter.columbia.edu/news/platforms-and-publishers" >as part of the Platforms and Publishers project</a>, identifies key developments in the relationship between technology platforms and news publishers. 
  </p>

  <p class="article-intro">
    This timeline contains updates related to the following categories, as they relate to platforms and publishers:
    <span class="category-tags">
      {#each Object.entries(categoryDefinitions) as [category, definition]}
        <Tooltip text={definition}>
          <span class="category-tag">
            {category}
          </span>
        </Tooltip>
      {/each}
    </span>
  </p>

  <p class="article-intro">
    You can filter by any of the categories above. You can also filter the timeline to search by specific platforms. Updates that pertain broadly to social platforms, AI companies, or newsroom tools are categorized under those respective platform groupings:
    <span class="platform-tags">
    <Tooltip text="Updates that apply to the overall landscape of mainstream social media platforms like Facebook, Twitter, Instagram, etc.">
      <span class="platform-tag">social platforms</span>
    </Tooltip>
    <Tooltip text="Developments affecting AI providers as a whole, such as OpenAI, Anthropic, or Google AI">
      <span class="platform-tag">AI companies</span>
    </Tooltip>
    <Tooltip text="Technology initiatives and tools developed internally by newsrooms, such as CMS updates or automation workflows">
      <span class="platform-tag">newsroom tools</span>
    </Tooltip>
    </span>
  </p>

  <p class="article-intro">
    The timeline, which begins in October 2000, is currently maintained by <a href="https://towcenter.columbia.edu/content/klaudia-jazwinska">Klaudia Jaźwińska</a> and is updated at the beginning of each month. We welcome any feedback or suggestions regarding missed developments.
  </p>

  <div class="article-footer">

    <p class="update-date">
      Updated on {new Date().toLocaleDateString('en-US', { month: 'long', day: 'numeric', year: 'numeric' })}
    </p>
  </div>
</div>

<div class="filters">
  <div class="filter-container">
    <div class="dropdown date-filter">
      <div class="dropdown-header" on:click={() => (dateFilterOpen = !dateFilterOpen)}>
        {#if startDate || endDate}
          Date: {startDate ? new Date(startDate).toLocaleDateString() : 'Start'} - 
              {endDate ? new Date(endDate).toLocaleDateString() : 'End'}
        {:else}
          Filter by Date
        {/if}
        <span class="dropdown-arrow">{dateFilterOpen ? "▲" : "▼"}</span>
      </div>
      
      {#if dateFilterOpen}
        <div class="dropdown-menu date-menu">
          <div class="date-inputs">
            <label>
              Start Date
              <input type="date" bind:value={startDate} />
            </label>
            <label>
              End Date
              <input 
                type="date" 
                bind:value={endDate}
                min={startDate} 
              />
            </label>
          </div>
          {#if startDate || endDate}
            <button class="clear-button" on:click={() => { startDate = ""; endDate = ""; }}>
              Clear Dates
            </button>
          {/if}
        </div>
      {/if}
    </div>
    <div class="dropdown">
      <div class="dropdown-header" on:click={() => (dropdownOpen = !dropdownOpen)}>
        {#if selectedCategories.length > 0}
        Categories: {selectedCategories
          .slice()
          .sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()))
          .join(", ")}
        {:else}
          Filter Categories
        {/if}
        <span class="dropdown-arrow">{dropdownOpen ? "▲" : "▼"}</span>
      </div>
      {#if dropdownOpen}
        <div class="dropdown-menu">
          {#each uniqueCategories as category}
          <label
            class="dropdown-item {selectedCategories.includes(category) ? 'selected' : ''}"
          >
            <input
              type="checkbox"
              value={category}
              bind:group={selectedCategories}
            />
            <span>{category}</span>
          </label>
        
          {/each}
        </div>
      {/if}
    </div>

    <div class="dropdown">
      <div class="dropdown-header" on:click={() => (platformDropdownOpen = !platformDropdownOpen)}>
        {#if selectedPlatforms.length > 0}
          Platforms: {selectedPlatforms.slice().sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase())).join(", ")}
        {:else}
          Filter Platforms
        {/if}
        <span class="dropdown-arrow">{platformDropdownOpen ? "▲" : "▼"}</span>
      </div>
      {#if platformDropdownOpen}
        <div class="dropdown-menu">
          {#each uniquePlatforms as platform}
            <label class="dropdown-item">
              <input type="checkbox" value={platform} bind:group={selectedPlatforms} />
              <span>{platform}</span>
            </label>
          {/each}
        </div>
      {/if}
    </div>

    
  </div>

  <div class="search-container">
    <div class="search-wrapper">
      <input
        type="text"
        bind:value={searchQuery}
        placeholder="Search in timeline..."
        class="search-input"
      />
      {#if searchQuery}
        <button 
          class="clear-search" 
          on:click={() => searchQuery = ''}
          aria-label="Clear search"
        >
          ✕
        </button>
      {/if}
    </div>

    <div class="export-container">
      <button 
        class="export-button"
        on:click={() => exportToCSV(filteredData)}
        disabled={filteredData.length === 0}
      >
        {filteredData.length === 0 
          ? 'No data to export' 
          : `Export ${filteredData.length} items to CSV`}
      </button>

    </div>
  </div>
</div>


<div class="timeline-container">
  {#each Object.entries(groupByMonth(filteredData)) as [monthYear, items]}
    <div class="timeline-row">
      <div class="month-label">
        <span>{monthYear.split(' ')[0]}</span>
        <span class="year-label">{monthYear.split(' ')[1]}</span>
      </div>
      <div class="timeline-divider"></div>
      <div class="month-group">
        {#each items as item (item.id)}
          <div class="card-wrapper" in:fade={{ duration: 300 }} out:fade={{ duration: 300 }}>
            <TimelineCard
              date={item.date}
              title={item.title}
              platform={item.platform}
              tags={item.categories}
              description={item.description}
              links={item.links}
            />
          </div>
        {/each}
      </div>
    </div>
  {/each}

  <!-- Sentinel and loading states moved here -->
  <div bind:this={sentinel} class="sentinel"></div>

  {#if loading}
    <div class="loading">Loading more...</div>
  {/if}

  {#if !hasMore && timelineData.length > 0}
    <div class="end-msg">All items loaded.</div>
  {/if}

  {#if showBackToTop}
    <button
      class="back-to-top"
      on:click={scrollToTop}
      aria-label="Back to top"
      transition:fade={{ duration: 200 }}
    >
      ↑ Back to Top
    </button>
  {/if}
</div>

<style>
  /* Global Styles */
  :global(body) {
    background: #fff;
    color: #333;
    font-family: "EB Garamond", serif;
    font-size: 20px;
    line-height: 1.5;
    margin: 0;
    padding: 0;
  }

  /* Article Layout */
  .article {
    max-width: 680px;
    margin: 4rem auto;
    padding: 0 1rem;
    
  }

  .article-title {
    font-family: "Helvetica Neue", sans-serif;
    font-size: 3.4rem;
    line-height: 1.2;
    font-weight: 800;
    margin-bottom: 2rem;
    color: #d6613b;
    text-align: left;
    letter-spacing: -0.02em;
    margin-bottom: 2rem;
  }

  .article-intro {
    font-size: 1.2rem;
    font-weight: 400;
    color: #333;
    margin-bottom: 1.5rem;
    text-align: left;
    line-height: 1.8rem;
  }

  .article-footer {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 1rem;
    margin: 2rem 0;
    padding-top: 1rem;
    border-top: 1px solid rgba(67, 72, 90, 0.1);
  }

  .report-button {
    display: inline-block;
    padding: 0.75rem 1.5rem;
    background-color: #d6613b;
    color: white;
    text-decoration: none;
    font-family: 'Helvetica Neue', sans-serif;
    font-size: 0.875rem;
    font-weight: 600;
    transition: all 0.2s ease;
    border: 1px solid #d6613b;
  }

  .report-button:hover {
    background-color: white;
    color: #d6613b;
  }

  .update-date {
    font-family: 'Helvetica Neue', sans-serif;
    font-size: 0.875rem;
    color: #43485A;
    margin: 0;
  }

  /* Tags Styling */
  .category-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin: 1rem 0 2rem;
    align-items: center;
  }

  .platform-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin: 1rem 0 2rem;
    align-items: center;
  }

  .category-tag,
  .platform-tag {
    font-size: 0.75rem;
    font-family: "Helvetica Neue", sans-serif;
    padding: 0.15rem 0.75rem;
    color: #43485A;
    cursor: help;
    display: inline-block;
    transition: all 0.2s ease;
    
  }

  .category-tag {
    background-color: rgb(214, 97, 59, 0.2);
    border: 1px solid rgba(214, 97, 59);
  }

  .platform-tag {
    background-color: rgba(67, 72, 90, 0.1);
    border: 1px solid rgba(67, 72, 90);
  }

  .category-tag:hover,
  .platform-tag:hover {
    transform: translateY(-1px);
    box-shadow: 3px 3px 0 0 #d6613b;
    background-color: white;
    font-weight: 500;
  }
  

  /* Timeline Layout */
  .timeline-container {
    max-width: 900px;
    margin: 0 auto;
    padding: 2rem 1rem 4rem; /* Reduced top padding */
  }

  .timeline-row {
    display: grid;
    grid-template-columns: 100px 2px 1fr;
    gap: 2rem;
    align-items: flex-start;
    margin-bottom: 4rem;
  }

  .timeline-divider {
    background: #D9626B;
    width: 1px;
    height: 100%;
  }

  /* Month Labels */
  .month-label {
    font-family: 'EB Garamond', serif;
    font-size: 2rem;
    font-weight: 600;
    color: #111;
    text-transform: capitalize;
    position: sticky;
    top: calc(200px + 1rem); /* Adjust this value based on your filters' height */
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    padding-right: 0.5rem;
    z-index: 900; 
    padding-top: 1.5rem; 
        
  }

  .year-label {
    font-size: 1.25rem;
    color: #555;
    margin-top: 0.25rem;
  }

  .month-group {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  /* Filters and Search */
  .filter-container,
  .search-container {
    font-family: 'helvetica neue', sans-serif;
    max-width: 900px;
    margin: 2rem auto;
    padding: 0 1rem;
    display: flex;
    gap: 1rem;
    flex-wrap: wrap;
    align-items: flex-start;
  }

  /* Dropdown Styles */
  .dropdown {
    flex: 1;
    position: relative;
    font-family: 'Helvetica Neue', sans-serif;
  }

  .dropdown-header {
    background: white;
    border: 1px solid #43485A;
    padding: 0.75rem 1rem;
    font-size: 0.875rem;
    font-weight: 600;
    color: #43485A;
    cursor: pointer;
    display: flex;
    justify-content: space-between;
    align-items: center;
    transition: all 0.2s ease;
  }

  .dropdown-header:hover {
    background-color: rgba(67, 72, 90, 0.1);
  }

  .dropdown-menu {
    position: absolute;
    top: calc(100% + 4px);
    left: 0;
    right: 0;
    background: white;
    border: 1px solid #43485A;
    padding: 0.5rem;
    max-height: 300px;
    overflow-y: auto;
    z-index: 100;
    box-shadow: 4px 4px 0 0 #d6613b;
  }

  .dropdown-item {
    display: flex;
    align-items: center;
    padding: 0.5rem;
    cursor: pointer;
    transition: background-color 0.2s ease;
  }

  .dropdown-item:hover {
    background-color: rgba(67, 72, 90, 0.1);
  }

  .dropdown-item input[type="checkbox"] {
    margin-right: 0.75rem;
  }

  .dropdown-item span {
    font-size: 0.875rem;
    color: #43485A;
  }

  .dropdown-arrow {
    font-size: 0.75rem;
    margin-left: 0.5rem;
  }

  /* Selected state styling */
  .dropdown-item.selected {
    background-color: rgba(214, 97, 59, 0.1);
  }

  .dropdown-item.selected span {
    font-weight: 600;
  }

  /* Mobile responsiveness */
  @media (max-width: 768px) {
    .dropdown {
      width: 100%;
      margin-bottom: 0.5rem;
    }

    .dropdown-menu {
      max-height: 250px;
    }
  }

  /* Update the date filter styles */
  .date-filter {
    flex: 1;
  }

  .date-menu {
    padding: 1rem;
  }

  .date-inputs {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .date-inputs label {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    font-size: 0.75rem;
    font-weight: 600;
    color: #43485A;
  }

  .date-inputs input[type="date"] {
    padding: 0.5rem;
    border: 1px solid #43485A;
    font-family: "Helvetica Neue", sans-serif;
    font-size: 0.875rem;
    width: 100%;
    box-sizing: border-box;
  }

  .clear-button {
    margin-top: 1rem;
    width: 100%;
    padding: 0.5rem;
    background: white;
    border: 1px solid #d6613b;
    color: #d6613b;
    font-family: "Helvetica Neue", sans-serif;
    font-size: 0.75rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .clear-button:hover {
    background-color: #fdf2ec;
  }

  /* Search Input */
  .search-wrapper {
    flex: 2;
    position: relative;
    font-family: 'Helvetica Neue', sans-serif;
  }

  .search-input {
    width: 100%;
    padding: 0.75rem 1rem;
    font-size: 0.875rem;
    font-weight: 500;
    border: 1px solid #43485A;
    background: white;
    color: #43485A;
    font-family: 'Helvetica Neue', sans-serif;
    transition: all 0.2s ease;
    box-sizing: border-box;
  }

  .search-input:hover {
    background-color: rgba(67, 72, 90, 0.1);
  }

  .search-input::placeholder {
    color: #43485A;
    font-weight: 600;
  }

  .clear-search {
    position: absolute;
    right: 1rem;
    top: 50%;
    transform: translateY(-50%);
    border: none;
    background: none;
    color: #43485A;
    font-size: 0.875rem;
    cursor: pointer;
    padding: 0.25rem;
  }

  .clear-search:hover {
    color: #d6613b;
  }

  /* id title-style */
  #title-style {
    font-family: 'EB Garamond', serif;
    /* italic */
    /* font-style: italic; */
    font-size: 2rem;
    font-weight: 400;
    color: #d6613b;
    text-align: left;
    letter-spacing: 0em;
  }

  /* Export Button */
  .export-container {
    flex: 1;
    display: flex;
    justify-content: flex-end;
  }

  .export-button {
    background: #fff;
    color: #d6613b;
    border: 1px solid #d6613b;
    font-weight: 600;
    padding: 0.5rem 1rem;
    font-size: 0.875rem;
    cursor: pointer;
  }

  .export-button:hover:not(:disabled) {
    background-color: #fdf2ec;
  }

  .export-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  /* Utility Classes */
  .loading,
  .end-msg {
    text-align: center;
    padding: 1rem;
    color: #666;
    font-size: 0.875rem;
  }

  .sentinel {
    width: 100%;
    height: 20px;
    margin-top: 2rem;
  }

  /* Back to Top Button */
  .back-to-top {
    position: fixed;
    bottom: 2rem;
    right: 2rem;
    background: #fff;
    color: #d6613b;
    border: 1px solid #d6613b;
    padding: 0.5rem 1rem;
    font-size: 0.875rem;
    font-weight: 600;
    cursor: pointer;
    z-index: 100;
  }

  .back-to-top:hover {
    background: #fdf2ec;
  }

  /* Responsive Styles */
  @media (max-width: 768px) {
    .timeline-row {
      display: flex;
      flex-direction: column;
    }

    .month-label {
      flex-direction: row;
      justify-content: space-between;
      font-size: 1.25rem;
    }

    .year-label {
      font-size: 1rem;
    }

    .timeline-divider {
      display: none;
    }

    .search-container,
    .filter-container {
      flex-direction: column;
    }

    .search-wrapper,
    .export-container,
    .dropdown {
      width: 100%;
    }

    .back-to-top {
      bottom: 1rem;
      right: 1rem;
      font-size: 0.75rem;
    }
  }
  @media (max-width: 768px) {
  .article-title {
    font-size: 2rem;
    margin-bottom: 1.5rem;
  }
}

/* Update the search container and export button styles */
.search-container {
  font-family: 'helvetica neue', sans-serif;
  max-width: 900px;
  margin: 2rem auto;
  padding: 0 1rem;
  display: flex;
  gap: 1rem;
  align-items: stretch; /* Change from flex-start to stretch */
}

.search-wrapper {
  flex: 2;
  position: relative;
}

.export-container {
  flex: 1;
  display: flex;
}

.export-button {
  width: 100%;
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  font-weight: 600;
  color: #d6613b;
  background: white;
  border: 1px solid #d6613b;
  cursor: pointer;
  transition: all 0.2s ease;
  font-family: 'Helvetica Neue', sans-serif;
  height: 100%;
  box-sizing: border-box;
}

.export-button:hover:not(:disabled) {
  background-color: rgba(214, 97, 59, 0.1);
}

/* a  */

.article-intro a {
    color: #d6613b;
    text-decoration: none;
    font-weight: 600;
  }

  .article-intro a:hover {
    text-decoration: underline;
  }

/* Update the filters container */
.filters {
  position: sticky;
  top: 0;
  background: white;
  z-index: 1000;
  padding: 1rem 0;
  border-bottom: 1px solid rgba(67, 72, 90, 0.1);
  margin-bottom: 2rem;
}

/* Add shadow to filters when scrolling */
.filters::after {
  content: '';
  position: absolute;
  bottom: -8px;
  left: 0;
  right: 0;
  height: 8px;
  pointer-events: none;
}

  /* Add these styles */
  .header {
    margin-bottom: 2rem;
  }

  .title-container {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .title-row {
    display: flex;
    align-items: center;
    gap: 1rem;
  }

  .logo {
    flex-shrink: 0;
    width: 120px; 
  }

  .logo img {
    width: 100%;
    height: auto;
  }

  #title-style {
    font-family: 'EB Garamond', serif;
    font-size: 1.5rem;
    font-weight: 400;
    color: #d6613b;
  }

  .article-title {
    font-family: "Helvetica Neue", sans-serif;
    font-size: 3rem;
    line-height: 1.2;
    font-weight: 800;
    color: #d6613b;
    margin: 0;
  }

  /* Responsive adjustments */
  @media (max-width: 768px) {
    .title-row {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.5rem;
    }

    .logo {
      width: 50px;
    }

    #title-style {
      font-size: 1.25rem;
    }

    .article-title {
      font-size: 2rem;
    }
  }
</style>

<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Gantari:ital,wght@0,100..900;1,100..900&display=swap" rel="stylesheet">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=EB+Garamond:ital,wght@0,400..800;1,400..800&display=swap" rel="stylesheet">