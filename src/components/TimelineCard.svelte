<script>
  export let date;
  export let title;
  export let platform;
  export let tags = [];
  export let description;
  export let links = [];
</script>

<div class="card">
  <!-- Left Side -->
  <div class="card-left">
    <div class="card-meta">
      <div class="card-date">{date}</div>
      <h2 class="card-title">{title}</h2>

      <div class="meta-row">

        {#if tags.length}
          <div class="meta-block">
            <!-- <span class="meta-label">Type of Harm:</span> -->
            <div class="tag-list">
              {#each tags as tag}
                <span class="tag">{tag}</span>
              {/each}
            </div>
          </div>
        {/if}
      </div>

    </div>
  </div>
  
  <!-- Right Side -->
  <div class="card-right">
    <p class="card-description">{description}</p>
    {#if links.length}
    <div class="card-links">
      <div class="related-label">Read More</div>
      {#each links as l}
        <a href={l.url} class="related-link" target="_blank" rel="noopener">
          {l.text}
          <span class="link-host">({new URL(l.url).hostname.replace('www.', '')})</span>
        </a>
      {/each}
    </div>
  {/if}
  </div>
</div>

<style>
.card {
  width: 100%;
  padding: 2rem;
  /* border-radius: 1rem; */
  border: 1px solid #43485A;
  background-color: white;
  box-shadow: 5px 5px 0 0 #F2B544;
  display: flex;
  flex-direction: column;
  gap: 2rem;
  align-items: flex-start; 
  margin-bottom: 2rem;
  box-sizing: border-box;
}

@media (min-width: 768px) {
  .card {
    flex-direction: row;
    align-items: flex-start; 
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

.card-date {
  font-size: 0.75rem;
  font-weight: 600;
  color: #43485A;
  text-transform: uppercase;
  letter-spacing: 0.03em;
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
  font-size: 0.75rem;
  background-color: rgba(189, 211, 222, 0.4);
  color: #43485A;
  padding: 0.35rem 0.75rem;
  border-color: #F2B544;
  font-weight: 500;
}

.card-right {
  flex: 2;
  margin-top: 1rem;
  /* display: flex;
  flex-direction: column;
  justify-content: flex-start; Align content to the top */
  font-size: 0.9375rem;
  line-height: 1.5;
  color: #41434E;
}

.card-description {
  line-height: 1.7;
  margin-bottom: 1.5rem;
}

.card-links {
  border-top: 1px solid #D9626B;
  padding-top: 1rem;
}

.related-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #6b7280;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.related-link {
  display: block;
  font-size: 0.875rem;
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
  /* border-top: 1px solid #F2B544;
   */
  gap: 2rem;
  margin-top: 0.5rem;
  margin-bottom: 1rem;
  /* width: 70%; */
}
.meta-block {
  flex: 1;
  min-width: 150px;
}
.tag-platform {
  background-color: #43485A;
  color: #43485A;
}

</style>
