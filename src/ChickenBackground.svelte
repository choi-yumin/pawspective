<script>
//@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';

  let canvasRef;

  onMount(() => {
    const TAU = Zdog.TAU;

    const color = {
      body: '#FFF1E6', headBase: '#FFF5E8', eye: '#FFFFFF', pupil: '#111111',
      comb: '#FF3B5C', tail: '#F8D7C7', leg: '#FAA353',
      sky: '#FFB7D5', skyLight: '#FFE1F0',
      land: '#7ab535', landLight: '#9fd95c', darkGreen: '#3A5C18',
      trunkBrown: '#8B6340', cloudPink: '#FFE4F0', cloudWhite: '#FFF7FB',
      blossomPink: '#FFACD2', appleRed: '#FF4B4B',
      leafGreen: '#5DA020', daisyWhite: '#FFFFFF', daisyYellow: '#F9D342',
      roseRed: '#F04A6F', rosePink: '#FF8FAE',
      lavPurple: '#C07EE0', lavLight: '#E2B8F5',
      sunflowerY: '#FFB800', sunflowerC: '#7A3B10',
      tulipPink: '#FF85A2', tulipOrange: '#FF6B35'
    };

    const scene = new Zdog.Illustration({ element: canvasRef, dragRotate: false, resize: 'window' });
    const staticGroup = new Zdog.Anchor({ addTo: scene });

    new Zdog.Shape({ addTo: staticGroup, path: [{ x: -3000, y: -1500 }, { x: 3000, y: -1500 }, { x: 3000, y: 180 }, { x: -3000, y: 180 }], stroke: 0, fill: true, color: color.sky, translate: { z: -1500 } });
    new Zdog.Shape({ addTo: staticGroup, path: [{ x: -3000, y: 180 }, { x: 3000, y: 180 }, { x: 3000, y: 1600 }, { x: -3000, y: 1600 }], stroke: 0, fill: true, color: color.skyLight, translate: { z: -1500 } });

    function addHill(x, w, col, z, amp = 220) {
      const pts = [{ x: x - w / 2, y: 520 }];
      for (let i = 0; i <= 24; i++) {
        const t = i / 24;
        pts.push({ x: x - w / 2 + t * w, y: 520 - Math.sin(t * Math.PI) * amp });
      }
      pts.push({ x: x + w / 2, y: 520 });
      new Zdog.Shape({ addTo: staticGroup, path: pts, stroke: 0, fill: true, color: col, translate: { z } });
    }

    addHill(-900, 1600, color.darkGreen, -1200, 260);
    addHill(800,  1500, '#4A7020',       -1100, 220);
    addHill(-300, 1300, '#5A8A28',       -1000, 180);
    addHill(400,  1000, '#6BA330',       -900,  140);


    // ─── Land and Ground ──────────────────────────────────────────────
    // Pushed land further away by changing z from -700/-690 to -1200/-1190
    new Zdog.Shape({ addTo: staticGroup, path: [{ x: -3000, y: 340 }, { x: 3000, y: 340 }, { x: 3000, y: 1800 }, { x: -3000, y: 1800 }], stroke: 0, fill: true, color: color.land, translate: { y: -200, z: -1200 } });
    new Zdog.Shape({ addTo: staticGroup, path: [{ x: -3000, y: 340 }, { x: 3000, y: 340 }, { x: 3000, y: 400 }, { x: -3000, y: 400 }], stroke: 0, fill: true, color: color.landLight, translate: { y: -200, z: -1190 } });

    // ─── Trees (Scaled up) ──────────────────────────────────────────
    [
      // Background trees directly behind chicken
      { x: -100, y: 150, z: -400, s: 3.0, type: 'apples'   }, 
      { x:  150, y: 150, z: -350, s: 2.8, type: 'blossoms' },
      
      // Original landscape trees
      { x: -750, y: 210, z: -850, s: 4.0, type: 'apples'   }, 
      { x: -500, y: 240, z: -800, s: 3.5, type: 'blossoms' }, 
      { x:  650, y: 220, z: -860, s: 4.0, type: 'apples'   }, 
      { x:  850, y: 260, z: -780, s: 3.5, type: 'blossoms' }
    ].forEach(p => {
      // ... (your existing drawing logic)
      const t = new Zdog.Anchor({ addTo: staticGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: t, path: [{ y: 0 }, { y: -120 }], stroke: 90, color: color.trunkBrown });
      const fGroup = new Zdog.Anchor({ addTo: t, translate: { y: -150 } });
      new Zdog.Shape({ addTo: fGroup, stroke: 700, color: '#5DA020' });
      new Zdog.Shape({ addTo: fGroup, stroke: 300, color: '#7CBF37', translate: { x: -40, y: -40, z: 15 } });
      new Zdog.Shape({ addTo: fGroup, stroke: 200, color: '#4A8516', translate: { x: 45,  y: 25,  z: -15 } });
    });
    
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

    const flowerGroup = new Zdog.Anchor({ addTo: staticGroup, translate: { x: 0, y: 0, z: 0 } });
    createSunflower(flowerGroup, -620, 400, 10,  5.5, 150, -TAU/4.5, 0.2);
    createDaisy(flowerGroup,     -480, 390, 40,  5.0, 130, -TAU/4,   -0.2, color.daisyWhite);
    createTulip(flowerGroup,     -560, 370, -30, 6.0, 110, -TAU/4,   0.4, color.tulipPink);
    createRose(flowerGroup,      -420, 380, 70,  5.0, 120, -TAU/4.2, -0.3);
    createLavender(flowerGroup,  -680, 410, -10, 5.0, 160, -TAU/5,   0.3);
    createTulip(flowerGroup,      480, 375, -20, 6.0, 110, -TAU/4,   -0.3, color.tulipOrange);
    createDaisy(flowerGroup,      560, 395, 50,  5.0, 135, -TAU/4,   0.25, '#FFE4F0');
    createSunflower(flowerGroup,  420, 405, 10,  5.5, 150, -TAU/4.5, -0.15);
    createRose(flowerGroup,       650, 385, -40, 5.0, 125, -TAU/4.2, 0.3);
    createLavender(flowerGroup,   380, 415, 70,  5.0, 160, -TAU/5,   -0.25);

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
