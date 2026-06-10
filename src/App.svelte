<script>
  // @ts-nocheck
  import Bee from './Bee.svelte';
  // import Chicken from './Chicken.svelte';
  // import Blobfish from './Blobfish.svelte';
  let page = 'menu';

  function openPage(name) {
    page = name;
  }
</script>

<div class="app-shell">
  {#if page === 'menu'}
    <div class="menu-card">
      <h1>Choose a page</h1>
      <p>Select the bee, chicken, or pond blobfish scene.</p>
      <div class="button-grid">
        <button class="page-btn" on:click={() => openPage('bee')}>Bee</button>
        <button class="page-btn" on:click={() => openPage('chicken')}>Chicken</button>
        <button class="page-btn" on:click={() => openPage('blobfish')}>Blobfish</button>
      </div>
    </div>
  {:else if page === 'bee'}
    <Bee on:back={() => openPage('menu')} />
  {:else if page === 'chicken'}
    <Chicken on:back={() => openPage('menu')} />
  {:else if page === 'blobfish'}
    <Blobfish on:back={() => openPage('menu')} />
  {:else}
    <div class="menu-card placeholder">
      <h1>{page === 'page3' ? 'Page 3' : 'Page'}</h1>
      <p>This page is a placeholder. You can create it later and wire it in here.</p>
      <button class="back-btn" on:click={() => openPage('menu')}>Back to menu</button>
    </div>
  {/if}
</div>

<style>
  :global(body) {
    margin: 0;
    min-height: 100vh;
    font-family: 'Inter', sans-serif;
    background: #f0f7fb;
    color: #1f2937;
  }

  .app-shell {
    min-height: 100vh;
    display: grid;
    place-items: center;
    padding: 24px;
  }

  .menu-card {
    width: min(520px, 100%);
    padding: 32px;
    border-radius: 24px;
    background: white;
    box-shadow: 0 24px 80px rgba(15, 23, 42, 0.12);
    text-align: center;
  }

  .menu-card h1 {
    margin: 0 0 16px;
    font-size: 2rem;
    line-height: 1.1;
  }

  .menu-card p {
    margin: 0 0 28px;
    color: #4b5563;
    line-height: 1.7;
  }

  .button-grid {
    display: grid;
    gap: 14px;
  }

  .page-btn,
  .back-btn {
    cursor: pointer;
    width: 100%;
    padding: 16px 20px;
    border: none;
    border-radius: 18px;
    font-size: 1rem;
    font-weight: 700;
    background: #1d4ed8;
    color: white;
    transition: transform 0.14s ease, box-shadow 0.14s ease, background 0.14s ease;
  }

  .page-btn:hover,
  .back-btn:hover {
    transform: translateY(-1px);
    background: #2563eb;
  }

  .placeholder {
    max-width: 640px;
  }
</style>
