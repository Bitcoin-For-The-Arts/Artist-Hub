<script lang="ts">
  import { base } from '$app/paths';
  import { browser } from '$app/environment';
  import { isAuthed } from '$lib/stores/auth';

  let showOnboarding = false;

  $: if (browser && !$isAuthed) {
    showOnboarding = !window.nostr?.getPublicKey;
  }
</script>

<div class="hero">
  <div class="hero-logo">
    <svg viewBox="0 0 64 64" xmlns="http://www.w3.org/2000/svg" aria-hidden="true" class="hero-icon">
      <rect width="64" height="64" rx="18" fill="url(#hero-grad)" />
      <text x="50%" y="54%" dominant-baseline="central" text-anchor="middle"
        font-size="36" font-weight="900" fill="#0b0b0f" font-family="system-ui, -apple-system, sans-serif">B</text>
      <defs>
        <linearGradient id="hero-grad" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%" stop-color="#f7931a" />
          <stop offset="100%" stop-color="#f6c453" />
        </linearGradient>
      </defs>
    </svg>
    <div class="hero-titles">
      <h1 class="hero-title">Bitcoin for the Arts</h1>
      <div class="hero-subtitle">Artist Hub</div>
    </div>
  </div>

  <p class="hero-desc muted">
    A decentralized social marketplace for artists. Communicate, share your creativity, collaborate, and offer services
    — all powered by Nostr events signed in your browser. Payments flow peer-to-peer over Lightning.
  </p>

  <div class="hero-features">
    <div class="feature-pill">Decentralized</div>
    <div class="feature-pill">No Account Needed</div>
    <div class="feature-pill">Lightning Payments</div>
    <div class="feature-pill">Your Keys, Your Identity</div>
  </div>

  <div class="hero-actions">
    <a class="btn primary hero-btn" href={`${base}/discover`}>Explore Marketplace</a>
    <a class="btn hero-btn" href={`${base}/create`}>Offer a Service</a>
    <a class="btn hero-btn" href={`${base}/live`}>Live Streams</a>
    <a class="btn hero-btn" href={`${base}/artstack`}>ArtStack Forum</a>
  </div>

  {#if showOnboarding}
    <div class="onboard-hint card">
      <div style="font-weight: 850;">New to Nostr?</div>
      <div class="muted" style="margin-top: 0.3rem; line-height: 1.5;">
        You need a Nostr signer extension to publish content and interact. Install
        <a href="https://getalby.com" target="_blank" rel="noreferrer" style="color: var(--accent);">Alby</a>
        or
        <a href="https://github.com/nicnocquee/nos2x" target="_blank" rel="noreferrer" style="color: var(--accent);">nos2x</a>
        to get started, or browse the marketplace in read-only mode.
      </div>
    </div>
  {/if}
</div>

<div class="info-grid">
  <div class="card info-card">
    <div class="info-icon">🎨</div>
    <div class="info-title">For Artists, By Artists</div>
    <div class="muted info-text">Share your work, find collaborators, and sell services in a Fiverr-style marketplace built on open protocols.</div>
  </div>
  <div class="card info-card">
    <div class="info-icon">⚡</div>
    <div class="info-title">Lightning Zaps</div>
    <div class="muted info-text">Instant, borderless payments. Tip artists, pay for commissions, and support creativity — all peer-to-peer.</div>
  </div>
  <div class="card info-card">
    <div class="info-icon">🔑</div>
    <div class="info-title">Own Your Identity</div>
    <div class="muted info-text">No passwords, no email sign-up. Your cryptographic keys are your identity across all Nostr clients.</div>
  </div>
  <div class="card info-card">
    <div class="info-icon">📡</div>
    <div class="info-title">Live Sessions</div>
    <div class="muted info-text">Stream live art demos, music performances, and collaborative sessions with real-time chat and zap support.</div>
  </div>
</div>

<style>
  .hero {
    text-align: center;
    padding: 1.5rem 0 0;
  }
  .hero-logo {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 1rem;
    margin-bottom: 1.2rem;
  }
  .hero-icon {
    width: 64px;
    height: 64px;
    flex-shrink: 0;
  }
  .hero-titles {
    text-align: left;
  }
  .hero-title {
    font-size: 2rem;
    font-weight: 950;
    margin: 0;
    line-height: 1.1;
    letter-spacing: -0.5px;
  }
  .hero-subtitle {
    font-size: 0.85rem;
    font-weight: 700;
    color: var(--accent);
    letter-spacing: 1.5px;
    text-transform: uppercase;
    margin-top: 0.15rem;
  }
  .hero-desc {
    max-width: 640px;
    margin: 0 auto;
    line-height: 1.65;
    font-size: 1.02rem;
  }
  .hero-features {
    display: flex;
    justify-content: center;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-top: 1.1rem;
  }
  .feature-pill {
    padding: 0.3rem 0.75rem;
    border-radius: 999px;
    font-size: 0.82rem;
    font-weight: 650;
    border: 1px solid rgba(246, 196, 83, 0.2);
    background: rgba(246, 196, 83, 0.06);
    color: var(--accent);
  }
  .hero-actions {
    display: flex;
    justify-content: center;
    gap: 0.5rem;
    flex-wrap: wrap;
    margin-top: 1.4rem;
  }
  .hero-btn {
    font-size: 0.95rem;
  }
  .onboard-hint {
    margin: 1.5rem auto 0;
    max-width: 480px;
    padding: 1rem 1.2rem;
    text-align: left;
    border-color: rgba(246, 196, 83, 0.3);
  }
  .info-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 1rem;
    margin-top: 2rem;
  }
  .info-card {
    padding: 1.1rem 1.2rem;
    transition: transform 0.2s, border-color 0.2s;
  }
  .info-card:hover {
    transform: translateY(-2px);
    border-color: rgba(246, 196, 83, 0.25);
  }
  .info-icon {
    font-size: 1.6rem;
    margin-bottom: 0.5rem;
  }
  .info-title {
    font-weight: 850;
    font-size: 1.02rem;
    margin-bottom: 0.35rem;
  }
  .info-text {
    line-height: 1.5;
    font-size: 0.92rem;
  }
  @media (max-width: 600px) {
    .hero-title {
      font-size: 1.5rem;
    }
    .info-grid {
      grid-template-columns: 1fr;
    }
    .hero-icon {
      width: 52px;
      height: 52px;
    }
  }
</style>

