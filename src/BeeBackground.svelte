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

    // Replaced the frontal orange tulip layout at x: -350 with a normal radial white Daisy.
    const flowers = [
      { f: 's', x: -600, y: 120, z: -150, s: 6.5, len: 130, rx: -TAU/3.7, ry: -0.1  }, 
      { f: 'd', x: -350, y: 125, z:   83, s: 10, len: 110, rx: -TAU/4.1, ry:  0.2,  c: color.tulipOrange}, // Replaced orange tulip
      { f: 'r', x: -120, y:  95, z: -280, s: 5.0, len: 140, rx: -TAU/3.9, ry:  0.15 }, 
      { f: 'd', x:  100, y: 135, z:   20, s: 8.8, len: 125, rx: -TAU/4,   ry: -0.2,  c: '#FFFFFF' }, 
      { f: 's', x:  320, y: 100, z: -320, s: 4.8, len: 115, rx: -TAU/3.6, ry:  0.1  }, 
      { f: 'd', x:  520, y: 145, z:   90, s: 9.5, len: 135, rx: -TAU/4.2, ry: -0.25, c: '#FFE4F0' }, 
      { f: 't', x:  700, y: 110, z: -180, s: 5.8, len: 100, rx: -TAU/4.4, ry:  0.2,  c: color.tulipPink }   
    ];

    flowers.forEach(d => {
      if      (d.f === 'd') createDaisy(fgGroup,    d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
      else if (d.f === 'r') createRose(fgGroup,     d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
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