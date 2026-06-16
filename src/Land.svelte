<script>
// @ts-nocheck
  import { onMount, onDestroy } from 'svelte';
  import Zdog from 'zdog';

  const TAU = Zdog.TAU;

  // tweakables
  const MIN_ZOOM = 0.15;    // lower = can zoom out further
  const MAX_ZOOM = 16;
  const START_ZOOM = 0.6;   // initial zoom (lower = smaller on load)
  const ZOOM_EASE = 0.12;   // lower = slower zoom glide
  const ZOOM_STEP = 1.06;   // zoom per wheel step (lower = slower)
  const SPIN_SPEED = 0.003;
  const LAND_SCALE = 0.82;  // overall size of the island (lower = smaller)

  // dove placement (shared by the model and the click hit-test)
  const DOVE_POS   = { x: 220, y: -280, z: -360 };
  const DOVE_SCALE = 1.35;

  // bee placement (high in open sky so it never sorts behind the grass disk)
  const BEE_POS   = { x: -200, y: -380, z: -140 };
  const BEE_SCALE = 1.1;

  // navigation callback — call from a parent: <Island onSelectDove={() => view = 'dove'} />
  let { onSelectDove, onSelectBee } = $props();

  // reactive state (Svelte 5 runes)
  let canvas = $state(null);   // bound to the <canvas>

  // plain (non-reactive) state
  let illo;
  let rafId;
  let spin = true;
  let targetZoom = START_ZOOM;
  let anchorX = 0;
  let anchorY = 0;
  let dove;                  // { dove, frontWing, backWing } handle
  let bee;                   // { cont, leftWing, rightWing, antler } handle
  let wingBeat = 0;          // drives the dove flap
  let beeFrame = 0;          // drives the bee buzzing animation
  let downX = 0;             // pointer-down position (to tell click vs drag)
  let downY = 0;

  const lerp = (a, b, t) => a + (b - a) * t;

  function buildScene(illo) {
    const GRASS      = '#5ab030';
    const GRASS_DARK = '#3a8018';
    const DIRT       = '#9A6438';   // soil side of the island
    const DIRT_DARK  = '#6E4424';   // shaded underside
    const POND       = '#73BFF5';
    const POND_EDGE  = '#5EA7DC';
    const BRANCH     = '#6B4A2A';
    const TREE       = '#4A7B28';
    const TREE_DARK  = '#36601A';
    const CLOUD_PINK  = '#FFE4F0';
    const CLOUD_WHITE = '#FFF7FB';

    // The grass surface plane (top of the island) lives at this height.
    const SURFACE_Y = 170;
    const ISLAND_DEPTH = 260;        // thickness of the slab
    const ISLAND_Z = -310;           // disc centre on the z axis

    // Everything that should rotate lives under this anchor.
    const r = new Zdog.Anchor({ addTo: illo });

    // The land + its contents share one scaled group so they resize together.
    const land = new Zdog.Anchor({ addTo: r, scale: LAND_SCALE });

    // ── Island slab: a cylinder lying flat, giving the land real depth ──
    // Rotated 90° about x so its length runs vertically; the front cap
    // (grass) points up, the curved side shows the soil, giving thickness.
    new Zdog.Cylinder({
      addTo: land,
      diameter: 2160,
      length: ISLAND_DEPTH,
      color: DIRT,            // curved soil side
      frontFace: GRASS,       // top surface
      backface: DIRT_DARK,    // shaded bottom
      stroke: false,
      rotate: { x: TAU / 4 },
      translate: { y: SURFACE_Y + ISLAND_DEPTH / 2, z: ISLAND_Z },
    });

    // Subtle two-tone grass: a slightly smaller darker patch on top.
    new Zdog.Ellipse({
      addTo: land, diameter: 1820, stroke: 0, fill: true, color: GRASS_DARK,
      rotate: { x: TAU / 4 }, translate: { y: SURFACE_Y - 1, z: ISLAND_Z },
    });

    // ── Pond: flush with the grass surface (sits IN the land, not floating) ──
    // Bank rim slightly raised, water slightly recessed -> reads as a basin.
    new Zdog.Ellipse({
      addTo: land, diameter: 600, stroke: 24, color: POND_EDGE,
      rotate: { x: TAU / 4 }, translate: { x: 100, y: SURFACE_Y - 4, z: ISLAND_Z },
    });
    new Zdog.Ellipse({
      addTo: land, diameter: 576, stroke: 0, fill: true, color: POND,
      rotate: { x: TAU / 4 }, translate: { x: 100, y: SURFACE_Y - 2, z: ISLAND_Z },
    });

    // ── Trees, planted on the grass surface (y = SURFACE_Y) ──
    function createTree(parent, x, z, scale = 1) {
      const t = new Zdog.Anchor({ addTo: parent, translate: { x, y: SURFACE_Y, z }, scale });
      new Zdog.Shape({ addTo: t, path: [{ x: 0, y: 0 }, { x: 0, y: -180 }], stroke: 34, color: BRANCH });
      new Zdog.Shape({ addTo: t, stroke: 190, color: TREE,      translate: { y: -210 } });
      new Zdog.Shape({ addTo: t, stroke: 160, color: TREE_DARK, translate: { x: -52, y: -190, z: 18 } });
      new Zdog.Shape({ addTo: t, stroke: 160, color: TREE_DARK, translate: { x: 52,  y: -190, z: 18 } });
    }

    createTree(land, -480, -480, 0.85);
    createTree(land,  500, -200, 1.00);
    createTree(land,  180, -680, 0.80);
    createTree(land, -420, -100, 1.05);
    createTree(land,  560, -560, 0.75);
    createTree(land, -560, -240, 0.90);

    // ── Clouds (rotate with the land, but stay full-size as backdrop) ──
    function createCloud(parent, x, y, z, s = 1) {
      const c = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale: s });
      new Zdog.Shape({ addTo: c, stroke: 110, color: CLOUD_WHITE });
      new Zdog.Shape({ addTo: c, stroke: 76,  color: CLOUD_PINK,  translate: { x: -58, y: 12 } });
      new Zdog.Shape({ addTo: c, stroke: 82,  color: CLOUD_WHITE, translate: { x: 56,  y: 6 } });
      new Zdog.Shape({ addTo: c, stroke: 64,  color: CLOUD_PINK,  translate: { x: -98, y: 16 } });
      new Zdog.Shape({ addTo: c, stroke: 58,  color: CLOUD_WHITE, translate: { x: 100, y: 12 } });
    }

    createCloud(r, -680, -360, -500, 1.25);
    createCloud(r, -180, -450, -960, 1.15);
    createCloud(r,  320, -360, -600, 1.10);

    // ── Dove (shapes only; wings returned so tick() can flap them) ──
    // Added to r so it rotates with the scene like the clouds.
    function createDove(parent, x, y, z, scale = 1) {
      const C = {
        white:  '#ffffff',
        mid:    '#f2f2f2',
        shade:  '#DAD6CE',
        beak:   '#D89A6E',
        beakDk: '#C07E50',
        cheek:  '#F0A8A0',
        eye:    '#1A1410',
      };

      const d = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale });

      // body & neck
      new Zdog.Shape({ addTo: d, path: [{ x: -10, y: 15 }, { x: 25, y: 10 }], stroke: 52, color: C.white });
      new Zdog.Shape({ addTo: d, path: [{ x: -5, y: 15 }, { x: -35, y: -8 }], stroke: 36, color: C.white });

      // head & face
      const head = new Zdog.Anchor({ addTo: d, translate: { x: -35, y: -10, z: 0 } });
      new Zdog.Shape({ addTo: head, stroke: 35, color: C.white });
      new Zdog.Shape({ addTo: head, stroke: 6,   color: C.eye,   translate: { x: -6, y: -4, z: 15 } });
      new Zdog.Shape({ addTo: head, stroke: 1.8, color: '#fff',  translate: { x: -7.5, y: -5, z: 16 } });
      new Zdog.Shape({ addTo: head, stroke: 6,   color: C.eye,   translate: { x: -6, y: -4, z: -15 } });
      new Zdog.Shape({ addTo: head, stroke: 8,   color: C.cheek, translate: { x: -1, y: 4, z: 15 } });
      new Zdog.Shape({ addTo: head, stroke: 8,   color: C.cheek, translate: { x: -1, y: 4, z: -15 } });

      // beak
      const beak = new Zdog.Anchor({ addTo: head, translate: { x: -14, y: 2, z: 0 } });
      new Zdog.Shape({ addTo: beak, path: [{ x: 0, y: -4 }, { x: -18, y: 0 }, { x: 0, y: 4 }], stroke: 2, color: C.beak, fill: true });
      new Zdog.Shape({ addTo: beak, path: [{ x: 0, y: 0 }, { x: -14, y: 0 }, { x: 0, y: 2 }], stroke: 1.5, color: C.beakDk, fill: true, translate: { y: 2 } });

      // wings on their own anchors so they can rotate
      const wingPath = [
        { x: -5, y: 0 },
        { bezier: [{ x: 10, y: -45 }, { x: 45, y: -50 }, { x: 65, y: -15 }] },
        { bezier: [{ x: 50, y: 5 }, { x: 20, y: 15 }, { x: -5, y: 0 }] },
      ];
      const frontWing = new Zdog.Anchor({ addTo: d, translate: { x: -12, y: 10, z: 24 } });
      new Zdog.Shape({ addTo: frontWing, path: wingPath, stroke: 20, color: C.mid, fill: true });
      const backWing = new Zdog.Anchor({ addTo: d, translate: { x: -12, y: 10, z: -24 } });
      new Zdog.Shape({ addTo: backWing, path: wingPath, stroke: 20, color: C.mid, fill: true });

      // tail fan
      const tail = new Zdog.Anchor({ addTo: d, translate: { x: 22, y: 10 } });
      new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 38, y: 4 }], stroke: 16, color: C.shade });
      new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 32, y: 2 }], stroke: 12, color: C.mid,   translate: { z: 10 } });
      new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 32, y: 2 }], stroke: 12, color: C.shade, translate: { z: -10 } });

      return { dove: d, frontWing, backWing };
    }

    dove = createDove(r, DOVE_POS.x, DOVE_POS.y, DOVE_POS.z, DOVE_SCALE);

    // ── Bee (shapes only; wings + antennae returned so tick() can animate) ──
    // Added to r so it buzzes in the sky and rotates with the scene.
    function createBee(parent, x, y, z, scale = 1) {
      const B = { yellow: '#FCD116', black: '#3D2620', white: '#FFFDF0', cheek: '#F26B50' };

      const cont = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale });
      const b    = new Zdog.Anchor({ addTo: cont });

      // head + face
      const head = new Zdog.Shape({ addTo: b, stroke: 130, color: B.yellow });
      new Zdog.Shape({ addTo: head, stroke: 10, color: B.black, translate: { x: -22, y: 4, z: 36 } });
      new Zdog.Shape({ addTo: head, stroke: 10, color: B.black, translate: { x:  14, y: 4, z: 40 } });
      new Zdog.Shape({ addTo: head, stroke: 3.5, color: B.black, closed: false,
        path: [{ x: -4, y: 10, z: 42 }, { bezier: [{ x: -2, y: 13, z: 42 }, { x: 2, y: 13, z: 42 }, { x: 4, y: 10, z: 42 }] }] });
      new Zdog.Shape({ addTo: head, path: [{ x: -34, y: -14, z: 44 }, { x: -10, y: -22, z: 44 }], stroke: 8, color: B.black, translate: { x: -18, y: -8, z: 20 }, rotate: { z: -0.18 } });
      new Zdog.Shape({ addTo: head, path: [{ x: 10, y: -22, z: 44 }, { x: 34, y: -14, z: 44 }], stroke: 8, color: B.black, translate: { x: 18, y: -8, z: 20 }, rotate: { z: 0.18 } });
      new Zdog.Shape({ addTo: head, stroke: 14, color: B.cheek, translate: { x: -32, y: 14, z: 24 } });
      new Zdog.Shape({ addTo: head, stroke: 14, color: B.cheek, translate: { x:  24, y: 14, z: 30 } });

      // antennae (bob in tick)
      const antler = new Zdog.Anchor({ addTo: head, translate: { y: -44, z: 10 } });
      new Zdog.Shape({ addTo: antler, path: [{ y: 0, x: -10 }, { y: -22, x: -24, z: 8 }],  stroke: 4.5, color: B.black });
      new Zdog.Shape({ addTo: antler, path: [{ y: 0, x:  10 }, { y: -22, x:  -4, z: 12 }], stroke: 4.5, color: B.black });

      // arms
      const leftArm  = new Zdog.Anchor({ addTo: b, translate: { x: -18, y: 45, z: 32 } });
      const rightArm = new Zdog.Anchor({ addTo: b, translate: { x:  18, y: 45, z: 32 } });
      new Zdog.Shape({ addTo: leftArm,  path: [{ y: 0 }, { y: 22 }], stroke: 7, color: B.black });
      new Zdog.Shape({ addTo: rightArm, path: [{ y: 0 }, { y: 22 }], stroke: 7, color: B.black });

      // striped body
      const body = new Zdog.Anchor({ addTo: b, translate: { y: 32, z: -35 } });
      const seg = new Zdog.Shape({ addTo: body, stroke: 140, color: B.yellow });
      seg.copy({ addTo: body, stroke: 162, color: B.black,  translate: { z: -32 } });
      seg.copy({ addTo: body, stroke: 168, color: B.yellow, translate: { z: -64 } });
      seg.copy({ addTo: body, stroke: 156, color: B.black,  translate: { z: -96 } });
      new Zdog.Shape({ addTo: body, stroke: 108, color: B.black, translate: { z: -123 } });

      // wings (flap in tick)
      const rightWing = new Zdog.Anchor({ addTo: body, translate: { z: -43, y: -65, x:  29 } });
      const leftWing  = new Zdog.Anchor({ addTo: body, translate: { z: -43, y: -65, x: -29 } });
      new Zdog.Ellipse({ addTo: rightWing, width: 80, height: 160, color: B.white, fill: true, rotate: { x: TAU/5, z:  TAU/5 }, translate: { x:  65 }, stroke: 0 });
      new Zdog.Ellipse({ addTo: leftWing,  width: 80, height: 160, color: B.white, fill: true, rotate: { x: TAU/5, z: -TAU/5 }, translate: { x: -65 }, stroke: 0 });

      return { cont, leftWing, rightWing, antler };
    }

    bee = createBee(r, BEE_POS.x, BEE_POS.y, BEE_POS.z, BEE_SCALE);
  }

  // Where the dove's centre lands on screen, in CSS px.
  // The dove sits under `r` (no rotation of its own) and the scene's spin
  // lives on illo.rotate.y, so the only rotation on the point is illo.rotate.
  function doveScreenPos() {
    const v = new Zdog.Vector(DOVE_POS);
    v.rotate(illo.rotate);
    const rect = canvas.getBoundingClientRect();
    return {
      x: rect.width  / 2 + illo.zoom * (v.x + illo.translate.x),
      y: rect.height / 2 + illo.zoom * (v.y + illo.translate.y),
    };
  }

  function beeScreenPos() {
    // use the bee's live (hovering) position, not the static start point
    const src = bee ? bee.cont.translate : BEE_POS;
    const v = new Zdog.Vector(src);
    v.rotate(illo.rotate);
    const rect = canvas.getBoundingClientRect();
    return {
      x: rect.width  / 2 + illo.zoom * (v.x + illo.translate.x),
      y: rect.height / 2 + illo.zoom * (v.y + illo.translate.y),
    };
  }

  function isOnDove(e) {
    if (!illo) return false;
    const rect = canvas.getBoundingClientRect();
    const p = doveScreenPos();
    const dx = (e.clientX - rect.left) - p.x;
    const dy = (e.clientY - rect.top)  - p.y;
    const hitRadius = 80 * DOVE_SCALE * illo.zoom; // generous, scales with zoom
    return Math.hypot(dx, dy) <= hitRadius;
  }

  function isOnBee(e) {
    if (!illo || !bee) return false;
    const rect = canvas.getBoundingClientRect();
    const p = beeScreenPos();
    const dx = (e.clientX - rect.left) - p.x;
    const dy = (e.clientY - rect.top)  - p.y;
    const hitRadius = 90 * BEE_SCALE * illo.zoom; // bee model is chunky → a bit bigger
    return Math.hypot(dx, dy) <= hitRadius;
  }

  // Offset of the cursor from the canvas centre, in CSS px.
  // Zdog's translate is in world units that equal CSS px at zoom 1, so this
  // matches without any devicePixelRatio scaling — which is what made the
  // old (canvas.width-scaled) version overshoot on retina screens.
  function cursorOffset(e) {
    const rect = canvas.getBoundingClientRect();
    return {
      x: (e.clientX - rect.left) - rect.width / 2,
      y: (e.clientY - rect.top) - rect.height / 2,
    };
  }

  // ── event handlers ──
  function handleWheel(e) {
    e.preventDefault();
    // NOTE: zooming no longer stops the auto-spin — only dragging does.
    const o = cursorOffset(e);
    anchorX = o.x;
    anchorY = o.y;
    const factor = e.deltaY < 0 ? ZOOM_STEP : 1 / ZOOM_STEP;
    targetZoom = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, targetZoom * factor));
  }

  function handlePointerDown(e) {
    downX = e.clientX;
    downY = e.clientY;
    spin = false;
    canvas.style.cursor = 'grabbing';
  }

  function handlePointerUp(e) {
    if (canvas) canvas.style.cursor = 'grab';
    const moved = Math.hypot(e.clientX - downX, e.clientY - downY);
    if (moved < 6) {
      if (isOnDove(e))      onSelectDove?.();
      else if (isOnBee(e))  onSelectBee?.();
    }
  }

  // turn the cursor into a pointer when hovering the dove
  function handlePointerMove(e) {
    if (!canvas || canvas.style.cursor === 'grabbing') return;
    canvas.style.cursor = (isOnDove(e) || isOnBee(e)) ? 'pointer' : 'grab';
  }

  onMount(() => {
    illo = new Zdog.Illustration({
      element: canvas,
      dragRotate: true,
      resize: 'window',
      zoom: START_ZOOM,
      rotate: { x: -TAU / 9, y: TAU / 10 },
    });

    buildScene(illo);

    // Zdog renders:  screen = center + zoom * (worldPoint + illo.translate)
    // Keep the point under the cursor fixed when zoom changes z0 -> z1:
    //   translate += anchorOffset * (1/z1 - 1/z0)
    function tick() {
      rafId = requestAnimationFrame(tick);
      if (spin) illo.rotate.y += SPIN_SPEED;

      // gentle dove wing flap (no GSAP — same sin math as the original dove)
      wingBeat += 0.13;
      if (dove) {
        dove.frontWing.rotate.z = Math.sin(wingBeat) * 0.18;
        dove.backWing.rotate.z  = Math.sin(wingBeat - 0.5) * 0.18;
      }

      // bee buzzing: fast wing flap, antenna bob, gentle hover drift
      beeFrame++;
      if (bee) {
        const f = beeFrame;
        bee.rightWing.rotate.z = -TAU / 6 + (TAU / 10) * Math.sin(f / 0.9);
        bee.leftWing.rotate.z  =  TAU / 6 - (TAU / 10) * Math.sin(f / 0.9);
        bee.antler.rotate.x    = (TAU / 40) * Math.sin(f / 10);
        bee.cont.translate.y   = BEE_POS.y + Math.sin(f / 14) * 6;
        bee.cont.translate.x   = BEE_POS.x + Math.cos(f / 28) * 5;
      }

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
  on:pointermove={handlePointerMove}
  class="land-canvas"
></canvas>

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
    background: #FFD9EC; /* static pink background — does not rotate */
    cursor: grab;
    touch-action: none;
    display: block;
    border: none;
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }
</style>