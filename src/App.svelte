<script>
  // @ts-nocheck
  import Land from './Land.svelte';
  import Bee from './Bee.svelte';
  import Chicken from './Chicken.svelte';
  import Blobfish from './Blobfish.svelte';
  import Sloth from './Sloth.svelte';
  import Dove from './Dove.svelte';

  let page = 'land';

  function openPage(name) {
    page = name;
  }
</script>

{#if page === 'land'}
  <Land
    onSelectBee={() => openPage('bee')}
    onSelectDove={() => openPage('dove')}
    onSelectChicken={() => openPage('chicken')}
    onSelectFish={() => openPage('blobfish')}
    onSelectSloth={() => openPage('sloth')}
  />
{:else if page === 'bee'}
  <Bee on:back={() => openPage('land')} />
{:else if page === 'chicken'}
  <Chicken on:back={() => openPage('land')} />
{:else if page === 'blobfish'}
  <Blobfish on:back={() => openPage('land')} />
{:else if page === 'sloth'}
  <Sloth on:back={() => openPage('land')} />
{:else if page === 'dove'}
  <Dove on:back={() => openPage('land')} />
{/if}

<style>
  :global(body) {
    margin: 0;
    min-height: 100vh;
    overflow: hidden;
  }
</style>