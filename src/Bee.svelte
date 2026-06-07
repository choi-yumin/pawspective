<script>
  //@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';

  let canvasRef;

  onMount(() => {
    const TAU = Zdog.TAU;

    const color = {
      beeYellow:'#FCD116', beeBlack:'#3D2620', beeWhite:'#FFFDF0', beeCheek:'#F26B50',
      leafGreen:'#5DA020', darkGreen:'#3A5C18', land:'#7ab535', landLight:'#9fd95c',
      trunkBrown:'#8B6340', cloudPink:'#FFE4F0', cloudWhite:'#FFF7FB',
      daisyWhite:'#FFFFFF', daisyYellow:'#F9D342',
      roseRed:'#F04A6F', rosePink:'#FF8FAE',
      lavPurple:'#C07EE0', lavLight:'#E2B8F5',
      sunflowerYellow:'#FFB800', sunflowerCenter:'#7A3B10',
      tulipPink:'#FF85A2', tulipOrange:'#FF6B35'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: {
        x: -0.32,
        y: -0.18,
        z: 0,
      },
    });

    const envGroup = new Zdog.Anchor({ addTo: scene });

    new Zdog.Shape({
      addTo: envGroup,
      path: [{x:-2000,y:-1000},{x:2000,y:-1000},{x:2000,y:0},{x:-2000,y:0}],
      stroke: 0, fill: true, color: '#FFB7D5', translate: { z: -800 }
    });

    new Zdog.Shape({
      addTo: envGroup,
      path: [{x:-2000,y:0},{x:2000,y:0},{x:2000,y:600},{x:-2000,y:600}],
      stroke: 0, fill: true, color: '#FFE1F0', translate: { z: -800 }
    });

    function addHill(x, w, col, z, amp = 160) {
      const pts = [{x: x - w / 2, y: 300}];
      for (let i = 0; i <= 24; i++) {
        const t = i / 24;
        pts.push({ x: x - w/2 + t*w, y: 300 - Math.sin(t * Math.PI) * amp });
      }
      pts.push({x: x + w / 2, y: 300});
      new Zdog.Shape({ addTo: envGroup, path: pts, stroke: 0, fill: true, color: col, translate: { z } });
    }

    addHill(-600, 900, color.darkGreen, -700, 160);
    addHill(500,  800, '#4a7020',       -700, 160);
    addHill(-200, 700, '#5a8a28',       -680, 150);

    new Zdog.Shape({
      addTo: envGroup,
      path: [{x:-2000,y:220},{x:2000,y:220},{x:2000,y:800},{x:-2000,y:800}],
      stroke: 0, fill: true, color: color.land, translate: { z: -500 }
    });

    new Zdog.Shape({
      addTo: envGroup,
      path: [{x:-2000,y:220},{x:2000,y:220},{x:2000,y:260},{x:-2000,y:260}],
      stroke: 0, fill: true, color: color.landLight, translate: { z: -490 }
    });

    [
      {x:-420,y:-280,z:-420,s:1.2},
      {x:-160,y:-310,z:-510,s:0.9},
      {x: 360,y:-220,z:-380,s:1.4},
      {x:  80,y:-290,z:-460,s:1.0}
    ].forEach(p => {
      let c = new Zdog.Anchor({
        addTo: envGroup,
        translate: { x: p.x, y: p.y, z: p.z },
        scale: p.s
      });
      new Zdog.Shape({ addTo: c, stroke: 95, color: color.cloudPink });
      new Zdog.Shape({ addTo: c, stroke: 72, color: color.cloudWhite, translate: { x: -55, y: 10 } });
      new Zdog.Shape({ addTo: c, stroke: 78, color: color.cloudPink, translate: { x: 55, y: 5 } });
      new Zdog.Shape({ addTo: c, stroke: 62, color: color.cloudWhite, translate: { x: -95, y: 15 } });
      new Zdog.Shape({ addTo: c, stroke: 58, color: color.cloudPink, translate: { x: 100, y: 12 } });
    });

    [
      {x:-460,y:140,z:-320,s:1.0},
      {x:-310,y:160,z:-260,s:0.8},
      {x: 390,y:150,z:-370,s:1.2},
      {x: 520,y:170,z:-220,s:0.85}
    ].forEach(p => {
      let t = new Zdog.Anchor({
        addTo: envGroup,
        translate: { x: p.x, y: p.y, z: p.z },
        scale: p.s
      });
      new Zdog.Shape({ addTo: t, path: [{y:0},{y:-90}], stroke: 18, color: color.trunkBrown });
      new Zdog.Shape({ addTo: t, stroke: 130, color: color.darkGreen, translate: { y: -130 } });
      new Zdog.Shape({ addTo: t, stroke: 100, color: color.leafGreen, translate: { y: -175, x: -20 } });
      new Zdog.Shape({ addTo: t, stroke: 90, color: '#7fcf38', translate: { y: -185, x: 20 } });
    });

    const fgGroup = new Zdog.Anchor({
      addTo: scene,
      translate: { x: 0, y: 50, z: 120 }
    });

    function createDaisy(parent, x, y, z, scale, stemLen, rotX, rotY, petalCol) {
      petalCol = petalCol || color.daisyWhite;
      const fg = new Zdog.Anchor({
        addTo: parent,
        translate: { x, y: y + (120 - stemLen), z },
        scale
      });

      new Zdog.Shape({
        addTo: fg,
        path: [{x:0,y:stemLen,z:0},{x:0,y:0,z:0}],
        stroke: 7,
        color: color.leafGreen
      });

      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 13; i++) {
        let ph = new Zdog.Anchor({ addTo: hd, rotate: { z: (TAU / 13) * i } });
        new Zdog.Ellipse({
          addTo: ph,
          width: 11,
          height: 30,
          fill: true,
          color: petalCol,
          stroke: 0,
          translate: { y: -18 }
        });
      }
      new Zdog.Ellipse({
        addTo: hd,
        diameter: 15,
        fill: true,
        stroke: 4,
        color: color.daisyYellow,
        translate: { z: 1.5 }
      });
    }

    function createRose(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({
        addTo: parent,
        translate: { x, y: y + (140 - stemLen), z },
        scale
      });

      new Zdog.Shape({
        addTo: fg,
        path: [{x:0,y:stemLen},{x:0,y:0}],
        stroke: 7,
        color: color.leafGreen
      });

      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });

      for (let l = 0; l < 3; l++) {
        const pc = 6 + l * 2;
        const r = 10 + l * 9;
        for (let i = 0; i < pc; i++) {
          const a = (TAU / pc) * i + l * 0.3;
          const ph = new Zdog.Anchor({
            addTo: hd,
            translate: { x: Math.cos(a) * r, y: Math.sin(a) * r }
          });
          new Zdog.Ellipse({
            addTo: ph,
            width: l === 0 ? 8 : 10 + l * 2,
            height: l === 0 ? 12 : 14 + l * 3,
            fill: true,
            color: l === 0 ? color.roseRed : color.rosePink,
            stroke: 0,
            rotate: { z: a }
          });
        }
      }

      new Zdog.Ellipse({
        addTo: hd,
        diameter: 8,
        fill: true,
        stroke: 0,
        color: '#c0243a',
        translate: { z: 1 }
      });
    }

    function createLavender(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({
        addTo: parent,
        translate: { x, y: y + (150 - stemLen), z },
        scale
      });

      new Zdog.Shape({
        addTo: fg,
        path: [{x:0,y:stemLen},{x:0,y:0}],
        stroke: 6,
        color: color.leafGreen
      });

      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });

      for (let i = 0; i < 8; i++) {
        const ph = new Zdog.Anchor({
          addTo: hd,
          translate: { x: 0, y: -i * 10 },
          rotate: { z: i % 2 === 0 ? 0.3 : -0.3 }
        });
        new Zdog.Ellipse({
          addTo: ph,
          width: 9,
          height: 14,
          fill: true,
          color: i < 3 ? color.lavPurple : color.lavLight,
          stroke: 0,
          translate: { x: 6 }
        });
        new Zdog.Ellipse({
          addTo: ph,
          width: 9,
          height: 14,
          fill: true,
          color: i < 3 ? color.lavPurple : color.lavLight,
          stroke: 0,
          translate: { x: -6 }
        });
      }
    }

    function createSunflower(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({
        addTo: parent,
        translate: { x, y: y + (160 - stemLen), z },
        scale
      });

      new Zdog.Shape({
        addTo: fg,
        path: [{x:0,y:stemLen},{x:0,y:0}],
        stroke: 9,
        color: color.leafGreen
      });

      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });

      for (let i = 0; i < 18; i++) {
        let ph = new Zdog.Anchor({ addTo: hd, rotate: { z: (TAU / 18) * i } });
        new Zdog.Ellipse({
          addTo: ph,
          width: 13,
          height: 36,
          fill: true,
          color: color.sunflowerYellow,
          stroke: 0,
          translate: { y: -24 }
        });
      }

      new Zdog.Ellipse({
        addTo: hd,
        diameter: 22,
        fill: true,
        stroke: 5,
        color: color.sunflowerCenter,
        translate: { z: 2 }
      });
    }

    function createTulip(parent, x, y, z, scale, stemLen, rotX, rotY, col) {
      col = col || color.tulipPink;
      const fg = new Zdog.Anchor({
        addTo: parent,
        translate: { x, y: y + (130 - stemLen), z },
        scale
      });

      new Zdog.Shape({
        addTo: fg,
        path: [{x:0,y:stemLen},{x:0,y:0}],
        stroke: 8,
        color: color.leafGreen
      });

      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });

      for (let i = 0; i < 6; i++) {
        const a = (TAU / 6) * i;
        new Zdog.Ellipse({
          addTo: hd,
          width: 18,
          height: 36,
          fill: true,
          color: col,
          stroke: 0,
          translate: { x: Math.cos(a) * 10, z: Math.sin(a) * 10 },
          rotate: { y: a, x: -0.3 }
        });
      }
    }

    createDaisy(fgGroup, 0, 100, 10, 5, 120, -TAU / 4, 0, color.daisyWhite);

    const fd = [
      {f:'r',x:-55, y:135,z:-80, s:5.5, len:160, rx:-TAU/5, ry:0.3},
      {f:'s',x:115,y:210,z:120, s:4.2, len:110, rx:-TAU/3.5, ry:-0.4},
      {f:'d',x:-195,y:115,z:-180,s:5,   len:150, rx:-TAU/4.5, ry:0.1, c:'#ffe4ef'},
      {f:'t',x:215,y:165,z:-100,s:7,   len:100, rx:-TAU/4, ry:-0.2, c:color.tulipPink},
      {f:'l',x:-110,y:125,z:60, s:4.8, len:170, rx:-TAU/5, ry:-0.5},
      {f:'d',x:90,y:155,z:-140,s:4.5, len:130, rx:-TAU/3.8, ry:0.2, c:'#ffe9f5'},
      {f:'t',x:-280,y:160,z:90, s:5.5, len:145, rx:-TAU/4.2, ry:0.5, c:color.tulipOrange},
      {f:'d',x:280,y:145,z:50, s:5,   len:155, rx:-TAU/4, ry:-0.3, c:color.daisyWhite},
      {f:'r',x:350,y:185,z:-110,s:4.8, len:115, rx:-TAU/3.2, ry:0.1},
      {f:'s',x:-340,y:135,z:-30, s:4.5, len:180, rx:-TAU/4.8, ry:-0.2},
      {f:'l',x:160,y:195,z:100,s:5,   len:125, rx:-TAU/4, ry:0.4},
      {f:'t',x:-160,y:150,z:140,s:5.5, len:165, rx:-TAU/5.2, ry:-0.1, c:'#ff85a2'},
      {f:'d',x:45,y:205,z:170,s:4.5, len:95,  rx:-TAU/3.4, ry:0.3, c:'#fff0fb'},
      {f:'r',x:-260,y:140,z:-130,s:4.2, len:175, rx:-TAU/4.5, ry:-0.4},
      {f:'d',x:420,y:170,z:0, s:4,   len:140, rx:-TAU/4, ry:0.2, c:'#fff'},
      {f:'s',x:-450,y:210,z:30, s:4, len:105, rx:-TAU/3.6, ry:-0.3}
    ];

    fd.forEach(d => {
      if      (d.f === 'd') createDaisy(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
      else if (d.f === 'r') createRose(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 'l') createLavender(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 's') createSunflower(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 't') createTulip(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
    });

    // Bee
    let BEE_CONT = new Zdog.Anchor({
      addTo: fgGroup,
      translate: { x: 0, y: -130, z: 0 },
      scale: 1.45
    });

    // pivot anchor located at the body's center so rotations feel natural
    let BEE_PIVOT = new Zdog.Anchor({ addTo: BEE_CONT, translate: { x: 0, y: 32, z: -35 }, rotate: { y: 0 } });
    // BEE is translated so its children keep the same world positions
    let BEE = new Zdog.Anchor({ addTo: BEE_PIVOT, translate: { x: 0, y: -32, z: 35 } });

    let beeHead = new Zdog.Shape({ addTo: BEE, stroke: 135, color: color.beeYellow });
    new Zdog.Shape({ addTo: beeHead, stroke: 10, color: color.beeBlack, translate: { x: -22, y: 4, z: 36 } });
    new Zdog.Shape({ addTo: beeHead, stroke: 10, color: color.beeBlack, translate: { x: 14, y: 4, z: 40 } });
    new Zdog.Shape({
      addTo: beeHead,
      stroke: 3.5,
      color: color.beeBlack,
      closed: false,
      path: [{x:-4,y:10,z:42},{bezier:[{x:-2,y:13,z:42},{x:2,y:13,z:42},{x:4,y:10,z:42}]}]
    });

    let rc = new Zdog.Shape({
      addTo: beeHead,
      stroke: 14,
      color: color.beeCheek,
      translate: { x: -32, y: 14, z: 24 }
    });
    rc.copy({ translate: { x: 24, y: 14, z: 30 } });

    let antlerAnchor = new Zdog.Anchor({ addTo: beeHead, translate: { y: -44, z: 10 } });
    new Zdog.Shape({ addTo: antlerAnchor, path: [{y:0,x:-10},{y:-22,x:-24,z:8}], stroke: 4.5, color: color.beeBlack });
    new Zdog.Shape({ addTo: antlerAnchor, path: [{y:0,x:10},{y:-22,x:-4,z:12}], stroke: 4.5, color: color.beeBlack });

    let leftArm = new Zdog.Anchor({ addTo: BEE, translate: { x: -18, y: 45, z: 32 } });
    new Zdog.Shape({ addTo: leftArm, path: [{y:0},{y:22}], stroke: 7, color: color.beeBlack });

    let rightArm = new Zdog.Anchor({ addTo: BEE, translate: { x: 18, y: 45, z: 32 } });
    new Zdog.Shape({ addTo: rightArm, path: [{y:0},{y:22}], stroke: 7, color: color.beeBlack });

    let bodyAnchor = new Zdog.Anchor({ addTo: BEE, translate: { y: 32, z: -35 } });
    let p1 = new Zdog.Shape({ addTo: bodyAnchor, stroke: 140, color: color.beeYellow });
    p1.copy({ addTo: bodyAnchor, stroke: 162, color: color.beeBlack, translate: { z: -32 } });
    p1.copy({ addTo: bodyAnchor, stroke: 168, color: color.beeYellow, translate: { z: -64 } });
    p1.copy({ addTo: bodyAnchor, stroke: 156, color: color.beeBlack, translate: { z: -96 } });
    new Zdog.Shape({ addTo: bodyAnchor, stroke: 108, color: color.beeBlack, translate: { z: -123 } });

    let rightWing = new Zdog.Anchor({ addTo: bodyAnchor, translate: { z: -43, y: -65, x: 29 } });
    let leftWing  = new Zdog.Anchor({ addTo: bodyAnchor, translate: { z: -43, y: -65, x: -29 } });

    new Zdog.Ellipse({
      addTo: rightWing,
      width: 80,
      height: 160,
      color: color.beeWhite,
      fill: true,
      rotate: { x: TAU / 5, z: TAU / 5 },
      translate: { x: 65 },
      stroke: 0
    });

    new Zdog.Ellipse({
      addTo: leftWing,
      width: 80,
      height: 160,
      color: color.beeWhite,
      fill: true,
      rotate: { x: TAU / 5, z: -TAU / 5 },
      translate: { x: -65 },
      stroke: 0
    });

    let frame = 0;
    let isRunning = true;
    let isGathering = false;

    function render() {
      if (!isRunning) return;
      frame++;

      // faster wing flapping frequency
      rightWing.rotate.z = -TAU / 6 + (TAU / 10) * Math.sin(frame / 0.9);
      leftWing.rotate.z = TAU / 6 - (TAU / 10) * Math.sin(frame / 0.9);
      antlerAnchor.rotate.x = (TAU / 40) * Math.sin(frame / 10);

      if (!isGathering) {
        BEE_CONT.translate.y = -130 + Math.sin(frame / 14) * 4;
        BEE_CONT.translate.x = Math.cos(frame / 28) * 3;
      }

      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }

    render();

    function gatherPollen() {
      if (isGathering) return;
      isGathering = true;

      const tl = gsap.timeline({ onComplete: () => { isGathering = false; } });

      tl.to(BEE_CONT.translate, { duration: 0.55, x: 0, y: 80, z: 5, ease: 'power2.inOut' });
      // animate the pivot so the whole bee rotates around its body center
      // avoid roll (z) to prevent skew — only pitch (x) and yaw (y)
      tl.to(BEE_PIVOT.rotate, { duration: 0.25, x: 0.5, y: 0.4, ease: 'power2.out' }, '-=0.2');
      tl.to(leftArm.rotate, { duration: 0.2, z: 0.2, x: 0.1, ease: 'back.out(1.5)' }, '-=0.1');
      tl.to(rightArm.rotate, { duration: 0.2, z: -0.2, x: 0.1, ease: 'back.out(1.5)' }, '<');
      tl.to(BEE_CONT.translate, { duration: 0.12, y: 90, ease: 'power1.inOut' });
      tl.to(BEE_CONT.translate, { duration: 0.12, y: 75, ease: 'power1.inOut' });
      tl.to(leftArm.rotate, { duration: 0.2, z: 0, x: 0, ease: 'power2.in' });
      tl.to(rightArm.rotate, { duration: 0.2, z: 0, x: 0, ease: 'power2.in' }, '<');
      tl.to(BEE_CONT.translate, { duration: 0.6, x: 0, y: -130, z: 0, ease: 'power2.inOut' });
      // return to a neutral rotation (no roll)
      tl.to(BEE_PIVOT.rotate, { duration: 0.4, x: 0.4, y: 0.4, ease: 'power2.inOut' }, '-=0.5');
      tl.set(BEE_PIVOT.rotate, { z: 0 });
    }

    let isDragging = false;
    let lastX = 0;
    let lastY = 0;
    const sensitivity = 0.008;

    function handleCanvasClick(event) {
      const rect = canvasRef.getBoundingClientRect();
      const clickX = event.clientX - rect.left - rect.width / 2;
      const clickY = event.clientY - rect.top - rect.height / 2;

      const targetX = BEE_CONT.translate.x;
      const targetY = BEE_CONT.translate.y + 50;

      const dx = clickX - targetX;
      const dy = clickY - targetY;
      const distance = Math.sqrt(dx * dx + dy * dy);

      if (distance < 120) {
        gatherPollen();
      }
    }

    function onPointerDown(e) {
      isDragging = true;
      lastX = e.clientX;
      lastY = e.clientY;
      try { canvasRef.setPointerCapture(e.pointerId); } catch (err) {}
    }

    function onPointerMove(e) {
      if (!isDragging) return;
      const dx = e.clientX - lastX;
      const dy = e.clientY - lastY;
      lastX = e.clientX;
      lastY = e.clientY;
      if (typeof BEE_PIVOT !== 'undefined' && BEE_PIVOT) {
        BEE_PIVOT.rotate.y += dx * sensitivity;
        BEE_PIVOT.rotate.x += dy * sensitivity;
        BEE_PIVOT.rotate.x = Math.max(-Zdog.TAU/8, Math.min(Zdog.TAU/8, BEE_PIVOT.rotate.x));
        scene.updateRenderGraph();
      }
    }

    function onPointerUp(e) {
      isDragging = false;
      try { canvasRef.releasePointerCapture(e.pointerId); } catch (err) {}
    }

    canvasRef.addEventListener('click', handleCanvasClick);
    canvasRef.addEventListener('pointerdown', onPointerDown);
    canvasRef.addEventListener('pointermove', onPointerMove);
    window.addEventListener('pointerup', onPointerUp);

    return () => {
      isRunning = false;
      canvasRef.removeEventListener('click', handleCanvasClick);
      canvasRef.removeEventListener('pointerdown', onPointerDown);
      canvasRef.removeEventListener('pointermove', onPointerMove);
      window.removeEventListener('pointerup', onPointerUp);
    };
  });
</script>

<div class="credit">
  <p>Made by MakMin</p>
</div>

<main>
  <canvas bind:this={canvasRef} class="scene"></canvas>
</main>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap");

  :global(body) {
    margin: 0;
    padding: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100vh;
    background: #FFB7D5;
    overflow: hidden;
    position: relative;
    font-family: "Inter", sans-serif;
  }

  main {
    width: 100vw;
    height: 100vh;
    display: grid;
    place-items: center;
  }

  canvas {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    display: block;
    z-index: 0;
    touch-action: none;
  }
</style>