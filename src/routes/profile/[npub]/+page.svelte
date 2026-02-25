<script lang="ts">
  import { onDestroy, onMount } from 'svelte';
  import { page } from '$app/stores';
  import { base } from '$app/paths';
  import { nip19 } from 'nostr-tools';
  import { ensureNdk } from '$lib/stores/ndk';
  import { fetchProfileFor, profileByPubkey } from '$lib/stores/profiles';
  import ListingCard from '$lib/components/ListingCard.svelte';
  import type { Listing } from '$lib/nostr/types';
  import { NOSTR_KINDS } from '$lib/nostr/constants';
  import { eventToListing } from '$lib/nostr/parse';
  import DMComposer from '$lib/components/DMComposer.svelte';
  import ZapComposer from '$lib/components/ZapComposer.svelte';

  let pubkey = '';
  let error: string | null = null;
  let listings: Listing[] = [];
  let stop: (() => void) | null = null;

  let dmOpen = false;
  let zapOpen = false;

  type ProfileTab = 'listings' | 'portfolio' | 'about';
  let activeTab: ProfileTab = 'listings';

  async function start() {
    if (!pubkey) return;
    const ndk = await ensureNdk();
    const sub = ndk.subscribe(
      {
        kinds: [NOSTR_KINDS.nip15_product, NOSTR_KINDS.nip99_classified],
        authors: [pubkey],
        limit: 100,
      },
      { closeOnEose: false },
    );
    sub.on('event', (ev) => {
      const l = eventToListing(ev as any);
      if (!l) return;
      listings = [l, ...listings.filter((x) => x.eventId !== l.eventId)].sort((a, b) => b.createdAt - a.createdAt);
    });
    stop = () => sub.stop();
  }

  onMount(() => {
    error = null;
    listings = [];
    const npub = String($page.params.npub || '');
    try {
      const decoded = nip19.decode(npub);
      if (decoded.type !== 'npub') throw new Error('Not an npub');
      pubkey = decoded.data as string;
      void fetchProfileFor(pubkey);
      void start();
    } catch (e) {
      error = e instanceof Error ? e.message : String(e);
    }
  });

  onDestroy(() => {
    if (stop) stop();
  });

  $: prof = pubkey ? $profileByPubkey[pubkey] : undefined;
  $: name = prof?.display_name || prof?.name || 'Artist';
  $: about = (prof?.about || '').trim();
  $: skills = (prof as any)?.skills as string[] | undefined;
  $: hashtags = (prof as any)?.hashtags as string[] | undefined;
  $: portfolio = (prof as any)?.portfolio as string[] | undefined;
  $: nip05 = (prof as any)?.nip05 as string | undefined;
  $: lud16 = (prof as any)?.lud16 as string | undefined;
</script>

{#if error}
  <div class="card" style="padding: 1rem; border-color: rgba(251,113,133,0.35);">
    <div class="muted">{error}</div>
  </div>
{:else}
  <div class="profile-header card">
    <div class="profile-cover"></div>
    <div class="profile-info">
      <div class="avatar-section">
        {#if prof?.picture}
          <img src={prof.picture} alt="" class="profile-avatar" />
        {:else}
          <div class="profile-avatar profile-avatar-placeholder">🎨</div>
        {/if}
      </div>
      <div class="profile-details">
        <h1 class="profile-name">{name}</h1>
        {#if nip05}
          <div class="profile-nip05 muted">{nip05}</div>
        {/if}
        <div class="profile-npub muted mono">{$page.params.npub}</div>

        <div class="profile-actions">
          <button class="btn" on:click={() => (dmOpen = true)}>Message</button>
          <button class="btn primary" on:click={() => (zapOpen = true)}>Zap</button>
          {#if prof?.website}
            <a class="btn" href={prof.website} target="_blank" rel="noreferrer">Website</a>
          {/if}
        </div>

        <div class="profile-meta">
          {#if lud16}
            <span class="pill muted" title="Lightning address">⚡ {lud16}</span>
          {/if}
          <span class="pill muted">{listings.length} listing{listings.length !== 1 ? 's' : ''}</span>
        </div>
      </div>
    </div>
  </div>

  <div class="profile-tabs">
    <button class="tab-btn" class:active={activeTab === 'listings'} on:click={() => (activeTab = 'listings')}>
      Listings ({listings.length})
    </button>
    <button class="tab-btn" class:active={activeTab === 'about'} on:click={() => (activeTab = 'about')}>
      About
    </button>
    {#if portfolio?.length}
      <button class="tab-btn" class:active={activeTab === 'portfolio'} on:click={() => (activeTab = 'portfolio')}>
        Portfolio ({portfolio.length})
      </button>
    {/if}
  </div>

  {#if activeTab === 'listings'}
    <div class="listings-grid">
      {#each listings as l (l.eventId)}
        <ListingCard listing={l} />
      {/each}
      {#if listings.length === 0}
        <div class="card" style="padding: 1.5rem; grid-column: 1 / -1; text-align: center;">
          <div class="muted">No listings found on the connected relays yet.</div>
        </div>
      {/if}
    </div>

  {:else if activeTab === 'about'}
    <div class="about-section">
      {#if about}
        <div class="card about-card">
          <div style="font-weight: 850; margin-bottom: 0.5rem;">Bio</div>
          <div class="muted about-text">{about}</div>
        </div>
      {/if}

      {#if skills?.length}
        <div class="card about-card">
          <div style="font-weight: 850; margin-bottom: 0.5rem;">Skills</div>
          <div class="tag-cloud">
            {#each skills.slice(0, 20) as s}
              <span class="pill skill-pill">{s}</span>
            {/each}
          </div>
        </div>
      {/if}

      {#if hashtags?.length}
        <div class="card about-card">
          <div style="font-weight: 850; margin-bottom: 0.5rem;">Hashtags</div>
          <div class="tag-cloud">
            {#each hashtags.slice(0, 20) as t}
              <span class="pill muted">#{t}</span>
            {/each}
          </div>
        </div>
      {/if}

      {#if !about && !skills?.length && !hashtags?.length}
        <div class="card" style="padding: 1.5rem; text-align: center;">
          <div class="muted">No bio or skills information available.</div>
        </div>
      {/if}
    </div>

  {:else if activeTab === 'portfolio'}
    <div class="portfolio-grid">
      {#each (portfolio ?? []).slice(0, 20) as url}
        <a href={url} target="_blank" rel="noreferrer" class="card portfolio-item">
          {#if url.match(/\.(jpg|jpeg|png|gif|webp|svg)$/i)}
            <img src={url} alt="" loading="lazy" class="portfolio-img" />
          {:else}
            <div class="portfolio-link">
              <div class="muted" style="font-size: 1.5rem;">🔗</div>
              <div class="muted portfolio-url">{url}</div>
            </div>
          {/if}
        </a>
      {/each}
    </div>
  {/if}

  <DMComposer open={dmOpen} toPubkey={pubkey} toLabel={name} onClose={() => (dmOpen = false)} />
  <ZapComposer
    open={zapOpen}
    recipientPubkey={pubkey}
    recipientLabel={name}
    onClose={() => (zapOpen = false)}
  />
{/if}

<style>
  .profile-header {
    overflow: hidden;
  }
  .profile-cover {
    height: 120px;
    background: linear-gradient(135deg, rgba(247, 147, 26, 0.15), rgba(139, 92, 246, 0.15), rgba(56, 189, 248, 0.1));
  }
  .profile-info {
    padding: 0 1.2rem 1.2rem;
    display: flex;
    gap: 1.2rem;
    align-items: flex-start;
    flex-wrap: wrap;
  }
  .avatar-section {
    margin-top: -40px;
  }
  .profile-avatar {
    width: 88px;
    height: 88px;
    border-radius: 24px;
    border: 3px solid var(--bg);
    object-fit: cover;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 2.5rem;
  }
  .profile-avatar-placeholder {
    background: linear-gradient(135deg, rgba(246, 196, 83, 0.15), rgba(139, 92, 246, 0.1));
  }
  .profile-details {
    flex: 1;
    min-width: 0;
    padding-top: 0.3rem;
  }
  .profile-name {
    font-size: 1.6rem;
    font-weight: 950;
    margin: 0;
    line-height: 1.1;
  }
  .profile-nip05 {
    margin-top: 0.2rem;
    font-size: 0.9rem;
    color: var(--accent);
  }
  .profile-npub {
    margin-top: 0.25rem;
    font-size: 0.78rem;
    word-break: break-all;
  }
  .mono {
    font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono',
      'Courier New', monospace;
  }
  .profile-actions {
    margin-top: 0.85rem;
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
  }
  .profile-meta {
    margin-top: 0.65rem;
    display: flex;
    gap: 0.4rem;
    flex-wrap: wrap;
  }
  .profile-tabs {
    margin-top: 1rem;
    display: flex;
    gap: 0.35rem;
    border-bottom: 1px solid var(--border);
    padding-bottom: 0.5rem;
  }
  .tab-btn {
    padding: 0.5rem 0.8rem;
    border-radius: 10px 10px 0 0;
    border: 1px solid transparent;
    border-bottom: none;
    background: transparent;
    color: var(--muted);
    font-weight: 650;
    font-size: 0.9rem;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
  }
  .tab-btn:hover {
    background: rgba(255, 255, 255, 0.06);
    color: var(--text);
  }
  .tab-btn.active {
    background: rgba(246, 196, 83, 0.1);
    border-color: rgba(246, 196, 83, 0.25);
    color: var(--accent);
  }
  .listings-grid {
    margin-top: 1rem;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 1rem;
  }
  .about-section {
    margin-top: 1rem;
    display: grid;
    gap: 0.85rem;
  }
  .about-card {
    padding: 1rem 1.1rem;
  }
  .about-text {
    line-height: 1.65;
    white-space: pre-wrap;
  }
  .tag-cloud {
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
  }
  .skill-pill {
    background: rgba(246, 196, 83, 0.08);
    border-color: rgba(246, 196, 83, 0.2);
    color: var(--accent);
  }
  .portfolio-grid {
    margin-top: 1rem;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 0.75rem;
  }
  .portfolio-item {
    overflow: hidden;
    transition: transform 0.2s, border-color 0.2s;
  }
  .portfolio-item:hover {
    text-decoration: none;
    transform: translateY(-2px);
    border-color: rgba(246, 196, 83, 0.3);
  }
  .portfolio-img {
    width: 100%;
    height: 180px;
    object-fit: cover;
    display: block;
  }
  .portfolio-link {
    padding: 1.5rem 1rem;
    text-align: center;
  }
  .portfolio-url {
    margin-top: 0.5rem;
    font-size: 0.82rem;
    word-break: break-all;
  }
  @media (max-width: 600px) {
    .profile-name {
      font-size: 1.3rem;
    }
    .listings-grid {
      grid-template-columns: 1fr;
    }
    .portfolio-grid {
      grid-template-columns: repeat(2, 1fr);
    }
  }
</style>
