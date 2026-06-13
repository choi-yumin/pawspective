<script>
  //@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';

  let canvasRef;
  let isDeep = false;
  let isAnimating = false;
  let hintText = 'click the blobfish to dive';

  let fishState = { pinkAlpha: 1, darkAlpha: 0 };

  let scene;
  let bgAnchor;
  let blobMasterAnchor, handsomeMasterAnchor;
  let leftPecContainer, rightPecContainer;
  let leftWingContainer, rightWingContainer;
  let handsomeTailFin, blobTailFin;
  let eyeL, eyeR;

  let pinkFadeShapes = [];
  let darkFadeShapes = [];

  const color = {
    skyTop: '#86D7FF',
    skyMid: '#B9EEF7',
    skyHaze: '#EAFBFF',
    pondShallow: '#82E3D3',
    pondDeep: '#17546E',
    pondDeepDark: '#0B3042',
    pondMist: '#DDFBF7',
    shore: '#90D855',
    shoreLight: '#C0EA72',
    reed: '#628D2A',
    trunk: '#8B6442',
    leafDark: '#2C5D24',
    leafLight: '#78C93A',
    petal: '#FFFFFF',
    petalPink: '#FF91B5',
    petalOrange: '#FFAA49',
    center: '#F8C53A',
    beeYellow: '#FFD11A',
    beeBlack: '#2A1B16',
    beeWhite: '#FFFDF6',
    beeCheek: '#FF7B6A',
    blobSkin: '#F4B8C2',
    blobNose: '#F3AEBB',
    blobFin: '#C97A8D',
    blobMouth: '#6B3A45',
    blobTailBody: '#D88A9D',
    blobDeepSea: '#6C8EA4',
    blobDeepSeaDark: '#4A6578',
    blobMouthDeep: '#334856'
  };

  function hexToRgb(hex) {
    return {
      r: parseInt(hex.slice(1, 3), 16),
      g: parseInt(hex.slice(3, 5), 16),
      b: parseInt(hex.slice(5, 7), 16)
    };
  }

  function setupFadeGroup(anchor) {
    const shapes = [];
    function traverse(node) {
      if (node.color && typeof node.color === 'string' && node.color.startsWith('#')) {
        const { r, g, b } = hexToRgb(node.color);
        shapes.push({ node, r, g, b });
      }
      if (node.children) node.children.forEach(traverse);
    }
    traverse(anchor);
    return shapes;
  }

  function makeFlower(parent, x, y, z, kind) {
    const g = new Zdog.Anchor({ addTo: parent, translate: { x, y, z } });
    new Zdog.Shape({ addTo: g, path: [{ y: 55 }, { y: 0 }], stroke: 5, color: '#5C9F2D' });
    const head = new Zdog.Anchor({ addTo: g, translate: { y: 0 } });

    if (kind === 'daisy') {
      for (let i = 0; i < 10; i++) {
        const p = new Zdog.Anchor({ addTo: head, rotate: { z: (Zdog.TAU / 10) * i } });
        new Zdog.Ellipse({ addTo: p, width: 9, height: 25, fill: true, color: color.petal, stroke: 0, translate: { y: -14 } });
      }
      new Zdog.Ellipse({ addTo: head, diameter: 14, fill: true, color: color.center, stroke: 0 });
    }

    if (kind === 'rose') {
      for (let i = 0; i < 6; i++) {
        const p = new Zdog.Anchor({ addTo: head, rotate: { z: (Zdog.TAU / 6) * i } });
        new Zdog.Ellipse({ addTo: p, width: 12, height: 20, fill: true, color: color.petalPink, stroke: 0, translate: { y: -10 } });
      }
      new Zdog.Ellipse({ addTo: head, diameter: 8, fill: true, color: '#C62C52', stroke: 0 });
    }

    if (kind === 'tulip') {
      for (let i = 0; i < 4; i++) {
        const p = new Zdog.Anchor({ addTo: head, rotate: { z: (Zdog.TAU / 4) * i } });
        new Zdog.Ellipse({ addTo: p, width: 12, height: 22, fill: true, color: color.petalOrange, stroke: 0, translate: { y: -12 } });
      }
    }
  }

  function makeReed(parent, x, y, z, h) {
    const r = new Zdog.Anchor({ addTo: parent, translate: { x, y, z } });
    new Zdog.Shape({ addTo: r, path: [{ y: 0 }, { y: -h }], stroke: 4, color: color.reed });
    new Zdog.Shape({ addTo: r, stroke: 10, color: '#7B5A28', translate: { y: -h + 2 } });
  }

  onMount(() => {
    scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.2, y: -0.12, z: 0 }
    });

    bgAnchor = new Zdog.Anchor({ addTo: scene });

    new Zdog.Shape({
      addTo: bgAnchor,
      stroke: 0,
      fill: true,
      color: color.skyTop,
      path: [{ x: -2200, y: -1400 }, { x: 2200, y: -1400 }, { x: 2200, y: 1400 }, { x: -2200, y: 1400 }],
      translate: { z: -1300 }
    });

    new Zdog.Shape({
      addTo: bgAnchor,
      stroke: 0,
      fill: true,
      color: color.skyMid,
      path: [{ x: -2200, y: -220 }, { x: 2200, y: -220 }, { x: 2200, y: 400 }, { x: -2200, y: 400 }],
      translate: { z: -1200 }
    });

    for (let i = 0; i < 10; i++) {
      const cx = -700 + i * 150;
      const cy = -420 + (i % 3) * 28;
      const cloud = new Zdog.Anchor({ addTo: bgAnchor, translate: { x: cx, y: cy, z: -1000 }, scale: 1 + (i % 2) * 0.16 });
      new Zdog.Shape({ addTo: cloud, stroke: 55, color: color.skyHaze });
      new Zdog.Shape({ addTo: cloud, stroke: 35, color: '#FFFFFF', translate: { x: -34 } });
      new Zdog.Shape({ addTo: cloud, stroke: 42, color: color.skyHaze, translate: { x: 32 } });
    }

    const hills = new Zdog.Anchor({ addTo: bgAnchor, translate: { z: -900 } });
    new Zdog.Shape({
      addTo: hills,
      stroke: 0,
      fill: true,
      color: '#3C6D24',
      path: [
        { x: -2200, y: 220 },
        { x: -900, y: 210 },
        { x: -500, y: 120 },
        { x: -90, y: 200 },
        { x: 360, y: 110 },
        { x: 760, y: 210 },
        { x: 2200, y: 220 },
        { x: 2200, y: 1200 },
        { x: -2200, y: 1200 }
      ]
    });

    new Zdog.Shape({
      addTo: hills,
      stroke: 0,
      fill: true,
      color: '#5D942C',
      path: [
        { x: -2200, y: 260 },
        { x: -1200, y: 240 },
        { x: -760, y: 150 },
        { x: -260, y: 240 },
        { x: 180, y: 150 },
        { x: 680, y: 240 },
        { x: 2200, y: 260 },
        { x: 2200, y: 1200 },
        { x: -2200, y: 1200 }
      ]
    });

    new Zdog.Shape({
      addTo: bgAnchor,
      stroke: 0,
      fill: true,
      color: color.pondDeep,
      path: [{ x: -2200, y: 160 }, { x: 2200, y: 160 }, { x: 2200, y: 1200 }, { x: -2200, y: 1200 }],
      translate: { z: -520 }
    });

    const pondSurface = new Zdog.Shape({
      addTo: bgAnchor,
      stroke: 0,
      fill: true,
      color: color.pondShallow,
      path: [{ x: -2200, y: 120 }, { x: 2200, y: 120 }, { x: 2200, y: 250 }, { x: -2200, y: 250 }],
      translate: { z: -510 }
    });

    const pondDeep = new Zdog.Shape({
      addTo: bgAnchor,
      stroke: 0,
      fill: true,
      color: color.pondDeepDark,
      path: [{ x: -2200, y: 250 }, { x: 2200, y: 250 }, { x: 2200, y: 1200 }, { x: -2200, y: 1200 }],
      translate: { z: -500 }
    });

    const pondMist = new Zdog.Shape({
      addTo: bgAnchor,
      stroke: 0,
      fill: true,
      color: color.pondMist,
      path: [{ x: -2200, y: 150 }, { x: 2200, y: 150 }, { x: 2200, y: 255 }, { x: -2200, y: 255 }],
      translate: { z: -505 }
    });

    const shore = new Zdog.Shape({
      addTo: bgAnchor,
      stroke: 0,
      fill: true,
      color: color.shore,
      path: [{ x: -2200, y: 220 }, { x: 2200, y: 220 }, { x: 2200, y: 290 }, { x: -2200, y: 290 }],
      translate: { z: -490 }
    });

    new Zdog.Shape({
      addTo: bgAnchor,
      stroke: 0,
      fill: true,
      color: color.shoreLight,
      path: [{ x: -2200, y: 220 }, { x: 2200, y: 220 }, { x: 2200, y: 245 }, { x: -2200, y: 245 }],
      translate: { z: -488 }
    });

    for (let i = -6; i <= 6; i += 2) {
      new Zdog.Shape({
        addTo: bgAnchor,
        path: [{ x: -220 + i * 44, y: 214 }, { x: -160 + i * 44, y: 218 }, { x: -88 + i * 44, y: 214 }],
        stroke: 5,
        color: '#DDFBF5',
        translate: { z: -487 }
      });
    }

    for (let i = 0; i < 12; i++) makeReed(bgAnchor, -820 + i * 140, 260, -470, 65 + (i % 3) * 12);

    makeFlower(bgAnchor, -320, 280, 20, 'rose');
    makeFlower(bgAnchor, -170, 300, 18, 'daisy');
    makeFlower(bgAnchor, -30, 290, 24, 'tulip');
    makeFlower(bgAnchor, 130, 296, 24, 'daisy');
    makeFlower(bgAnchor, 290, 302, 18, 'rose');

    const fishZone = new Zdog.Anchor({ addTo: scene, translate: { z: 120 } });

    blobMasterAnchor = new Zdog.Anchor({ addTo: fishZone, rotate: { y: 0.15 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 130, color: color.blobSkin });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 90, color: color.blobSkin, translate: { y: -40, z: 6 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 50, color: color.blobNose, translate: { y: 14, z: 42 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 36, color: color.blobNose, translate: { y: 32, z: 34 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 24, color: color.blobNose, translate: { y: 46, z: 22 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 60, color: color.blobSkin, translate: { x: -38, y: 24, z: 18 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 60, color: color.blobSkin, translate: { x: 38, y: 24, z: 18 } });

    eyeL = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { x: -18, y: -14, z: 52 } });
    new Zdog.Shape({ addTo: eyeL, stroke: 15, color: '#FFFFFF' });
    new Zdog.Shape({ addTo: eyeL, stroke: 6, color: '#111111', translate: { z: 6 } });
    new Zdog.Shape({ addTo: eyeL, stroke: 5, color: color.blobNose, closed: false, path: [{ x: -10, y: 4 }, { x: 0, y: 8 }, { x: 10, y: 4 }] });

    eyeR = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { x: 18, y: -14, z: 52 } });
    new Zdog.Shape({ addTo: eyeR, stroke: 15, color: '#FFFFFF' });
    new Zdog.Shape({ addTo: eyeR, stroke: 6, color: '#111111', translate: { z: 6 } });
    new Zdog.Shape({ addTo: eyeR, stroke: 5, color: color.blobNose, closed: false, path: [{ x: -10, y: 4 }, { x: 0, y: 8 }, { x: 10, y: 4 }] });

    new Zdog.Shape({
      addTo: blobMasterAnchor,
      stroke: 7,
      color: color.blobMouth,
      closed: false,
      path: [{ x: -28, y: 40, z: 44 }, { bezier: [{ x: -12, y: 28, z: 54 }, { x: 12, y: 28, z: 54 }, { x: 28, y: 40, z: 44 }] }]
    });

    leftPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    const leftPecAnchor = new Zdog.Anchor({ addTo: leftPecContainer, translate: { x: -52, y: 22, z: -8 } });
    new Zdog.Shape({
      addTo: leftPecAnchor,
      stroke: 14,
      color: color.blobFin,
      closed: false,
      path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: -18, y: 6, z: 6 }, { x: -34, y: 16, z: 0 }, { x: -40, y: 20, z: -12 }] }]
    });

    rightPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    const rightPecAnchor = new Zdog.Anchor({ addTo: rightPecContainer, translate: { x: 52, y: 22, z: -8 } });
    new Zdog.Shape({
      addTo: rightPecAnchor,
      stroke: 14,
      color: color.blobFin,
      closed: false,
      path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: 18, y: 6, z: 6 }, { x: 34, y: 16, z: 0 }, { x: 40, y: 20, z: -12 }] }]
    });

    const blobTail = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { y: 10, z: -68 } });
    new Zdog.Shape({ addTo: blobTail, stroke: 26, color: color.blobTailBody });
    blobTailFin = new Zdog.Anchor({ addTo: blobTail, translate: { z: -14 } });
    new Zdog.Shape({
      addTo: blobTailFin,
      stroke: 18,
      color: color.blobFin,
      closed: false,
      path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -4 }, { x: -12, y: -12, z: -8 }, { x: 0, y: -18, z: -14 }] }]
    });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 96, color: color.blobSkin, translate: { y: 22, z: 4 } });

    handsomeMasterAnchor = new Zdog.Anchor({ addTo: fishZone, rotate: { y: 0.15 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 82, color: color.blobDeepSea });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 78, color: color.blobDeepSea, translate: { z: -30 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 50, color: color.blobDeepSea, translate: { x: -28, y: 14, z: 12 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 50, color: color.blobDeepSea, translate: { x: 28, y: 14, z: 12 } });

    new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { x: -22, y: -12, z: 30 } });
    new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { x: 22, y: -12, z: 30 } });

    new Zdog.Shape({
      addTo: handsomeMasterAnchor,
      stroke: 5.5,
      color: color.blobMouthDeep,
      closed: false,
      path: [{ x: -18, y: 18, z: 40 }, { x: 0, y: 24, z: 42 }, { x: 18, y: 18, z: 40 }]
    });

    leftWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    const leftWingFin = new Zdog.Anchor({ addTo: leftWingContainer, translate: { x: -42, y: 16, z: -6 } });
    new Zdog.Shape({
      addTo: leftWingFin,
      stroke: 12,
      color: color.blobDeepSeaDark,
      closed: false,
      path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: -18, y: 3, z: 7 }, { x: -30, y: 10, z: 0 }, { x: -36, y: 12, z: -8 }] }]
    });

    rightWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    const rightWingFin = new Zdog.Anchor({ addTo: rightWingContainer, translate: { x: 42, y: 16, z: -6 } });
    new Zdog.Shape({
      addTo: rightWingFin,
      stroke: 12,
      color: color.blobDeepSeaDark,
      closed: false,
      path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: 18, y: 3, z: 7 }, { x: 30, y: 10, z: 0 }, { x: 36, y: 12, z: -8 }] }]
    });

    const handsomeTail = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { y: -4, z: -72 } });
    new Zdog.Shape({ addTo: handsomeTail, stroke: 30, color: color.blobDeepSea });
    handsomeTailFin = new Zdog.Anchor({ addTo: handsomeTail, translate: { z: -14 } });
    new Zdog.Shape({
      addTo: handsomeTailFin,
      stroke: 40,
      color: color.blobDeepSeaDark,
      closed: false,
      path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -5 }, { x: -14, y: -14, z: -9 }, { x: 0, y: -22, z: -14 }] }]
    });

    pinkFadeShapes = setupFadeGroup(blobMasterAnchor);
    darkFadeShapes = setupFadeGroup(handsomeMasterAnchor);

    let frame = 0;
    let running = true;

    function render() {
      if (!running) return;
      frame++;

      pinkFadeShapes.forEach(({ node, r, g, b }) => {
        node.color = `rgba(${r},${g},${b},${fishState.pinkAlpha})`;
      });
      darkFadeShapes.forEach(({ node, r, g, b }) => {
        node.color = `rgba(${r},${g},${b},${fishState.darkAlpha})`;
      });

      blobMasterAnchor.visible = fishState.pinkAlpha > 0.01;
      handsomeMasterAnchor.visible = fishState.darkAlpha > 0.01;

      const bob = Math.sin(frame * 0.025) * 5;
      blobMasterAnchor.translate.y = bob;
      handsomeMasterAnchor.translate.y = bob;

      const sway = Math.sin(frame * 0.018) * 0.06;
      blobMasterAnchor.rotate.x = sway;
      handsomeMasterAnchor.rotate.x = sway;

      leftPecContainer.rotate.z = Math.sin(frame / 22) * 0.14;
      rightPecContainer.rotate.z = -Math.sin(frame / 22) * 0.14;
      leftWingContainer.rotate.z = Math.sin(frame / 26) * 0.11;
      rightWingContainer.rotate.z = -Math.sin(frame / 26) * 0.11;

      if (blobTailFin) blobTailFin.rotate.y = Math.sin(frame * 0.07) * 0.18;
      if (handsomeTailFin) handsomeTailFin.rotate.y = Math.sin(frame * 0.06) * 0.16;

      const blink = frame % 280;
      if (blink > 262) {
        const s = 0.5 + 0.5 * Math.cos(((blink - 262) / 18) * Math.PI * 2);
        eyeL.scale.y = s;
        eyeR.scale.y = s;
      } else {
        eyeL.scale.y = 1;
        eyeR.scale.y = 1;
      }

      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }

    render();

    return () => { running = false; };
  });

  function handleFishClick() {
    if (isAnimating) return;
    isAnimating = true;
    isDeep = !isDeep;

    gsap.timeline({
      onComplete: () => {
        isAnimating = false;
        hintText = isDeep ? 'click to surface' : 'click the blobfish to dive';
      }
    })
    .to(fishState, {
      duration: 1.2,
      pinkAlpha: isDeep ? 0 : 1,
      darkAlpha: isDeep ? 1 : 0,
      ease: 'power2.inOut'
    }, 0)
    .to([blobMasterAnchor, handsomeMasterAnchor], {
      duration: 1.2,
      rotate: { y: 0.15 },
      ease: 'power2.inOut'
    }, 0);
  }
</script>

<div class="scene-wrapper">
  <canvas bind:this={canvasRef} class="zdog-canvas"></canvas>
  <button class="fish-hit" on:click={handleFishClick} aria-label="Toggle blobfish mode"></button>
  <p class="hint">{hintText}</p>
</div>

<style>
  :global(body) {
    margin: 0;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    background: #86D7FF;
    font-family: Georgia, serif;
  }

  .scene-wrapper {
    position: fixed;
    inset: 0;
    overflow: hidden;
  }

  .zdog-canvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    display: block;
    pointer-events: none;
    background: transparent;
  }

  .fish-hit {
    position: absolute;
    left: 50%;
    top: 50%;
    width: 360px;
    height: 360px;
    transform: translate(-50%, -50%);
    background: transparent;
    border: 0;
    cursor: pointer;
    z-index: 20;
    pointer-events: auto;
  }

  .hint {
    position: absolute;
    left: 50%;
    bottom: 18px;
    transform: translateX(-50%);
    color: rgba(255,255,255,0.78);
    text-shadow: 0 1px 8px rgba(0,0,0,0.35);
    z-index: 30;
    pointer-events: none;
    font-style: italic;
    letter-spacing: 0.05em;
  }
</style>