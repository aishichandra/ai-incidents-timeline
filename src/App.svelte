<script>
  import { fade } from 'svelte/transition';
  import TimelineCard from './components/TimelineCard.svelte';
  import { onMount } from 'svelte';

  let timelineData = [];
  let dropdownOpen = false;
  let selectedCategories = [];

  function handleClickOutside(event) {
    if (!event.target.closest('.dropdown')) {
      dropdownOpen = false;
    }
  }

  onMount(() => {
    document.addEventListener('click', handleClickOutside);
    return () => document.removeEventListener('click', handleClickOutside);
  });

  onMount(async () => {
    try {
      const res = await fetch(
        "https://api.airtable.com/v0/app8ahfwVPbUKPgEz/tbl2wAKmEwGXQ8ELY",
        {
          headers: {
            Authorization: "Bearer patUw9YjcnlCQoIZr.8ed788e6012e4870566558b401cc31a8304d0c360d7dcd2fefc69ad1e8f6e1af"
          }
        }
      );

      if (!res.ok) throw new Error(`Error fetching primary table: ${res.status}`);

      const harmRes = await fetch(
        "https://api.airtable.com/v0/app8ahfwVPbUKPgEz/tblAWw0vc5tCIhMJU",
        {
          headers: {
            Authorization: "Bearer patUw9YjcnlCQoIZr.8ed788e6012e4870566558b401cc31a8304d0c360d7dcd2fefc69ad1e8f6e1af"
          }
        }
      );

      if (!harmRes.ok) throw new Error(`Error fetching linked table: ${harmRes.status}`);
      
      const records = await res.json();
      const harmRecords = await harmRes.json();

      const harmMap = new Map(harmRecords.records.map(h => [h.id, h.fields.Name || "Unknown"]));

      timelineData = records.records.map(r => {
        const rawLink = r.fields.link;
        let links = [];

        if (Array.isArray(rawLink)) {
          links = rawLink.map(link => ({
            text: "View Source",
            url: typeof link === "string" ? (link.match(/\((.*?)\)/)?.[1] || link) : "#"
          }));
        } else if (typeof rawLink === "string") {
          links = [{
            text: "View Source",
            url: rawLink.match(/\((.*?)\)/)?.[1] || rawLink
          }];
        }

        return {
          id: r.id,
          date: r.fields.date || "Unknown date",
          platform: r.fields.platform || "Unknown platform",
          categories: r.fields.harm ? r.fields.harm.map(hid => harmMap.get(hid) || "Unknown") : [],
          title: r.fields.title || "(Untitled)",
          description: r.fields.desc || "(No description provided)",
          links
        };
      });

      timelineData = timelineData.map(item => ({
        ...item,
        links: item.links.map(link => ({
          text: link.text || "View Source",
          url: link.url || "#"
        }))
      }));
    } catch (error) {
      console.error("Failed to fetch Airtable data:", error);
    }
  });

  function groupByMonth(data) {
    const groups = {};
    const sorted = [...data].sort((a, b) => new Date(b.date) - new Date(a.date));
    sorted.forEach(item => {
      const month = new Date(item.date).toLocaleString('default', { month: 'long' });
      const year = new Date(item.date).getFullYear();
      const key = `${month} ${year}`;
      if (!groups[key]) groups[key] = [];
      groups[key].push(item);
    });
    return groups;
  }

  function filterByCategories(data, selectedCategories) {
    if (!selectedCategories.length) return data;
    return data.filter(item => item.categories.some(cat => selectedCategories.includes(cat)));
  }

  const groupedTimeline = groupByMonth(timelineData);
  $: uniqueCategories = Array.from(
  new Set(timelineData.flatMap(item => item.categories))
);
  let filteredData = filterByCategories(timelineData, selectedCategories);
  $: filteredData = filterByCategories(timelineData, selectedCategories);
</script>

<div class="article">
  <div class="article">
    <h1 class="article-title">Timeline of Platform and Publisher Developments</h1>
    <h3 class="byline">Database Maintained by <em>Klaudia Jaźwińska</em></h3>


    <p class="article-intro">
      The timeline below identifies key developments in the relationship between technology platforms and news publishers. Though this timeline was originally created to monitor shifts and trends in the social media platform landscape, it has been expanded to also incorporate developments related to artificial intelligence technologies.
    </p>
  
    <p class="article-body">
      This timeline contains updates related to the following categories, as they relate to platforms and publishers:
      This timeline is currently maintained by <em>Klaudia Jaźwińska</em> and will be updated at the beginning of every month. We welcome feedback or input on any missed developments.
    </p>
  </div>
  
</div>

<div class="filter-container">
  <div class="dropdown">
    <div class="dropdown-header" on:click={() => (dropdownOpen = !dropdownOpen)}>
      {#if selectedCategories.length > 0}
        {selectedCategories.join(", ")}
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
  max-width: 300px; 
  padding: 1rem;
  display: flex;
  margin-top: 2rem;
  margin-bottom: auto;
  margin-left: auto;
  margin-right: auto;
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
  box-shadow: 4px 4px 0 0 #F2B544;
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
  background-color: #F2B544; 
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
  background: #f0f0f0; /* matches body so it doesn’t clip under content */
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

</style>

<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Gantari:ital,wght@0,100..900;1,100..900&display=swap" rel="stylesheet">