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
    "AI/automation": "Integration of artificial intelligence and automation technologies within platforms’ or publishers’ operations; noteworthy advancements in the broader AI landscape; partnerships between AI companies and publishers; news related to AI-generated content",
    "algorithm": "Changes and updates made to the platform's algorithms, impacting how content is distributed, ranked, or presented",
    "alt-tech": "Updates related to social platforms popular among the alt-right, far-right and others who espouse extremism or fringe theories",
    "AR/VR": "Incorporation and advancement of Augmented Reality (AR) and Virtual Reality (VR) technologies within the platform's framework",
    "audience engagement": "User engagement metrics, features, or strategies",
    "bug/glitch/issue": "Technical glitches, bugs, or issues that impact the platform's functionality",
    "business": "Corporate activities such as acquisitions, rebrands, and strategic shifts, highlighting significant changes in the platform's business landscape",
    "content moderation": "Changes or controversies surrounding the platform's policies and practices for moderating content",
    "COVID-19": "Impact of the COVID-19 pandemic on the platform, including changes in user behavior, policies, or response strategies",
    "data/security": "Data breaches, security measures, or changes in data handling policies and practices",
    "elections/civic integrity": "Platform involvement in elections, including measures taken to prevent misinformation, political advertising policies, and responses to election-related events",
    "fediverse": "Updates related to decentralized social networking platforms that allow independent servers to communicate with each other through the ActivityPub protocol",
    "mis/disinformation": "Efforts and controversies surrounding the platform's attempts to combat or handle misinformation",
    "mission statement": "Updates or changes in the platform's mission and values",
    "partnership": "Partnerships within and between tech companies and news organizations (or lack thereof)",
    "philanthropy": "Charitable initiatives, donations, or philanthropic efforts undertaken by the platform",
    "platform as publisher": "Instances where platforms themselves engage in publishing content, blurring the lines between platform and publisher",
    "platform competition": "Competitor-related events, market dynamics, or strategic moves in response to competition",
    "platform personnel": "Changes in key personnel within the platform, such as CEOs, executives, or notable hires",
    "policy change": "Changes in terms of service, rules, or overall policies governing user behavior on the platform",
    "product": "Launches or updates to platform features, tools, or services",
    "regulation/legal dispute": "Legal challenges, regulatory scrutiny, or changes in legislation affecting the platform",
    "research/investigation": "Studies, investigations, or research initiatives undertaken by the platform or external parties related to its impact or practices",
    "revenue/funding": "Updates about platforms’ or publishers’ income or earnings"
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
      url.searchParams.append("pageSize", 20);

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

  $: uniqueCategories = Array.from(new Set(timelineData.flatMap(item => item.categories)));
  $: uniquePlatforms = Array.from(new Set(timelineData.flatMap(item =>
    Array.isArray(item.platform) ? item.platform : [item.platform]
  )));

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

    <h1 class="article-title">Tow Platforms and Publishers Timeline    </h1>

    <p class="article-intro">
      The timeline below identifies key developments in the relationship between technology platforms and news publishers. Though this timeline was originally created to monitor shifts and trends in the social media platform landscape, it has been expanded to also incorporate developments related to artificial intelligence technologies.
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
      You can learn more about how we define our categories <a href="#" class="category-link">here</a>.
    </p>

    <p class="article-intro">
      You can filter the timeline to search by specific platforms. Updates that pertain broadly to all social media platforms can be found under Social platforms; updates that pertain to all AI companies are filed under AI companies and Newsroom tools captures publishers’ in-house technology developments.
    </p>

    <p class="article-intro">
      This timeline is currently maintained by Klaudia Jaźwińska and will be updated at the beginning of every month. We welcome feedback or input on any missed developments.



    </p>
  
</div>

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
        Categories: {selectedCategories.join(", ")}
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
        Platforms: {selectedPlatforms.join(", ")}
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
/* Article styling */
.article {
  max-width: 800px;
  margin: 2rem auto;
  padding: 1.5rem;
  font-family: 'Gantari', sans-serif;
  line-height: 1.6;
}

.article-title {
  font-size: 2rem;
  font-weight: 700;
  color: #111827;
  margin-bottom: 1rem;
  text-align: center;
}

.byline {
  font-size: 1rem;
  font-weight: 400;
  color: #6b7280;
  margin-bottom: 4rem;
  text-align: center;
}

.article-intro {
  font-size: 1.125rem;
  font-weight: 500;
  color: #374151;
  margin-bottom: 1.5rem;
  text-align: justify;
}

.article-body {
  font-size: 1rem;
  font-weight: 400;
  color: #4b5563;
  text-align: justify;
}

/* Filter container styling */
.filter-container {
  width: 100%;
  max-width: 800px;
  padding: 1rem;
  display: flex;
  gap: 1rem;
  margin: 2rem auto 1rem; 
}

/* Update dropdown width */
.dropdown {
  position: relative;
  flex: 1; /* Make all dropdowns take equal space */
  min-width: 0; /* Allow dropdowns to shrink if needed */
}

/* Match date filter width with other dropdowns */
.date-filter {
  width: 220px; /* was 300px */
}

/* Add responsive styling for the filters */
@media (max-width: 768px) {
  .filter-container {
    flex-direction: column;
    align-items: stretch;
  }

  .dropdown,
  .date-filter {
    width: 100%;
    max-width: none;
  }
}

/* Dropdown styling */
.dropdown {
  position: relative;
  width: 100%;
}

.dropdown-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  font-weight: 600;
  color: #374151;
  /* border-radius: 0.5rem; */
  border: 1px solid #43485A;
  background-color: white;
  box-shadow: 4px 4px 0 0 #d6613b;
  cursor: pointer;
  transition: all 0.2s ease;
}

.dropdown-header:hover {
  background-color: #e5e7eb;
}

.dropdown-arrow {
  font-size: 0.75rem;
  color: #6b7280;
}

.dropdown-menu {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  margin-top: 0.5rem;
  /* border-radius: 0.5rem; */
  border: 1px solid #43485A;
  background-color: white;
  box-shadow: 3px 3px 0 0 #43485A;
  z-index: 10;
  max-height: 200px;
  overflow-y: auto;
}

.dropdown-item {
  display: flex;
  align-items: center;
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  font-weight: 500;
  color: #374151;
  cursor: pointer;
  transition: all 0.2s ease;
}

.dropdown-item:hover {
  background-color: #f3f4f6;
}

.dropdown-item input[type="checkbox"] {
  margin-right: 0.5rem;
  appearance: none; 
  width: 1rem;
  height: 1rem;
  min-width: 1rem;
  min-height: 1rem;
  border: 1px solid #43485A; 
  background-color: white; 
  cursor: pointer;
  transition: all 0.2s ease;
}

.dropdown-item input[type="checkbox"]:checked {
  background-color: #d6613b; 
  border-color: #43485A; 
}

.timeline-container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 4rem 2rem;
  width: 100%;
  position: relative;
}

/* Each row in the timeline */
.timeline-row {
  display: grid;
  grid-template-columns: 120px 2px 1fr;
  gap: 2rem;
  position: relative;
  align-items: flex-start;
  /* margin-bottom: 6rem; */
}

/* Sticky month label */
.month-label {
  position: sticky;
  top: 2rem;
  z-index: 2;
  width: 120px;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  justify-content: flex-start;
  font-size: 2rem;
  font-weight: 700;
  color: #43485A;
  text-transform: capitalize;
  padding-right: 0.5rem;
}

.year-label {
  font-size: 1.5rem;
  font-weight: 500;
  color: #43485A;
  margin-top: 0.5rem;
}

.timeline-divider {
  background: #43485A;
  width: 1px;
  height: 100%;
  /* opacity: 0.4; */
}

/* Timeline cards container */
.month-group {
  display: flex;
  flex-direction: column;
  
  align-items: center;
  width: 100%;
}

.month-group > * {
  width: 100%;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

:global(body) {
  background: #f0f0f0;
  font-family: 'Gantari', sans-serif;
  margin: 0;
  
}

@media (max-width: 768px) {
  /* Make timeline row stack vertically */
  .timeline-row {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
    margin-bottom: 4rem;
  }

  .month-label {
    position: relative;
    top: unset;
    flex-direction: row;
    justify-content: space-between;
    font-size: 1.25rem;
    width: 100%;
    background: transparent;
    padding-right: 0;
  }

  .year-label {
    font-size: 1rem;
    margin-top: 0;
  }

  .timeline-divider {
    display: none;
  }

  .month-group {
    width: 100%;
    padding-left: 0;
  }

  .card-wrapper {
    width: 100%;
  }
}

@media (max-width: 480px) {
  .dropdown-header {
    font-size: 0.75rem;
    padding: 0.4rem 0.75rem;
  }

  .dropdown-menu {
    font-size: 0.75rem;
  }

  .dropdown-item {
    padding: 0.4rem 0.75rem;
  }

  .month-label {
    font-size: 1rem;
  }

  .year-label {
    font-size: 0.875rem;
  }

  .timeline-container {
    padding: 2rem 1rem;
  }

  .filter-container {
    padding: 1rem;
  }
}

.loading,
.end-msg {
  text-align: center;
  padding: 1rem;
  color: #6b7280;
  font-size: 0.875rem;
}
  
.sentinel {
  width: 100%;
  height: 20px;
  margin-top: 2rem;
}

.back-to-top {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  background-color: white;
  color: #43485A;
  border: 1px solid #43485A;
  box-shadow: 4px 4px 0 0 #d6613b;
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  z-index: 100;
}

.back-to-top:hover {
  transform: translateY(-2px);
  box-shadow: 6px 6px 0 0 #d6613b;
}

@media (max-width: 768px) {
  .back-to-top {
    bottom: 1rem;
    right: 1rem;
    padding: 0.5rem 0.75rem;
    font-size: 0.75rem;
  }
}

.date-range-filter {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  width: 300px;
}

.date-range-filter label {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 1rem;
  background-color: white;
  border: 1px solid #43485A;
  box-shadow: 4px 4px 0 0 #d6613b;
  font-size: 0.875rem;
  font-weight: 600;
  color: #374151;
}

.date-range-filter input[type="date"] {
  border: none;
  font-size: 0.875rem;
  color: #374151;
  font-family: inherit;
  width: 150px;
  padding: 0;
  cursor: pointer;
}

.date-range-filter input[type="date"]::-webkit-calendar-picker-indicator {
  cursor: pointer;
  opacity: 0.6;
  transition: opacity 0.2s ease;
}

.date-range-filter input[type="date"]::-webkit-calendar-picker-indicator:hover {
  opacity: 1;
}

@media (max-width: 768px) {
  .date-range-filter {
    width: 100%;
    max-width: 300px;
  }
  
  .date-range-filter label {
    padding: 0.4rem 0.75rem;
  }
  
  .date-range-filter input[type="date"] {
    width: 120px;
  }
}

.date-filter {
  width: 300px;
}

.date-menu {
  padding: 1rem;
  min-width: 280px; /* Ensure enough width for the date inputs */
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
  font-size: 0.875rem;
  color: #374151;
}

.date-inputs input[type="date"] {
  padding: 0.5rem;
  border: 1px solid #43485A;
  font-size: 0.875rem;
  font-family: inherit;
  width: 100%; /* ensures full use of parent label */
  box-sizing: border-box;
}

/* Style the calendar picker icon */
.date-inputs input[type="date"]::-webkit-calendar-picker-indicator {
  padding: 0.2rem;
  margin: 0;
  cursor: pointer;
}

.clear-button {
  margin-top: 1rem;
  padding: 0.5rem;
  width: 100%;
  background-color: white;
  border: 1px solid #43485A;
  color: #374151;
  font-size: 0.875rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.clear-button:hover {
  background-color: #e5e7eb;
}

@media (max-width: 768px) {
  .date-menu {
    min-width: 260px;
  }
  
  .date-inputs input[type="date"] {
    min-width: 220px;
  }
}

/* Add to your existing styles */
.sort-filter {
  flex: 1;
  min-width: 0;
}

.sort-filter .dropdown-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.sort-filter .dropdown-arrow {
  font-size: 1rem;
  color: #43485A;
}

/* Add to your existing styles in App.svelte */

.category-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin: 1rem 0 3rem; /* <-- Added extra bottom margin */
  position: relative;
  z-index: 1;
}

.category-tag {
  font-size: 0.75rem;
  background-color: rgba(214, 97, 59, 0.2);
  color: #43485A;
  padding: 0.35rem 0.75rem;
  border: 1px solid #d6613b;
  font-weight: 500;
  cursor: help;
}

.category-tag:hover {
  transform: translateY(-1px);
  box-shadow: 3px 3px 0 0 #d6613b;
}

.category-link {
  color: #d6613b;
  text-decoration: none;
  font-weight: 600;
}

.category-link:hover {
  text-decoration: underline;
}

@media (max-width: 768px) {
  .category-tag {
    font-size: 0.75rem;
    padding: 0.2rem 0.5rem;
  }
}

.search-container {
  width: 100%;
  max-width: 800px;
  margin: 0 auto; /* Remove top margin */
  padding: 0 1rem;
  display: flex;
  gap: 2rem;
  align-items: center;
}

.search-wrapper {
  position: relative;
  flex: 2; 
}

.export-container {
  flex: 1; 
  margin: 0; 
  padding: 0; 
}

.export-button {
  width: 100%; 
}

@media (max-width: 768px) {
  .search-container {
    flex-direction: column;
  }

  .search-wrapper,
  .export-container {
    width: 100%;
  }

  .export-button {
    margin-top: 1rem;
  }
}

.search-input {
  width: 100%;
  padding: 0.5rem 1rem; 
  font-size: 0.875rem;
  font-weight: 500;
  color: #374151;
  border: 1px solid #43485A;
  background-color: white;
  box-shadow: 4px 4px 0 0 #d6613b;
  font-family: inherit;
  height: 36px; 
  box-sizing: border-box; 
}


.search-input:focus {
  outline: none;
  border-color: #43485A;
}

.clear-search {
  position: absolute;
  right: 1rem;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  font-size: 1rem;
  color: #6b7280;
  cursor: pointer;
}

.clear-search:hover {
  color: #374151;
}

@media (max-width: 768px) {
  .search-container {
    margin: 1rem auto;
  }
  
  .search-input {
    padding: 0.5rem 0.75rem;
    font-size: 0.875rem;
  }
}

/* Add to your existing styles */
.export-container {
  width: 100%;
  max-width: 800px;
  display: flex;
  justify-content: flex-end;
}

.export-button {
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  font-weight: 600;
  color: #43485A;
  background-color: white;
  border: 1px solid #43485A;
  box-shadow: 4px 4px 0 0 #d6613b;
  cursor: pointer;
  transition: all 0.2s ease;
}

.export-button:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 5px 5px 0 0 #d6613b;
}

.export-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  box-shadow: none;
}

@media (max-width: 768px) {
  .export-button {
    width: 100%;
    padding: 0.75rem;
  }
}
</style>

<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Gantari:ital,wght@0,100..900;1,100..900&display=swap" rel="stylesheet">