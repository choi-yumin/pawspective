<script>
//@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';

  let canvasRef;

  onMount(() => {
    const TAU = Zdog.TAU;

    const color = {
      beeYellow:   '#FCD116',
      beeBlack:    '#3D2620',
      beeWhite:    '#FFFDF0',
      beeCheek:    '#F26B50',
      leafGreen:   '#5DA020',
      darkGreen:   '#3A5C18',
      land:        '#7ab535',
      landLight:   '#9fd95c',
      trunkBrown:  '#8B6340',
      cloudPink:   '#FFE4F0',
      cloudWhite:  '#FFF7FB',
      daisyWhite:  '#FFFFFF',
      daisyYellow: '#F9D342',
      roseRed:     '#F04A6F',
      rosePink:    '#FF8FAE',
      lavPurple:   '#C07EE0',
      lavLight:    '#E2B8F5',
      sunflowerY:  '#FFB800',
      sunflowerC:  '#7A3B10',
      tulipPink:   '#FF85A2',
      tulipOrange: '#FF6B35',
      honeyGold:   '#F2C94C',
      honeyDark:   '#D4A017'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.28, y: -0.12, z: 0 }
    });

    const envGroup = new Zdog.Anchor({ addTo: scene });

    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: -1000 }, { x: 2000, y: -1000 }, { x: 2000, y: 0 }, { x: -2000, y: 0 }],
      stroke: 0, fill: true, color: '#FFB7D5', translate: { z: -800 }
    });
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: 0 }, { x: 2000, y: 0 }, { x: 2000, y: 600 }, { x: -2000, y: 600 }],
      stroke: 0, fill: true, color: '#FFE1F0', translate: { z: -800 }
    });

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
    addHill(500,  800, '#4a7020',       -700, 160);
    addHill(-200, 700, '#5a8a28',       -680, 150);
    addHill(900,  700, '#3d6a18',       -720, 170);

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

    [
      { x: -420, y: -280, z: -420, s: 1.2 },
      { x: -160, y: -310, z: -510, s: 0.9 },
      { x:  360, y: -220, z: -380, s: 1.4 },
      { x:   80, y: -290, z: -460, s: 1.0 },
      { x:  700, y: -260, z: -500, s: 1.1 }
    ].forEach(p => {
      const c = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: c, stroke: 95, color: color.cloudPink });
      new Zdog.Shape({ addTo: c, stroke: 72, color: color.cloudWhite, translate: { x: -55, y: 10 } });
      new Zdog.Shape({ addTo: c, stroke: 78, color: color.cloudPink,  translate: { x:  55, y:  5 } });
      new Zdog.Shape({ addTo: c, stroke: 62, color: color.cloudWhite, translate: { x: -95, y: 15 } });
      new Zdog.Shape({ addTo: c, stroke: 58, color: color.cloudPink,  translate: { x: 100, y: 12 } });
    });

    [
      { x: -560, y: 140, z: -320, s: 1.0 },
      { x: -380, y: 160, z: -260, s: 0.8 },
      { x:  460, y: 150, z: -370, s: 1.2 },
      { x:  580, y: 170, z: -220, s: 0.85 },
      { x:  740, y: 145, z: -300, s: 0.9 },
      { x: -680, y: 155, z: -340, s: 1.0 }
    ].forEach(p => {
      const t = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: t, path: [{ y: 0 }, { y: -90 }], stroke: 18, color: color.trunkBrown });
      new Zdog.Shape({ addTo: t, stroke: 130, color: color.darkGreen, translate: { y: -130 } });
      new Zdog.Shape({ addTo: t, stroke: 100, color: color.leafGreen, translate: { y: -175, x: -20 } });
      new Zdog.Shape({ addTo: t, stroke: 90, color: '#7fcf38', translate: { y: -185, x: 20 } });
    });

    const fgGroup = new Zdog.Anchor({ addTo: scene, translate: { x: 0, y: 50, z: 120 } });

    function createDaisy(parent, x, y, z, scale, stemLen, rotX, rotY, petalCol) {
      petalCol = petalCol || color.daisyWhite;
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (120 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen, z: 0 }, { x: 0, y: 0, z: 0 }], stroke: 7, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 13; i++) {
        const ph = new Zdog.Anchor({ addTo: hd, rotate: { z: (TAU / 13) * i } });
        new Zdog.Ellipse({ addTo: ph, width: 11, height: 30, fill: true, color: petalCol, stroke: 0, translate: { y: -18 } });
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 15, fill: true, stroke: 4, color: color.daisyYellow, translate: { z: 1.5 } });
    }

    function createRose(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (140 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen }, { x: 0, y: 0 }], stroke: 7, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let l = 0; l < 3; l++) {
        const pc = 6 + l * 2, r = 10 + l * 9;
        for (let i = 0; i < pc; i++) {
          const a = (TAU / pc) * i + l * 0.3;
          const ph = new Zdog.Anchor({ addTo: hd, translate: { x: Math.cos(a) * r, y: Math.sin(a) * r } });
          new Zdog.Ellipse({ addTo: ph, width: l === 0 ? 8 : 10 + l * 2, height: l === 0 ? 12 : 14 + l * 3, fill: true, color: l === 0 ? color.roseRed : color.rosePink, stroke: 0, rotate: { z: a } });
        }
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 8, fill: true, stroke: 0, color: '#c0243a', translate: { z: 1 } });
    }

    function createLavender(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (150 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen }, { x: 0, y: 0 }], stroke: 6, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 8; i++) {
        const ph = new Zdog.Anchor({ addTo: hd, translate: { x: 0, y: -i * 10 }, rotate: { z: i % 2 === 0 ? 0.3 : -0.3 } });
        new Zdog.Ellipse({ addTo: ph, width: 9, height: 14, fill: true, color: i < 3 ? color.lavPurple : color.lavLight, stroke: 0, translate: { x: 6 } });
        new Zdog.Ellipse({ addTo: ph, width: 9, height: 14, fill: true, color: i < 3 ? color.lavPurple : color.lavLight, stroke: 0, translate: { x: -6 } });
      }
    }

    function createSunflower(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (160 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen }, { x: 0, y: 0 }], stroke: 9, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 18; i++) {
        const ph = new Zdog.Anchor({ addTo: hd, rotate: { z: (TAU / 18) * i } });
        new Zdog.Ellipse({ addTo: ph, width: 13, height: 36, fill: true, color: color.sunflowerY, stroke: 0, translate: { y: -24 } });
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 22, fill: true, stroke: 5, color: color.sunflowerC, translate: { z: 2 } });
    }

    function createTulip(parent, x, y, z, scale, stemLen, rotX, rotY, col) {
      col = col || color.tulipPink;
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (130 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen }, { x: 0, y: 0 }], stroke: 8, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 6; i++) {
        const a = (TAU / 6) * i;
        new Zdog.Ellipse({ addTo: hd, width: 18, height: 36, fill: true, color: col, stroke: 0, translate: { x: Math.cos(a) * 10, z: Math.sin(a) * 10 }, rotate: { y: a, x: -0.3 } });
      }
    }

    const honeycombGroup = new Zdog.Anchor({ addTo: fgGroup, translate: { x: -310, y: -180, z: -60 } });
    new Zdog.Shape({ addTo: honeycombGroup, path: [{ x: 0, y: -80 }, { x: 0, y: -30 }], stroke: 5, color: color.trunkBrown });
    new Zdog.Shape({ addTo: honeycombGroup, stroke: 14, color: '#E8A800', translate: { x: 12, y: 46 } });

    const hexCoords = [
      { x: 0,    y: 0   },
      { x: 0,    y: -38 },
      { x: -33,  y: -19 },
      { x:  33,  y: -19 },
      { x: -33,  y:  19 },
      { x:  33,  y:  19 },
      { x: 0,    y:  38 }
    ];
    hexCoords.forEach((pos, i) => {
      new Zdog.Polygon({
        addTo: honeycombGroup,
        sides: 6,
        radius: 17,
        fill: true,
        stroke: 9,
        color: i % 2 === 0 ? color.honeyGold : '#EDAB20',
        translate: { x: pos.x, y: pos.y, z: i % 2 === 0 ? 5 : 0 },
        rotate: { z: TAU / 12 }
      });
      new Zdog.Polygon({
        addTo: honeycombGroup,
        sides: 6,
        radius: 10,
        fill: true,
        stroke: 0,
        color: '#D4920A',
        translate: { x: pos.x, y: pos.y, z: i % 2 === 0 ? 8 : 3 },
        rotate: { z: TAU / 12 }
      });
    });

    const flowers = [
      { f: 'd', x: -750, y: 140, z:  20, s: 4.8, len: 120, rx: -TAU/4,   ry:  0.3,  c: color.daisyWhite },
      { f: 't', x: -680, y: 155, z:  60, s: 5.0, len: 115, rx: -TAU/4.2, ry: -0.2,  c: color.tulipPink },
      { f: 'r', x: -620, y: 130, z: -40, s: 4.6, len: 110, rx: -TAU/3.8, ry:  0.1  },
      { f: 's', x: -580, y: 165, z:  80, s: 4.5, len: 105, rx: -TAU/3.6, ry: -0.3  },
      { f: 'l', x: -530, y: 145, z: 100, s: 5.2, len: 170, rx: -TAU/5,   ry: -0.5  },
      { f: 'd', x: -480, y: 150, z:  30, s: 4.7, len: 125, rx: -TAU/4,   ry:  0.2,  c: '#FFE4F0' },
      { f: 't', x: -420, y: 135, z: -80, s: 5.3, len: 120, rx: -TAU/4.3, ry:  0.4,  c: color.tulipOrange },
      { f: 'r', x: -360, y: 140, z:  50, s: 4.8, len: 115, rx: -TAU/4.4, ry: -0.2  },
      { f: 'd', x: -310, y: 125, z: -100, s: 4.9, len: 130, rx: -TAU/3.8, ry:  0.15, c: '#fff0fb' },
      { f: 's', x: -260, y: 155, z:  70, s: 4.4, len: 110, rx: -TAU/3.5, ry:  0.25 },
      { f: 'l', x: -200, y: 140, z:  45, s: 5.1, len: 165, rx: -TAU/4.8, ry: -0.35 },
      { f: 't', x: -140, y: 160, z: 120, s: 5.0, len: 100, rx: -TAU/4.5, ry:  0.3,  c: '#ff85a2' },
      { f: 'd', x:  -80, y: 145, z:  80, s: 4.6, len: 115, rx: -TAU/4,   ry:  0.1,  c: color.daisyWhite },
      { f: 'r', x:  -20, y: 130, z: -50, s: 4.7, len: 120, rx: -TAU/3.9, ry: -0.1  },
      { f: 's', x:   60, y: 155, z:  90, s: 4.5, len: 105, rx: -TAU/3.7, ry:  0.2  },
      { f: 'd', x:  140, y: 140, z:  40, s: 5.0, len: 125, rx: -TAU/4,   ry: -0.25, c: '#FFE4F0' },
      { f: 't', x:  210, y: 150, z: 110, s: 5.2, len: 115, rx: -TAU/4.2, ry:  0.35, c: color.tulipPink },
      { f: 'l', x:  280, y: 135, z:  60, s: 4.8, len: 160, rx: -TAU/5,   ry: -0.4  },
      { f: 'd', x:  350, y: 160, z:  75, s: 4.9, len: 120, rx: -TAU/4,   ry:  0.2,  c: '#fff0fb' },
      { f: 'r', x:  410, y: 140, z: -70, s: 4.6, len: 125, rx: -TAU/3.8, ry:  0.15 },
      { f: 's', x:  470, y: 155, z:  85, s: 4.7, len: 110, rx: -TAU/3.5, ry: -0.2  },
      { f: 't', x:  540, y: 130, z:  30, s: 5.1, len: 100, rx: -TAU/4.5, ry:  0.4,  c: color.tulipOrange },
      { f: 'l', x:  600, y: 145, z: 120, s: 5.0, len: 175, rx: -TAU/4.8, ry: -0.3  },
      { f: 'd', x:  660, y: 150, z:  50, s: 4.8, len: 130, rx: -TAU/4,   ry:  0.25, c: '#FFE4F0' },
      { f: 'r', x:  720, y: 140, z: -90, s: 4.9, len: 115, rx: -TAU/3.9, ry: -0.15 },
      { f: 's', x:  780, y: 165, z:  95, s: 4.5, len: 105, rx: -TAU/3.6, ry:  0.1  },
      { f: 't', x:  840, y: 135, z:  40, s: 5.3, len: 120, rx: -TAU/4.3, ry:  0.3,  c: color.tulipPink },
      { f: 'd', x:  900, y: 155, z: 110, s: 4.7, len: 125, rx: -TAU/4.1, ry: -0.2,  c: '#fff0fb' },
      { f: 'l', x:  950, y: 140, z:  60, s: 5.2, len: 170, rx: -TAU/5,   ry: -0.4  }
    ];

    flowers.forEach(d => {
      if      (d.f === 'd') createDaisy(fgGroup,    d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
      else if (d.f === 'r') createRose(fgGroup,     d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 'l') createLavender(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 's') createSunflower(fgGroup,d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 't') createTulip(fgGroup,    d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
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
