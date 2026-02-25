<script lang="ts">
  import { onMount } from 'svelte';
  import ListingCard from '$lib/components/ListingCard.svelte';
  import {
    discoveryCategory,
    discoveryLoading,
    discoveryQuery,
    discoveryTags,
    filteredListings,
    setQuickTag,
    startDiscovery,
  } from '$lib/stores/discovery';
  import { groupedTags } from '$lib/nostr/tags';

  type ViewTab = 'all' | 'services' | 'gigs' | 'collabs';

  let activeTab: ViewTab = 'all';
  let tagInput = '';
  let showFilters = true;

  const quickCategories = ['Visual Arts', 'Music', 'Film', 'Writing', 'Design', 'Performance', 'Workshops', 'Collaboration'];

  onMount(() => {
    void startDiscovery();
  });

  function addTag() {
    const t = tagInput.trim().replace(/^#/, '');
    if (!t) return;
    discoveryTags.update((prev) => {
      const has = prev.some((x) => x.toLowerCase() === t.toLowerCase());
      return has ? prev : [t, ...prev].slice(0, 8);
    });
    tagInput = '';
  }

  $: tabListings = $filteredListings.filter((l) => {
    if (activeTab === 'all') return true;
    if (activeTab === 'services') return l.kind === 'nip15_product';
    if (activeTab === 'gigs') return l.tags?.some((t) => ['gig', 'commission', 'forhire', 'freelance', 'customwork'].includes(t.toLowerCase()));
    if (activeTab === 'collabs') return l.tags?.some((t) => ['collab', 'collaboration', 'seeking', 'communityproject'].includes(t.toLowerCase()));
    return true;
  });
</script>

<div class="discover-hero card">
  <div class="hero-inner">
    <h1 class="hero-title">Discover</h1>
    <p class="muted hero-desc">
      Browse the decentralized artist marketplace. Services, gigs, commissions, and collaboration requests — all signed Nostr events on public relays.
    </p>
    <div class="search-bar">
      <input class="input search-input" bind:value={$discoveryQuery} placeholder="Search: illustration, mixing, collaboration, poster design..." />
    </div>
  </div>
</div>

<div class="tabs-bar">
  <button class="tab-btn" class:active={activeTab === 'all'} on:click={() => (activeTab = 'all')}>
    All Listings
  </button>
  <button class="tab-btn" class:active={activeTab === 'services'} on:click={() => (activeTab = 'services')}>
    Services
  </button>
  <button class="tab-btn" class:active={activeTab === 'gigs'} on:click={() => (activeTab = 'gigs')}>
    Gigs & Commissions
  </button>
  <button class="tab-btn" class:active={activeTab === 'collabs'} on:click={() => (activeTab = 'collabs')}>
    Collaborations
  </button>
  <div style="flex:1;"></div>
  <button class="btn filter-toggle" on:click={() => (showFilters = !showFilters)}>
    {showFilters ? 'Hide Filters' : 'Show Filters'}
  </button>
</div>

{#if showFilters}
  <div class="filters-panel card">
    <div class="filter-row">
      <div class="filter-group">
        <div class="filter-label muted">Category</div>
        <select class="select" bind:value={$discoveryCategory}>
          <option value="">All categories</option>
          {#each quickCategories as c}
            <option value={c}>{c}</option>
          {/each}
        </select>
      </div>

      <div class="filter-group" style="flex: 1;">
        <div class="filter-label muted">Add tag</div>
        <div style="display:flex; gap:0.5rem; align-items:center;">
          <input
            class="input"
            bind:value={tagInput}
            placeholder="#BitcoinArt, #VisualArt..."
            on:keydown={(e) => e.key === 'Enter' && addTag()}
          />
          <button class="btn" on:click={addTag}>Add</button>
        </div>
      </div>
    </div>

    {#if $discoveryTags.length}
      <div class="active-tags">
        {#each $discoveryTags as t}
          <button
            class="pill tag-active"
            on:click={() =>
              discoveryTags.update((prev) => prev.filter((x) => x.toLowerCase() !== t.toLowerCase()))}
            title="Remove tag"
          >
            #{t} ✕
          </button>
        {/each}
        <button class="pill" on:click={() => discoveryTags.set([])}>Clear all</button>
      </div>
    {/if}

    <div class="tag-groups">
      {#each Object.entries(groupedTags) as [group, tags]}
        <div class="tag-group">
          <div class="tag-group-label muted">{group}</div>
          <div class="tag-group-chips">
            {#each tags.slice(0, 6) as t}
              <button class="pill muted tag-chip" on:click={() => setQuickTag(t)}>#{t}</button>
            {/each}
          </div>
        </div>
      {/each}
    </div>
  </div>
{/if}

<div class="results-header">
  <div class="results-count">
    <span style="font-weight: 850;">{tabListings.length}</span>
    <span class="muted">listing{tabListings.length !== 1 ? 's' : ''}</span>
  </div>
  {#if $discoveryLoading}
    <div class="loading-indicator">
      <span class="loading-dot"></span>
      Loading from relays...
    </div>
  {/if}
</div>

<div class="results-grid">
  {#each tabListings as l (l.eventId)}
    <ListingCard listing={l} />
  {/each}
  {#if !$discoveryLoading && tabListings.length === 0}
    <div class="card empty-state" style="grid-column: 1 / -1;">
      <div class="muted">No listings found matching your filters. Try broadening your search or check back later.</div>
    </div>
  {/if}
</div>

<style>
  .discover-hero {
    padding: 1.5rem 1.3rem;
    text-align: center;
    border-color: rgba(246, 196, 83, 0.15);
    background: linear-gradient(180deg, rgba(246, 196, 83, 0.08), rgba(255, 255, 255, 0.04));
  }
  .hero-inner {
    max-width: 600px;
    margin: 0 auto;
  }
  .hero-title {
    font-size: 1.7rem;
    font-weight: 950;
    margin: 0;
  }
  .hero-desc {
    margin: 0.5rem 0 0;
    line-height: 1.55;
  }
  .search-bar {
    margin-top: 1rem;
  }
  .search-input {
    font-size: 1rem;
    padding: 0.75rem 1rem;
    border-radius: 14px;
  }
  .tabs-bar {
    display: flex;
    align-items: center;
    gap: 0.35rem;
    margin-top: 1rem;
    overflow-x: auto;
    padding-bottom: 0.15rem;
  }
  .tab-btn {
    padding: 0.55rem 0.85rem;
    border-radius: 10px;
    border: 1px solid transparent;
    background: transparent;
    color: var(--muted);
    font-weight: 650;
    font-size: 0.9rem;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.15s, color 0.15s, border-color 0.15s;
  }
  .tab-btn:hover {
    background: rgba(255, 255, 255, 0.06);
    color: var(--text);
  }
  .tab-btn.active {
    background: rgba(246, 196, 83, 0.1);
    border-color: rgba(246, 196, 83, 0.3);
    color: var(--accent);
  }
  .filter-toggle {
    font-size: 0.85rem;
    padding: 0.45rem 0.7rem;
  }
  .filters-panel {
    margin-top: 0.75rem;
    padding: 1rem;
  }
  .filter-row {
    display: flex;
    gap: 1rem;
    align-items: flex-end;
    flex-wrap: wrap;
  }
  .filter-group {
    min-width: 180px;
  }
  .filter-label {
    font-size: 0.85rem;
    margin-bottom: 0.35rem;
  }
  .active-tags {
    margin-top: 0.75rem;
    display: flex;
    gap: 0.35rem;
    flex-wrap: wrap;
  }
  .tag-active {
    background: rgba(246, 196, 83, 0.12);
    border-color: rgba(246, 196, 83, 0.3);
    color: var(--accent);
    cursor: pointer;
  }
  .tag-groups {
    margin-top: 0.9rem;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 0.6rem;
  }
  .tag-group {
    padding: 0.6rem 0.7rem;
    border-radius: 12px;
    background: rgba(0, 0, 0, 0.15);
  }
  .tag-group-label {
    font-size: 0.8rem;
    font-weight: 700;
    margin-bottom: 0.4rem;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
  .tag-group-chips {
    display: flex;
    flex-wrap: wrap;
    gap: 0.25rem;
  }
  .tag-chip {
    cursor: pointer;
    font-size: 0.78rem;
    padding: 0.2rem 0.45rem;
    transition: background 0.15s, color 0.15s;
  }
  .tag-chip:hover {
    background: rgba(246, 196, 83, 0.1);
    color: var(--accent);
  }
  .results-header {
    margin-top: 1rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 1rem;
  }
  .results-count {
    font-size: 0.95rem;
    display: flex;
    gap: 0.35rem;
  }
  .loading-indicator {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    color: var(--muted);
    font-size: 0.88rem;
  }
  .loading-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--accent);
    animation: pulse 1.2s ease-in-out infinite;
  }
  @keyframes pulse {
    0%, 100% { opacity: 0.4; transform: scale(0.85); }
    50% { opacity: 1; transform: scale(1.15); }
  }
  .results-grid {
    margin-top: 0.85rem;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 1rem;
  }
  .empty-state {
    padding: 2rem 1.5rem;
    text-align: center;
  }
  @media (max-width: 600px) {
    .hero-title {
      font-size: 1.35rem;
    }
    .filter-row {
      flex-direction: column;
    }
    .filter-group {
      width: 100%;
    }
    .results-grid {
      grid-template-columns: 1fr;
    }
  }
</style>
