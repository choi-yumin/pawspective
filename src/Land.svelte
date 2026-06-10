<script>
// @ts-nocheck
  import { onMount, onDestroy } from 'svelte';
  import Zdog from 'zdog';

  const TAU = Zdog.TAU;

  // tweakables
  const MIN_ZOOM = 0.5;
  const MAX_ZOOM = 16;
  const ZOOM_EASE = 0.12;   // lower = slower zoom glide
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
    const GRASS='#5ab030',GRASSD='#3a8018',DIRT='#8b4a18',DIRTD='#6a3510';
    const r = new Zdog.Anchor({ addTo: illo });

    // floating island
    new Zdog.Ellipse({ addTo:r, width:300, height:220, rotate:{x:TAU/4}, stroke:0, color:GRASS, fill:true });
    new Zdog.Cylinder({ addTo:r, diameter:290, length:18, rotate:{x:TAU/4}, translate:{y:9}, stroke:false, color:GRASSD, backface:GRASS });
    new Zdog.Cylinder({ addTo:r, diameter:270, length:55, rotate:{x:TAU/4}, translate:{y:38}, stroke:false, color:DIRT, backface:DIRTD });
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
    const factor = e.deltaY < 0 ? 1.18 : 1 / 1.18;
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

<div class="wrap">
  <canvas
    bind:this={canvas}
    width="520"
    height="520"
    onpointerdown={handlePointerDown}
  ></canvas>
</div>

<style>
  .wrap {
    display: flex;
    justify-content: center;
  }
  canvas {
    background: transparent;
    cursor: grab;
    touch-action: none;
  }
</style>