<script>
  export let date;
  export let title;
  export let tags = [];
  export let description;
  export let platform;
  export let links = [];

  // Convert platform string to array if needed
  $: platformArray = Array.isArray(platform) ? platform : [platform];
</script>

<div class="card">
  <div class="card-date">{date}</div>
  
  <!-- Right Side (now on left) -->
  <div class="card-content">
    <div class="card-description-section">
      <p class="card-description">{description}</p>
      {#if links.length}
        <div class="card-links">
          <div class="related-label">Read More</div>
          {#if links.length > 0}
            <ul>
              {#each links as link}
                <li>
                  <a class="related-link" href={link.url} target="_blank" rel="noopener noreferrer" title={link.url}>
                    {link.text}
                  </a>
                </li>
              {/each}
            </ul>
          {/if}
        </div>
      {/if}
    </div>

    <!-- Left Side (now on right) -->
    <div class="card-meta">
      <div class="meta-row">
        <div class="meta-block">
          <!-- Platform Section -->
          <div class="platform-section">
            <span class="section-label">Platform</span>
            <div class="platform-list">
              {#each platformArray as p}
                <span class="platform-tag">{p}</span>
              {/each}
            </div>
          </div>
          
          <!-- Categories Section -->
          <div class="categories-section">
            <span class="section-label">Categories</span>
            <div class="tag-list">
              {#each tags as tag}
                <span class="tag">{tag}</span>
              {/each}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
.card {
  width: 100%;
  padding: 2rem;
  border: 1px solid #d6613b;
  background-color: white;
  box-shadow: 5px 5px 0 0 #d6613b;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin-bottom: 2rem;
  box-sizing: border-box;
}

.card-date {
  font-family: 'Helvetica Neue', sans-serif;
  font-size: 0.75rem;
  font-weight: 600;
  color: #43485A;
  text-transform: uppercase;
  letter-spacing: 0.03em;
}

.card-content {
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

@media (min-width: 768px) {
  .card-content {
    flex-direction: row;
  }

  .card-description-section {
    flex: 2;
    padding-right: 2rem;
  }

  .card-meta {
    flex: 1;
    max-width: 300px;
    border-left: 1px solid #e5e7eb;
    padding-left: 2rem;
  }
}

.card-left {
  flex: 1.2;
  max-width: 50%;
  display: flex;
  flex-direction: column;
  justify-content: flex-start; 
}

.card-meta {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.card-title {
  font-size: 1.5rem;
  font-weight: 700;
  color: #111827;
  margin: 0;
}

.meta-block {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.meta-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #41434E;
}

.meta-value {
  font-size: 0.875rem;
  color: #111827;
}

.tag-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 0.25rem;
}

.tag {
  font-family: 'Helvetica Neue', sans-serif;
  font-size: 0.75rem;
  background-color: rgb(214, 97, 59, 0.2);
  color: #43485A;
  padding: 0.35rem 0.75rem;
  border: 1px solid #d6613b;
  /* font-weight: 500; */
}

.card-right {
  flex: 2;
  margin-top: 1rem;
  font-size: 0.9375rem;
  line-height: 1.5;
  color: #41434E;
}

.card-description {
  font-size: 1.2rem;
  line-height: 1.7;
  margin-bottom: 1.5rem;
}

.card-links {
  /* border-top: 1px solid #D9626B; */
  padding-top: 1rem;
  
}

.related-label {
  font-size: 0.75rem;
  font-family: 'Helvetica Neue', sans-serif;
  font-weight: 400;
  color: #6b7280;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.related-link {
  display: block;
  /* font-family: 'Helvetica Neue', sans-serif; */
  font-size: 1.1rem;
  color: #D9626B;
  text-decoration: underline;
  margin-bottom: 0.5rem;
  transition: color 0.2s ease;
}

.related-link:hover {
  color: #43485A;
}

.link-host {
  color: #6b7280;
  font-size: 0.75rem;
}

.meta-row {
  display: flex;
  gap: 2rem;
  margin-top: 0.5rem;
  margin-bottom: 1rem;
}

.meta-block {
  flex: 1;
  min-width: 150px;
}

.tag-platform {
  background-color: rgba(67, 72, 90, 0.1);
  border: 1px solid #43485A;
  color: #43485A;
}

.platform-section,
.categories-section {
  margin-bottom: 1rem;
}

.section-label {
  display: block;
  font-family: 'Helvetica Neue', sans-serif;
  font-size: 0.7rem;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  margin-bottom: 0.5rem;
}

.platform-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.platform-tag {
  font-family: 'Helvetica Neue', sans-serif;
  font-size: 0.75rem;
  background-color: rgba(67, 72, 90, 0.1);
  color: #43485A;
  padding: 0.35rem 0.75rem;
  border: 1px solid #43485A;
  /* font-weight: 500; */
}

</style>
 