<script>
//@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';

  let canvasRef;

  onMount(() => {
    const TAU = Zdog.TAU;
    const C = {
      skyTop:   '#BFD9EF',
      skyLow:   '#E7F1FA',
      sun:      '#FFF7E6',
      cloud:    '#FFFFFF',
      cloudDim: '#EAF2FA'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.1, y: 0.18, z: 0 },
      zoom: 1.0
    });

    const envGroup = new Zdog.Anchor({ addTo: scene });

    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2200, y: -1200 }, { x: 2200, y: -1200 }, { x: 2200, y: 0 }, { x: -2200, y: 0 }],
      stroke: 0, fill: true, color: C.skyTop, translate: { z: -860 }
    });
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2200, y: 0 }, { x: 2200, y: 0 }, { x: 2200, y: 1000 }, { x: -2200, y: 1000 }],
      stroke: 0, fill: true, color: C.skyLow, translate: { z: -860 }
    });
    new Zdog.Shape({ addTo: envGroup, stroke: 360, color: C.sun, translate: { x: 520, y: -260, z: -840 } });

    [
      { x: -460, y: -250, z: -560, s: 1.3 },
      { x: -120, y: -320, z: -640, s: 0.9 },
      { x:  380, y: -210, z: -520, s: 1.5 },
      { x:  120, y: -300, z: -600, s: 1.0 },
      { x:  760, y: -270, z: -640, s: 1.2 }
    ].forEach(p => {
      const c = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: c, stroke: 100, color: C.cloud });
      new Zdog.Shape({ addTo: c, stroke: 76, color: C.cloudDim, translate: { x: -58, y: 12 } });
      new Zdog.Shape({ addTo: c, stroke: 82, color: C.cloud,    translate: { x:  56, y:  6 } });
      new Zdog.Shape({ addTo: c, stroke: 64, color: C.cloudDim, translate: { x: -98, y: 16 } });
    });

    [{ x: -300, y: -360, z: -700 }, { x: -240, y: -340, z: -700 }, { x: 600, y: -380, z: -720 }].forEach(p => {
      const b = new Zdog.Anchor({ addTo: envGroup, translate: p, scale: 0.5 });
      new Zdog.Shape({ addTo: b, path: [{ x: -14, y: 6 }, { x: 0, y: 0 }, { x: 14, y: 6 }], stroke: 3, color: '#9FB4C6', closed: false });
    });

    scene.updateRenderGraph();
  });
</script>

<canvas bind:this={canvasRef} class="background-scene"></canvas>

<style>
  canvas.background-scene {
    position: fixed;
    inset: 0;
    width: 100vw;
    height: 100vh;
    z-index: 0;
    pointer-events: none;
    display: block;
  }
</style>
