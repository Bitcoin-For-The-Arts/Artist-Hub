<script lang="ts">
  import { base } from '$app/paths';
  import type { Listing } from '$lib/nostr/types';
  import { profileByPubkey } from '$lib/stores/profiles';
  import { npubFor } from '$lib/nostr/helpers';

  export let listing: Listing;

  $: author = $profileByPubkey[listing.pubkey];
  $: authorName = author?.display_name || author?.name || 'Artist';
  $: img = listing.images?.[0];
  $: price = listing.priceSats ? `${listing.priceSats.toLocaleString()} sats` : undefined;
  $: isGig = listing.tags?.some((t) => ['gig', 'commission', 'forhire', 'freelance'].includes(t.toLowerCase()));
  $: isCollab = listing.tags?.some((t) => ['collab', 'collaboration', 'seeking'].includes(t.toLowerCase()));
  $: kindLabel = isGig ? 'Gig' : isCollab ? 'Collab' : listing.kind === 'nip15_product' ? 'Service' : 'Classified';
</script>

<a class="listing-card card" href={`${base}/listing/${listing.eventId}`}>
  <div class="thumb">
    {#if img}
      <img src={img} alt="" loading="lazy" />
    {:else}
      <div class="placeholder">
        <span class="placeholder-icon">🎨</span>
      </div>
    {/if}
    <div class="kind-badge">{kindLabel}</div>
    {#if price}
      <div class="price-badge">{price}</div>
    {/if}
  </div>

  <div class="body">
    <div class="title">{listing.title}</div>

    {#if listing.summary}
      <div class="muted summary">{listing.summary}</div>
    {/if}

    <div class="meta">
      <div class="author">
        {#if author?.picture}
          <img src={author.picture} alt="" class="avatar" loading="lazy" />
        {:else}
          <div class="avatar avatar-placeholder"></div>
        {/if}
        <div class="who">
          <div class="name">{authorName}</div>
          <div class="muted npub">{npubFor(listing.pubkey).slice(0, 14)}...</div>
        </div>
      </div>
    </div>

    {#if listing.tags?.length}
      <div class="tags">
        {#each listing.tags.slice(0, 4) as t}
          <span class="pill muted tag">#{t}</span>
        {/each}
        {#if listing.tags.length > 4}
          <span class="pill muted tag">+{listing.tags.length - 4}</span>
        {/if}
      </div>
    {/if}
  </div>
</a>

<style>
  .listing-card {
    display: block;
    overflow: hidden;
    transition: transform 0.2s, border-color 0.2s, box-shadow 0.2s;
  }
  .listing-card:hover {
    text-decoration: none;
    transform: translateY(-3px);
    border-color: rgba(246, 196, 83, 0.3);
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.25);
  }
  .thumb {
    position: relative;
    height: 180px;
    background: rgba(255, 255, 255, 0.03);
    border-bottom: 1px solid var(--border);
    overflow: hidden;
  }
  .thumb img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
    transition: transform 0.3s;
  }
  .listing-card:hover .thumb img {
    transform: scale(1.05);
  }
  .placeholder {
    width: 100%;
    height: 100%;
    display: grid;
    place-items: center;
    background: linear-gradient(135deg, rgba(246, 196, 83, 0.05), rgba(139, 92, 246, 0.05));
  }
  .placeholder-icon {
    font-size: 2.5rem;
    opacity: 0.5;
  }
  .kind-badge {
    position: absolute;
    top: 0.5rem;
    left: 0.5rem;
    padding: 0.2rem 0.55rem;
    border-radius: 8px;
    font-size: 0.75rem;
    font-weight: 700;
    background: rgba(11, 11, 15, 0.75);
    backdrop-filter: blur(8px);
    color: var(--text);
    border: 1px solid rgba(255, 255, 255, 0.1);
  }
  .price-badge {
    position: absolute;
    bottom: 0.5rem;
    right: 0.5rem;
    padding: 0.25rem 0.6rem;
    border-radius: 8px;
    font-size: 0.82rem;
    font-weight: 750;
    background: rgba(246, 196, 83, 0.9);
    color: #0b0b0f;
  }
  .body {
    padding: 0.85rem 0.9rem 0.95rem;
  }
  .title {
    font-weight: 800;
    font-size: 0.98rem;
    line-height: 1.25;
  }
  .summary {
    margin-top: 0.4rem;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
    line-height: 1.35;
    font-size: 0.88rem;
  }
  .meta {
    margin-top: 0.75rem;
    display: flex;
    gap: 0.75rem;
    align-items: center;
  }
  .author {
    display: flex;
    gap: 0.55rem;
    align-items: center;
    min-width: 0;
  }
  .avatar {
    width: 30px;
    height: 30px;
    border-radius: 10px;
    border: 1px solid var(--border);
    object-fit: cover;
    flex-shrink: 0;
  }
  .avatar-placeholder {
    background: rgba(255, 255, 255, 0.06);
  }
  .who {
    display: grid;
    gap: 0.05rem;
    min-width: 0;
  }
  .name {
    font-weight: 700;
    font-size: 0.88rem;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 180px;
  }
  .npub {
    font-size: 0.76rem;
    font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono',
      'Courier New', monospace;
  }
  .tags {
    margin-top: 0.65rem;
    display: flex;
    flex-wrap: wrap;
    gap: 0.25rem;
  }
  .tag {
    font-size: 0.74rem;
    padding: 0.15rem 0.4rem;
  }
</style>
