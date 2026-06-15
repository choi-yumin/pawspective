<script>
// @ts-nocheck
  import { onMount, onDestroy } from 'svelte';
  import Zdog from 'zdog';
  import Bee from './Bee.svelte';
  import Chicken from './Chicken.svelte';
  import Sloth from './Sloth.svelte';
  import Dove from './Dove.svelte';
  import Blobfish from './Blobfish.svelte';

  const TAU = Zdog.TAU;

  // tweakables
  const MIN_ZOOM = 0.5;
  const MAX_ZOOM = 16;
  const ZOOM_EASE = 0.12;   // lower = slower zoom glide
  const ZOOM_STEP = 1.06;   // zoom per wheel step (lower = slower)
  const SPIN_SPEED = 0.003;

  // reactive state (Svelte 5 runes)
  let canvas = $state(null);   // bound to the <canvas>

  // plain (non-reactive) state
  let illo;
  let rafId;
  let spin = true;
  let targetZoom = 1;
  let anchorX = 0;
  let anchorY = 0;

  const lerp = (a, b, t) => a + (b - a) * t;

  function buildScene(illo) {
    const SKY_TOP = '#BFD9EF';
    const SKY_LOW = '#E7F1FA';
    const GRASS = '#5ab030';
    const GRASS_DARK = '#3a8018';
    const POND = '#73BFF5';
    const POND_EDGE = '#5EA7DC';
    const BRANCH = '#6B4A2A';
    const TREE = '#4A7B28';
    const TREE_DARK = '#36601A';
    const FLOWER_PINK = '#FF85A2';
    const FLOWER_YELLOW = '#FFB800';
    
    const r = new Zdog.Anchor({ addTo: illo });

    // Sky and ground
    new Zdog.Shape({ addTo: r, path: [{ x: -2600, y: -1400 }, { x: 2600, y: -1400 }, { x: 2600, y: 0 }, { x: -2600, y: 0 }], stroke: 0, fill: true, color: SKY_TOP, translate: { z: -1400 } });
    new Zdog.Shape({ addTo: r, path: [{ x: -2600, y: 0 }, { x: 2600, y: 0 }, { x: 2600, y: 1400 }, { x: -2600, y: 1400 }], stroke: 0, fill: true, color: SKY_LOW, translate: { z: -1400 } });

    new Zdog.Ellipse({ addTo: r, diameter: 2160, stroke: 0, fill: true, color: GRASS, rotate: { x: TAU / 4 }, translate: { y: 170, z: -310 } });
    new Zdog.Ellipse({ addTo: r, diameter: 1880, stroke: 0, fill: true, color: GRASS_DARK, rotate: { x: TAU / 4 }, translate: { y: 180, z: -260 } });
    new Zdog.Ellipse({ addTo: r, diameter: 760, stroke: 0, fill: true, color: POND, rotate: { x: TAU / 4 }, translate: { y: 140, z: -260 } });
    new Zdog.Ellipse({ addTo: r, diameter: 800, stroke: 24, color: POND_EDGE, rotate: { x: TAU / 4 }, translate: { y: 140, z: -260 } });

    // Clouds
    function createCloud(parent, x, y, z, s = 1) {
      const c = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale: s });
      new Zdog.Shape({ addTo: c, stroke: 110, color: '#FFFFFF' });
      new Zdog.Shape({ addTo: c, stroke: 76, color: '#EAF2FA', translate: { x: -58, y: 12 } });
      new Zdog.Shape({ addTo: c, stroke: 82, color: '#FFFFFF', translate: { x: 56, y: 6 } });
      new Zdog.Shape({ addTo: c, stroke: 64, color: '#EAF2FA', translate: { x: -98, y: 16 } });
    }

    createCloud(r, -680, -360, -920, 1.25);
    createCloud(r, -180, -380, -960, 1.15);
    createCloud(r, 320, -360, -980, 1.1);
    createCloud(r, 700, -300, -1040, 0.95);

    // Trees
    function createTree(parent, x, y, z, scale = 1) {
      const t = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale });
      new Zdog.Shape({ addTo: t, path: [{ x: 0, y: 0 }, { x: 0, y: -180 }], stroke: 34, color: BRANCH });
      new Zdog.Shape({ addTo: t, stroke: 190, color: TREE, translate: { y: -210 } });
      new Zdog.Shape({ addTo: t, stroke: 160, color: TREE_DARK, translate: { x: -52, y: -190, z: 18 } });
      new Zdog.Shape({ addTo: t, stroke: 160, color: TREE_DARK, translate: { x: 52, y: -190, z: 18 } });
    }

    createTree(r, -820, 30, -380, 1.1);
    createTree(r, 880, 60, -340, 0.85);

    // Flowers
    function createFlower(parent, x, y, z, scale = 1, color = FLOWER_PINK) {
      const f = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale });
      new Zdog.Shape({ addTo: f, path: [{ x: 0, y: 50 }, { x: 0, y: 0 }], stroke: 6, color: '#3A8018' });
      const top = new Zdog.Anchor({ addTo: f, translate: { y: -8 } });
      for (let i = 0; i < 8; i++) {
        const angle = (TAU / 8) * i;
        new Zdog.Ellipse({ addTo: top, width: 12, height: 30, fill: true, stroke: 0, color, translate: { x: Math.cos(angle) * 12, y: Math.sin(angle) * 8 }, rotate: { z: angle } });
      }
      new Zdog.Ellipse({ addTo: top, diameter: 14, fill: true, stroke: 4, color: FLOWER_YELLOW, translate: { z: 2 } });
    }

    createFlower(r, -560, 170, -250, 1.1, FLOWER_YELLOW);
    createFlower(r, -460, 190, -220, 1.0, FLOWER_PINK);
    createFlower(r, -380, 160, -210, 0.9, FLOWER_YELLOW);
    createFlower(r, 460, 190, -210, 1.0, FLOWER_PINK);
    createFlower(r, 520, 170, -230, 1.05, FLOWER_YELLOW);
    createFlower(r, 280, 150, -230, 0.85, FLOWER_PINK);
  }
// map client coords -> offset from canvas center, in logical px
  function cursorOffset(e) {
    const rect = canvas.getBoundingClientRect();
    const lx = (e.clientX - rect.left) * (canvas.width  / rect.width);
    const ly = (e.clientY - rect.top)  * (canvas.height / rect.height);
    return { x: lx - canvas.width / 2, y: ly - canvas.height / 2 };
  }

  // ── event handlers ──
  function handleWheel(e) {
    e.preventDefault();
    spin = false;
    const o = cursorOffset(e);
    anchorX = o.x;
    anchorY = o.y;
    const factor = e.deltaY < 0 ? ZOOM_STEP : 1 / ZOOM_STEP;
    targetZoom = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, targetZoom * factor));
  }

  function handlePointerDown() {
    spin = false;
    canvas.style.cursor = 'grabbing';
  }

  function handlePointerUp() {
    if (canvas) canvas.style.cursor = 'grab';
  }

  onMount(() => {
    illo = new Zdog.Illustration({
      element: canvas,
      dragRotate: true,
      resize: 'window',
      zoom: 1,
      rotate: { x: -TAU/9, y: TAU/10 },
    });

    buildScene(illo);

    // Zdog renders:  screen = center + zoom * (worldPoint + illo.translate)
    // Keep the point under the cursor fixed when zoom changes z0 -> z1:
    //   translate += anchorOffset * (1/z1 - 1/z0)
    function tick() {
      rafId = requestAnimationFrame(tick);
      if (spin) illo.rotate.y += SPIN_SPEED;

      const oldZoom = illo.zoom;
      const newZoom = lerp(oldZoom, targetZoom, ZOOM_EASE);
      if (Math.abs(newZoom - oldZoom) > 1e-5) {
        const k = (1 / newZoom) - (1 / oldZoom);
        illo.translate.x += anchorX * k;
        illo.translate.y += anchorY * k;
        illo.zoom = newZoom;
      }
      illo.updateRenderGraph();
    }
    tick();

    // wheel must be non-passive to allow preventDefault
    canvas.addEventListener('wheel', handleWheel, { passive: false });
    window.addEventListener('pointerup', handlePointerUp);
  });

  onDestroy(() => {
    cancelAnimationFrame(rafId);
    if (canvas) canvas.removeEventListener('wheel', handleWheel);
    if (typeof window !== 'undefined') window.removeEventListener('pointerup', handlePointerUp);
  });
</script>

<!-- full-viewport canvas — no surrounding container -->
<canvas
  bind:this={canvas}
  on:pointerdown={handlePointerDown}
  class="land-canvas"
></canvas>

<!-- Embedded animal components -->
<div class="animals-container">
  <div class="animal animal-sloth">
    <Sloth embedded={true} />
  </div>
  <div class="animal animal-chicken">
    <Chicken embedded={true} />
  </div>
  <div class="animal animal-bee">
    <Bee embedded={true} />
  </div>
  <div class="animal animal-blobfish">
    <Blobfish embedded={true} />
  </div>
  <div class="animal animal-dove">
    <Dove embedded={true} />
  </div>
</div>

<style>
  :global(html, body) {
    height: 100%;
    margin: 0;
    overflow: hidden;
  }
  .land-canvas {
    position: fixed;
    left: 0;
    top: 0;
    width: 100vw;
    height: 100vh;
    background: #f0f7fb; /* match app background to hide edges */
    cursor: grab;
    touch-action: none;
    display: block;
    border: none;
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  .animals-container {
    position: fixed;
    left: 0;
    top: 0;
    width: 100vw;
    height: 100vh;
    pointer-events: none;
    z-index: 1;
  }

  .animal {
    position: absolute;
    pointer-events: auto;
  }

  .animal-sloth {
    left: 15%;
    top: 55%;
    width: 200px;
    height: 200px;
  }

  .animal-chicken {
    right: 20%;
    top: 55%;
    width: 200px;
    height: 200px;
  }

  .animal-bee {
    right: 5%;
    top: 55%;
    width: 160px;
    height: 160px;
  }

  .animal-blobfish {
    left: 50%;
    top: 50%;
    transform: translateX(-50%) translateY(-50%);
    width: 200px;
    height: 200px;
  }

  .animal-dove {
    right: 15%;
    top: 15%;
    width: 200px;
    height: 200px;
  }

</style>