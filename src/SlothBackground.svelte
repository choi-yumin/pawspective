<script>
  //@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';

  let canvasRef;

  onMount(() => {
    const TAU = Zdog.TAU;

    const color = {
      // Sloth Tree Colors
      branch:      '#6B4A2A',
      branchDk:    '#4E3318',
      leafGreen:   '#5DA052',
      leafDark:    '#3A6A28',
      vine:        '#4E7D3A',

      // Background Aesthetic Colors
      skyTop:      '#FFB7D5',
      skyLow:      '#FFE1F0',
      darkGreen:   '#3A5C18',
      land:        '#7ab535',
      landLight:   '#9fd95c',
      cloudPink:   '#FFE4F0',
      cloudWhite:  '#FFF7FB'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.10, y: -0.08, z: 0 } 
    });

    const envGroup = new Zdog.Anchor({ addTo: scene });

    // --- PINK SKY ---
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: -1100 }, { x: 2000, y: -1100 }, { x: 2000, y: 0 }, { x: -2000, y: 0 }],
      stroke: 0, fill: true, color: color.skyTop, translate: { z: -820 }
    });
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: 0 }, { x: 2000, y: 0 }, { x: 2000, y: 700 }, { x: -2000, y: 700 }],
      stroke: 0, fill: true, color: color.skyLow, translate: { z: -820 }
    });

    // --- CLOUDS ---
    [
      { x: -420, y: -480, z: -420, s: 1.2 },
      { x: -160, y: -510, z: -510, s: 0.9 },
      { x:  360, y: -420, z: -380, s: 1.4 },
      { x:   80, y: -490, z: -460, s: 1.0 },
      { x:  700, y: -460, z: -500, s: 1.1 }
    ].forEach(p => {
      const c = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: c, stroke: 95, color: color.cloudPink });
      new Zdog.Shape({ addTo: c, stroke: 72, color: color.cloudWhite, translate: { x: -55, y: 10 } });
      new Zdog.Shape({ addTo: c, stroke: 78, color: color.cloudPink,  translate: { x:  55, y:  5 } });
      new Zdog.Shape({ addTo: c, stroke: 62, color: color.cloudWhite, translate: { x: -95, y: 15 } });
      new Zdog.Shape({ addTo: c, stroke: 58, color: color.cloudPink,  translate: { x: 100, y: 12 } });
    });

    // --- HILLS ---
    function addHill(x, w, col, z, amp = 160) {
      const pts = [{ x: x - w / 2, y: 300 }];
      for (let i = 0; i <= 24; i++) {
        const t = i / 24;
        pts.push({ x: x - w / 2 + t * w, y: 300 - Math.sin(t * Math.PI) * amp });
      }
      pts.push({ x: x + w / 2, y: 300 });
      new Zdog.Shape({ addTo: envGroup, path: pts, stroke: 0, fill: true, color: col, translate: { z } });
    }

    addHill(-600, 900, color.darkGreen, -700, 160);
    addHill( 500, 800, '#4a7020',       -700, 160);
    addHill(-200, 700, '#5a8a28',       -680, 150);
    addHill( 900, 700, '#3d6a18',       -720, 170);

    // --- GROUND PLANES ---
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: 220 }, { x: 2000, y: 220 }, { x: 2000, y: 800 }, { x: -2000, y: 800 }],
      stroke: 0, fill: true, color: color.land, translate: { z: -500 }
    });
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: 220 }, { x: 2000, y: 220 }, { x: 2000, y: 260 }, { x: -2000, y: 260 }],
      stroke: 0, fill: true, color: color.landLight, translate: { z: -490 }
    });

    // --- ENLARGED SLOTH'S TREE ---
    const treeScaleGroup = new Zdog.Anchor({ addTo: scene, scale: 1.5, translate: { y: 120 } });
    const slothTreeGroup = new Zdog.Anchor({ addTo: treeScaleGroup, translate: { x: 0, y: 30, z: 80 } });
    
    const getBranchY = (x) => -250 - 0.1 * x;
    const trunkX = -440; 
    
    new Zdog.Shape({ addTo: slothTreeGroup, path: [{ x: trunkX, y: getBranchY(trunkX) }, { x: 200, y: getBranchY(200) }], stroke: 80, color: color.branch });
    
    new Zdog.Shape({ 
      addTo: slothTreeGroup, 
      path: [{ x: trunkX, y: -600 }, { x: trunkX, y: 700 }], 
      stroke: 240, 
      color: color.branch,
      translate: { z: 40 }
    });
    new Zdog.Shape({ 
      addTo: slothTreeGroup, 
      path: [{ x: trunkX + 60, y: -600 }, { x: trunkX + 60, y: 700 }], 
      stroke: 20, 
      color: color.branchDk,
      translate: { z: 40 }
    });

    [[-240, -250], [-40, -250], [140, -250], [320, -250]].forEach(([x, _]) => {
      const baseY = getBranchY(x);
      new Zdog.Shape({ addTo: slothTreeGroup, path: [{ x: x - 6, y: baseY - 4 }, { x: x + 8, y: baseY - 18 }], stroke: 6, color: color.branchDk });
    });
    
    [{ x: -280, yOffset: 14 }, { x: -140, yOffset: 10 }, { x: 40, yOffset: 14 }, { x: 220, yOffset: 12 }, { x: 400, yOffset: 10 }].forEach((p, i) => {
      new Zdog.Ellipse({ addTo: slothTreeGroup, width: 38, height: 20, stroke: 5, fill: true,
        color: i % 2 ? color.leafGreen : color.leafDark, 
        translate: { x: p.x, y: getBranchY(p.x) + 42 + p.yOffset, z: 10 }, 
        rotate: { z: (i % 2 ? 0.5 : -0.4) } 
      });
    });

    [{ x: -180, len: 120 }, { x: 360, len: 90 }].forEach(v => {
      const startY = getBranchY(v.x) + 36;
      new Zdog.Shape({ addTo: slothTreeGroup, path: [
        { x: v.x, y: startY },
        { bezier: [{ x: v.x + 16, y: startY + v.len * 0.4 }, { x: v.x - 16, y: startY + v.len * 0.7 }, { x: v.x + 6, y: startY + v.len }] }
      ], stroke: 5, color: color.vine, closed: false });
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