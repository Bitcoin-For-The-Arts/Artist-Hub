<script lang="ts">
  import { onDestroy, onMount } from 'svelte';
  import { base } from '$app/paths';
  import { ensureNdk } from '$lib/stores/ndk';
  import { NOSTR_KINDS } from '$lib/nostr/constants';
  import { isAuthed, pubkey } from '$lib/stores/auth';
  import { fetchProfileFor, profileByPubkey } from '$lib/stores/profiles';
  import { npubFor } from '$lib/nostr/helpers';
  import { signWithNip07, publishSignedEvent } from '$lib/nostr/pool';
  import ZapComposer from '$lib/components/ZapComposer.svelte';

  type LiveEvent = {
    id: string;
    pubkey: string;
    title: string;
    summary: string;
    streaming: string;
    image: string;
    status: 'live' | 'planned' | 'ended';
    starts: number;
    tags: string[];
    dTag: string;
    address: string;
  };

  type ChatMessage = {
    id: string;
    pubkey: string;
    content: string;
    createdAt: number;
  };

  let liveEvents: LiveEvent[] = [];
  let loading = true;
  let stop: (() => void) | null = null;

  let selectedEvent: LiveEvent | null = null;
  let chatMessages: ChatMessage[] = [];
  let chatText = '';
  let chatBusy = false;
  let chatStop: (() => void) | null = null;

  let showGoLive = false;
  let goLiveTitle = '';
  let goLiveSummary = '';
  let goLiveStreamUrl = '';
  let goLiveImage = '';
  let goLiveTags = '';
  let goLiveBusy = false;
  let goLiveError: string | null = null;
  let goLiveOk: string | null = null;

  let zapOpen = false;
  let zapTarget = '';
  let zapLabel = '';

  function parseEvent(ev: any): LiveEvent | null {
    const tags = ev.tags as string[][];
    const title = tags.find((t: string[]) => t[0] === 'title')?.[1] || 'Untitled Stream';
    const summary = tags.find((t: string[]) => t[0] === 'summary')?.[1] || '';
    const streaming = tags.find((t: string[]) => t[0] === 'streaming')?.[1] || '';
    const image = tags.find((t: string[]) => t[0] === 'image')?.[1] || '';
    const status = (tags.find((t: string[]) => t[0] === 'status')?.[1] || 'planned') as LiveEvent['status'];
    const starts = Number(tags.find((t: string[]) => t[0] === 'starts')?.[1] || ev.created_at || 0);
    const tTags = tags.filter((t: string[]) => t[0] === 't').map((t: string[]) => t[1]);
    const dTag = tags.find((t: string[]) => t[0] === 'd')?.[1] || '';
    const address = `${NOSTR_KINDS.nip53_live_event}:${ev.pubkey}:${dTag}`;

    return { id: ev.id, pubkey: ev.pubkey, title, summary, streaming, image, status, starts, tags: tTags, dTag, address };
  }

  async function loadEvents() {
    loading = true;
    const ndk = await ensureNdk();
    const sub = ndk.subscribe(
      { kinds: [NOSTR_KINDS.nip53_live_event], limit: 100 },
      { closeOnEose: false },
    );

    sub.on('event', (ev) => {
      const parsed = parseEvent(ev);
      if (!parsed) return;
      void fetchProfileFor(parsed.pubkey);
      liveEvents = [parsed, ...liveEvents.filter((e) => e.id !== parsed.id)]
        .sort((a, b) => {
          const statusOrder = { live: 0, planned: 1, ended: 2 };
          if (statusOrder[a.status] !== statusOrder[b.status]) return statusOrder[a.status] - statusOrder[b.status];
          return b.starts - a.starts;
        })
        .slice(0, 200);
    });

    sub.on('eose', () => (loading = false));
    stop = () => sub.stop();
  }

  async function selectEvent(ev: LiveEvent) {
    if (chatStop) chatStop();
    chatMessages = [];
    selectedEvent = ev;

    const ndk = await ensureNdk();
    const sub = ndk.subscribe(
      { kinds: [NOSTR_KINDS.nip53_live_chat], '#a': [ev.address], limit: 200 },
      { closeOnEose: false },
    );

    sub.on('event', (cev) => {
      void fetchProfileFor(cev.pubkey!);
      const msg: ChatMessage = { id: cev.id!, pubkey: cev.pubkey!, content: cev.content || '', createdAt: cev.created_at! };
      chatMessages = [...chatMessages.filter((m) => m.id !== msg.id), msg].sort((a, b) => a.createdAt - b.createdAt).slice(-300);
    });

    chatStop = () => sub.stop();
  }

  async function sendChat() {
    if (!selectedEvent || !chatText.trim() || !$isAuthed) return;
    chatBusy = true;
    try {
      const pk = await window.nostr!.getPublicKey();
      const unsigned = {
        kind: NOSTR_KINDS.nip53_live_chat,
        created_at: Math.floor(Date.now() / 1000),
        content: chatText.trim(),
        tags: [['a', selectedEvent.address, '', 'root']],
        pubkey: pk,
      };
      const signed = await signWithNip07(unsigned as any);
      await publishSignedEvent(signed as any);
      chatText = '';
    } catch {
      // silently fail
    } finally {
      chatBusy = false;
    }
  }

  async function publishGoLive() {
    goLiveError = null;
    goLiveOk = null;
    if (!$isAuthed || !$pubkey) { goLiveError = 'Connect your signer first.'; return; }
    if (!goLiveTitle.trim()) { goLiveError = 'Title is required.'; return; }
    goLiveBusy = true;

    try {
      const d = `bfta-live-${Date.now()}`;
      const tags: string[][] = [
        ['d', d],
        ['title', goLiveTitle.trim()],
        ['status', goLiveStreamUrl.trim() ? 'live' : 'planned'],
      ];
      if (goLiveSummary.trim()) tags.push(['summary', goLiveSummary.trim()]);
      if (goLiveStreamUrl.trim()) tags.push(['streaming', goLiveStreamUrl.trim()]);
      if (goLiveImage.trim()) tags.push(['image', goLiveImage.trim()]);
      tags.push(['starts', String(Math.floor(Date.now() / 1000))]);
      for (const t of goLiveTags.split(',').map((s) => s.trim().replace(/^#/, '')).filter(Boolean)) {
        tags.push(['t', t]);
      }

      const unsigned = {
        kind: NOSTR_KINDS.nip53_live_event,
        created_at: Math.floor(Date.now() / 1000),
        content: '',
        tags,
        pubkey: $pubkey,
      };
      const signed = await signWithNip07(unsigned as any);
      await publishSignedEvent(signed as any);
      goLiveOk = 'Live event published to relays.';
      showGoLive = false;
    } catch (e) {
      goLiveError = e instanceof Error ? e.message : String(e);
    } finally {
      goLiveBusy = false;
    }
  }

  onMount(() => void loadEvents());
  onDestroy(() => {
    if (stop) stop();
    if (chatStop) chatStop();
  });
</script>

<div class="live-hero card">
  <div class="hero-inner">
    <h1 class="hero-title">Live Sessions</h1>
    <p class="muted hero-desc">
      Watch and host live art demos, music performances, and collaborative sessions.
      Chat in real-time and support artists with Lightning zaps.
    </p>
    {#if $isAuthed}
      <button class="btn primary go-live-btn" on:click={() => (showGoLive = !showGoLive)}>
        {showGoLive ? 'Cancel' : 'Go Live'}
      </button>
    {/if}
  </div>
</div>

{#if showGoLive}
  <div class="card go-live-form" style="margin-top: 1rem; padding: 1rem;">
    <div style="font-weight: 900; font-size: 1.1rem;">Create Live Event (NIP-53)</div>
    <div class="muted" style="margin-top: 0.3rem; line-height: 1.5;">
      Publish a live event to Nostr relays. Use OBS or similar to stream to an RTMP endpoint, then paste the HLS/stream URL below.
    </div>

    <div class="grid cols-2" style="margin-top: 0.9rem;">
      <div>
        <div class="muted" style="margin-bottom:0.35rem;">Title</div>
        <input class="input" bind:value={goLiveTitle} placeholder="Live painting session, music jam..." />
      </div>
      <div>
        <div class="muted" style="margin-bottom:0.35rem;">Tags (comma-separated)</div>
        <input class="input" bind:value={goLiveTags} placeholder="#BitcoinArt, #LiveMusic..." />
      </div>
    </div>

    <div style="margin-top: 0.9rem;">
      <div class="muted" style="margin-bottom:0.35rem;">Stream URL (HLS or embed URL)</div>
      <input class="input" bind:value={goLiveStreamUrl} placeholder="https://stream.example.com/live.m3u8" />
    </div>

    <div class="grid cols-2" style="margin-top: 0.9rem;">
      <div>
        <div class="muted" style="margin-bottom:0.35rem;">Summary</div>
        <input class="input" bind:value={goLiveSummary} placeholder="Short description..." />
      </div>
      <div>
        <div class="muted" style="margin-bottom:0.35rem;">Cover image URL</div>
        <input class="input" bind:value={goLiveImage} placeholder="https://..." />
      </div>
    </div>

    <div style="margin-top: 1rem; display: flex; gap: 0.5rem; align-items: center;">
      <button class="btn primary" disabled={goLiveBusy} on:click={publishGoLive}>
        {goLiveBusy ? 'Publishing...' : 'Publish Live Event'}
      </button>
      {#if goLiveOk}<span class="muted">{goLiveOk}</span>{/if}
    </div>

    {#if goLiveError}
      <div class="card" style="margin-top: 0.75rem; padding: 0.8rem; border-color: rgba(251,113,133,0.35);">
        <div class="muted">{goLiveError}</div>
      </div>
    {/if}
  </div>
{/if}

<div class="grid cols-2" style="margin-top: 1rem;">
  <div>
    <div style="font-weight: 900; margin-bottom: 0.65rem;">Events</div>
    {#if loading}
      <div class="card" style="padding: 1rem;"><div class="muted">Loading live events from relays...</div></div>
    {/if}

    <div class="events-list">
      {#each liveEvents as ev (ev.id)}
        <button
          class="card event-card"
          class:active={selectedEvent?.id === ev.id}
          on:click={() => selectEvent(ev)}
        >
          <div class="event-top">
            <span class="status-badge" class:live={ev.status === 'live'} class:planned={ev.status === 'planned'} class:ended={ev.status === 'ended'}>
              {ev.status}
            </span>
            <span class="muted event-time">{new Date(ev.starts * 1000).toLocaleString()}</span>
          </div>
          <div class="event-title">{ev.title}</div>
          {#if ev.summary}
            <div class="muted event-summary">{ev.summary}</div>
          {/if}
          <div class="event-author muted">
            {$profileByPubkey[ev.pubkey]?.display_name || $profileByPubkey[ev.pubkey]?.name || npubFor(ev.pubkey).slice(0, 16) + '...'}
          </div>
          {#if ev.tags.length}
            <div class="event-tags">
              {#each ev.tags.slice(0, 4) as t}
                <span class="pill muted" style="font-size: 0.75rem; padding: 0.15rem 0.4rem;">#{t}</span>
              {/each}
            </div>
          {/if}
        </button>
      {/each}
      {#if !loading && liveEvents.length === 0}
        <div class="card" style="padding: 1rem;">
          <div class="muted">No live events found yet. Be the first to go live!</div>
        </div>
      {/if}
    </div>
  </div>

  <div>
    {#if selectedEvent}
      <div class="card" style="padding: 1rem;">
        <div style="display: flex; align-items: center; justify-content: space-between; gap: 0.5rem; flex-wrap: wrap;">
          <div style="font-weight: 950; font-size: 1.2rem;">{selectedEvent.title}</div>
          <div style="display: flex; gap: 0.5rem;">
            <a class="btn" href={`${base}/profile/${npubFor(selectedEvent.pubkey)}`}>View Profile</a>
            <button class="btn primary" on:click={() => { zapTarget = selectedEvent?.pubkey ?? ''; zapLabel = $profileByPubkey[selectedEvent?.pubkey ?? '']?.display_name || 'Artist'; zapOpen = true; }}>
              Zap
            </button>
          </div>
        </div>

        {#if selectedEvent.streaming}
          <div class="stream-embed" style="margin-top: 0.85rem;">
            {#if selectedEvent.streaming.includes('.m3u8')}
              <div class="card" style="padding: 1.5rem 1rem; text-align: center; background: rgba(0,0,0,0.3);">
                <div class="muted">HLS stream available:</div>
                <a href={selectedEvent.streaming} target="_blank" rel="noreferrer" class="pill" style="margin-top: 0.5rem;">
                  Open Stream
                </a>
              </div>
            {:else if selectedEvent.streaming.includes('youtube') || selectedEvent.streaming.includes('youtu.be')}
              <iframe
                src={selectedEvent.streaming.replace('watch?v=', 'embed/')}
                title="Live Stream"
                width="100%"
                height="320"
                style="border: none; border-radius: 12px;"
                allowfullscreen
              ></iframe>
            {:else}
              <div class="card" style="padding: 1rem; text-align: center; background: rgba(0,0,0,0.3);">
                <a href={selectedEvent.streaming} target="_blank" rel="noreferrer" class="btn primary">
                  Open Stream
                </a>
              </div>
            {/if}
          </div>
        {/if}

        {#if selectedEvent.image && !selectedEvent.streaming}
          <div style="margin-top: 0.85rem;">
            <img src={selectedEvent.image} alt="" loading="lazy" style="width: 100%; border-radius: 12px; max-height: 240px; object-fit: cover;" />
          </div>
        {/if}

        <div class="chat-section" style="margin-top: 1rem;">
          <div style="font-weight: 850; margin-bottom: 0.5rem;">Live Chat (kind:{NOSTR_KINDS.nip53_live_chat})</div>

          <div class="chat-messages">
            {#each chatMessages as m (m.id)}
              <div class="chat-msg">
                <span class="chat-author">{$profileByPubkey[m.pubkey]?.display_name || $profileByPubkey[m.pubkey]?.name || m.pubkey.slice(0, 8) + '...'}</span>
                <span class="chat-text">{m.content}</span>
              </div>
            {/each}
            {#if chatMessages.length === 0}
              <div class="muted" style="padding: 0.5rem;">No chat messages yet.</div>
            {/if}
          </div>

          {#if $isAuthed}
            <div style="display: flex; gap: 0.5rem; margin-top: 0.5rem;">
              <input class="input" bind:value={chatText} placeholder="Say something..." on:keydown={(e) => e.key === 'Enter' && sendChat()} />
              <button class="btn primary" disabled={chatBusy || !chatText.trim()} on:click={sendChat}>Send</button>
            </div>
          {:else}
            <div class="muted" style="margin-top: 0.5rem;">Connect your signer to chat.</div>
          {/if}
        </div>
      </div>
    {:else}
      <div class="card" style="padding: 2rem 1.5rem; text-align: center;">
        <div style="font-size: 2rem; margin-bottom: 0.5rem;">📡</div>
        <div class="muted">Select a live event to view the stream and chat.</div>
      </div>
    {/if}
  </div>
</div>

<ZapComposer
  open={zapOpen}
  recipientPubkey={zapTarget}
  recipientLabel={zapLabel}
  onClose={() => (zapOpen = false)}
/>

<style>
  .live-hero {
    padding: 1.5rem 1.3rem;
    text-align: center;
    border-color: rgba(139, 92, 246, 0.2);
    background: linear-gradient(180deg, rgba(139, 92, 246, 0.08), rgba(255, 255, 255, 0.04));
  }
  .hero-inner {
    max-width: 560px;
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
  .go-live-btn {
    margin-top: 1rem;
    font-size: 1rem;
    padding: 0.7rem 1.5rem;
  }
  .events-list {
    display: grid;
    gap: 0.6rem;
  }
  .event-card {
    padding: 0.85rem 1rem;
    text-align: left;
    cursor: pointer;
    width: 100%;
    transition: border-color 0.15s, background 0.15s;
  }
  .event-card:hover {
    border-color: rgba(139, 92, 246, 0.3);
  }
  .event-card.active {
    border-color: rgba(139, 92, 246, 0.45);
    background: rgba(139, 92, 246, 0.06);
  }
  .event-top {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-bottom: 0.35rem;
  }
  .status-badge {
    font-size: 0.72rem;
    font-weight: 750;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    padding: 0.15rem 0.5rem;
    border-radius: 6px;
  }
  .status-badge.live {
    background: rgba(239, 68, 68, 0.2);
    color: #ef4444;
    border: 1px solid rgba(239, 68, 68, 0.3);
  }
  .status-badge.planned {
    background: rgba(246, 196, 83, 0.15);
    color: var(--accent);
    border: 1px solid rgba(246, 196, 83, 0.25);
  }
  .status-badge.ended {
    background: rgba(255, 255, 255, 0.05);
    color: var(--muted);
    border: 1px solid var(--border);
  }
  .event-time {
    font-size: 0.78rem;
  }
  .event-title {
    font-weight: 850;
    font-size: 1rem;
  }
  .event-summary {
    margin-top: 0.25rem;
    font-size: 0.88rem;
    line-height: 1.35;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }
  .event-author {
    margin-top: 0.45rem;
    font-size: 0.82rem;
  }
  .event-tags {
    margin-top: 0.4rem;
    display: flex;
    flex-wrap: wrap;
    gap: 0.25rem;
  }
  .chat-messages {
    max-height: 280px;
    overflow-y: auto;
    border: 1px solid var(--border);
    border-radius: 12px;
    padding: 0.5rem 0.65rem;
    background: rgba(0, 0, 0, 0.2);
    display: flex;
    flex-direction: column;
    gap: 0.3rem;
  }
  .chat-msg {
    font-size: 0.88rem;
    line-height: 1.35;
  }
  .chat-author {
    font-weight: 750;
    color: var(--accent);
    margin-right: 0.4rem;
  }
  .chat-text {
    color: var(--text);
  }
</style>
