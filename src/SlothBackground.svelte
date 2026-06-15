<script>
//@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';

  let canvasRef;

  onMount(() => {
    const TAU = Zdog.TAU;

    const color = {
      fur:        '#8B6F4E',
      furDark:    '#6B5238',
      furLight:   '#B89B72',
      faceCream:  '#D8C29A',
      mask:       '#5A4632',
      nose:       '#3D2A1A',
      eye:        '#2A1C12',
      cheek:      '#E59A86',
      claw:       '#4A3826',
      moss:       '#7E8B5A',
      branch:     '#6B4A2A',
      branchDk:   '#4E3318',
      leafGreen:  '#5DA052',
      leafDark:   '#3A6A28',
      leafLight:  '#7FC04A',
      canopy1:    '#A7D88C',
      canopy2:    '#84C268',
      canopyDk:   '#4E7D3A',
      skyTop:     '#C9ECC0',
      skyLow:     '#E7F6DF',
      hibiscusR:  '#F0584C',
      hibiscusC:  '#FFC94D',
      vine:       '#4E7D3A'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.18, y: -0.1, z: 0 }
    });

    const envGroup = new Zdog.Anchor({ addTo: scene });

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

    function addCanopy(x, w, col, z, amp = 170) {
      const pts = [{ x: x - w / 2, y: 320 }];
      for (let i = 0; i <= 24; i++) {
        const t = i / 24;
        pts.push({ x: x - w / 2 + t * w, y: 320 - Math.sin(t * Math.PI) * amp });
      }
      pts.push({ x: x + w / 2, y: 320 });
      new Zdog.Shape({ addTo: envGroup, path: pts, stroke: 0, fill: true, color: col, translate: { z } });
    }

    addCanopy(-620, 1000, color.canopyDk, -720, 180);
    addCanopy( 540,  900, '#436f24',      -720, 175);
    addCanopy(-180,  780, color.canopy2,  -690, 150);
    addCanopy( 940,  760, '#3f6a22',      -740, 185);

    [
      { x: -560, y: -210, z: -440, s: 1.3, c: color.canopy1 },
      { x:  -60, y: -260, z: -520, s: 1.0, c: color.canopy2 },
      { x:  420, y: -190, z: -400, s: 1.5, c: color.canopy1 },
      { x:  760, y: -240, z: -480, s: 1.1, c: color.canopy2 }
    ].forEach(p => {
      const c = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: c, stroke: 110, color: p.c });
      new Zdog.Shape({ addTo: c, stroke: 84, color: color.canopyDk, translate: { x: -64, y: 12 } });
      new Zdog.Shape({ addTo: c, stroke: 90, color: p.c,            translate: { x:  60, y:  6 } });
    });

    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: 240 }, { x: 2000, y: 240 }, { x: 2000, y: 900 }, { x: -2000, y: 900 }],
      stroke: 0, fill: true, color: color.canopy2, translate: { z: -520 }
    });

    const fg = new Zdog.Anchor({ addTo: scene, translate: { x: 0, y: 30, z: 80 } });

    new Zdog.Shape({ addTo: fg, path: [{ x: -380, y: -250 }, { x: 380, y: -250 }], stroke: 36, color: color.branch });
    new Zdog.Shape({ addTo: fg, path: [{ x: -380, y: -264 }, { x: 380, y: -264 }], stroke: 9,  color: color.branchDk });
    [[-230, -250], [-90, -250], [120, -250], [250, -250]].forEach(([x, y]) => {
      new Zdog.Shape({ addTo: fg, path: [{ x: x - 6, y: y - 4 }, { x: x + 8, y: y - 18 }], stroke: 6, color: color.branchDk });
    });
    [{ x: -300, y: -236 }, { x: -150, y: -240 }, { x: 60, y: -236 }, { x: 320, y: -238 }].forEach((p, i) => {
      new Zdog.Ellipse({ addTo: fg, width: 38, height: 20, stroke: 5, fill: true,
        color: i % 2 ? color.leafGreen : color.leafDark, translate: { x: p.x, y: p.y + 22, z: 10 }, rotate: { z: (i % 2 ? 0.5 : -0.4) } });
    });

    [{ x: -350, len: 120 }, { x: 360, len: 90 }].forEach(v => {
      new Zdog.Shape({ addTo: fg, path: [
        { x: v.x, y: -250 },
        { bezier: [{ x: v.x + 16, y: -250 + v.len * 0.4 }, { x: v.x - 16, y: -250 + v.len * 0.7 }, { x: v.x + 6, y: -250 + v.len }] }
      ], stroke: 5, color: color.vine, closed: false });
    });

    const hibiscus = new Zdog.Anchor({ addTo: fg, translate: { x: 270, y: -120, z: 30 } });
    new Zdog.Shape({ addTo: hibiscus, path: [{ x: 0, y: -130 }, { x: 0, y: -16 }], stroke: 5, color: color.vine });
    const hibHead = new Zdog.Anchor({ addTo: hibiscus });
    for (let i = 0; i < 5; i++) {
      const a = (TAU / 5) * i;
      new Zdog.Ellipse({ addTo: hibHead, width: 26, height: 38, fill: true, stroke: 0, color: color.hibiscusR,
        translate: { x: Math.cos(a) * 14, y: Math.sin(a) * 14 }, rotate: { z: a } });
    }
    new Zdog.Shape({ addTo: hibHead, stroke: 16, color: color.hibiscusC, translate: { z: 3 } });
    new Zdog.Shape({ addTo: hibHead, path: [{ x: 0, y: 0 }, { x: 0, y: -22 }], stroke: 3, color: '#E0A92E', translate: { z: 5 } });

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
