<script>
  //@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';

  let canvasRef;
  let isDeep = false;
  let isAnimating = false;
  let hintText = 'Drag to rotate • Click to dive';

  let fishState = { pinkAlpha: 1, darkAlpha: 0 };

  let scene;
  let blobMasterAnchor, handsomeMasterAnchor;
  let leftPecContainer, rightPecContainer;
  let leftWingContainer, rightWingContainer;
  let handsomeTailFin, blobTailFin;
  let eyeL_pink, eyeR_pink, eyeL_blue, eyeR_blue;

  let pinkFadeShapes = [];
  let darkFadeShapes = [];

  const color = {
    blobSkin: '#F4B8C2',
    blobNose: '#F3AEBB',
    blobFin: '#C97A8D',
    blobMouth: '#6B3A45',
    blobTailBody: '#D88A9D',
    blobDeepSea: '#6C8EA4',
    blobDeepSeaDark: '#4A6578',
    blobMouthDeep: '#334856',
    blobEyeDeep: '#94b3c7',
    blobPupilDeep: '#1d2a33'
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

  onMount(() => {
    scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: true,
      resize: 'window',
      rotate: { x: -0.2, y: -0.12 },
    });

    const fishZone = new Zdog.Anchor({ addTo: scene });

    // ==========================================
    // --- Blobfish (Surface - Pink) ---
    // ==========================================
    blobMasterAnchor = new Zdog.Anchor({ addTo: fishZone });
    
    // Body
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 130, color: color.blobSkin });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 90, color: color.blobSkin, translate: { y: -40, z: 6 } });
    
    // Nose
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 50, color: color.blobNose, translate: { y: 14, z: 62 } });
    
    // Cheeks
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 60, color: color.blobSkin, translate: { x: -38, y: 24, z: 18 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 60, color: color.blobSkin, translate: { x: 38, y: 24, z: 18 } });

    // Eyes
    eyeL_pink = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { x: -18, y: -14, z: 52 } });
    new Zdog.Shape({ addTo: eyeL_pink, stroke: 15, color: '#FFFFFF' });
    new Zdog.Shape({ addTo: eyeL_pink, stroke: 6, color: '#111111', translate: { z: 6 } });
    new Zdog.Shape({ addTo: eyeL_pink, stroke: 5, color: color.blobNose, closed: false, path: [{ x: -10, y: 4 }, { x: 0, y: 8 }, { x: 10, y: 4 }] });

    eyeR_pink = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { x: 18, y: -14, z: 52 } });
    new Zdog.Shape({ addTo: eyeR_pink, stroke: 15, color: '#FFFFFF' });
    new Zdog.Shape({ addTo: eyeR_pink, stroke: 6, color: '#111111', translate: { z: 6 } });
    new Zdog.Shape({ addTo: eyeR_pink, stroke: 5, color: color.blobNose, closed: false, path: [{ x: -10, y: 4 }, { x: 0, y: 8 }, { x: 10, y: 4 }] });

    // Mouth
    new Zdog.Shape({
      addTo: blobMasterAnchor,
      stroke: 7,
      color: color.blobMouth,
      closed: false,
      path: [{ x: -28, y: 40, z: 44 }, { bezier: [{ x: -12, y: 28, z: 54 }, { x: 12, y: 28, z: 54 }, { x: 28, y: 40, z: 44 }] }]
    });

    // Solid Wings/Pectorals Shared Path Geometry
    const finShapePath = [
      { x: 0, y: 0 },
      { x: -30, y: -10 },
      { x: -50, y: 15 },
      { x: -40, y: 35 },
      { x: -10, y: 25 },
    ];

    leftPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    const leftPecAnchor = new Zdog.Anchor({ addTo: leftPecContainer, translate: { x: -52, y: 22, z: -8 } });
    new Zdog.Shape({
      addTo: leftPecAnchor,
      path: finShapePath,
      stroke: 12,
      color: color.blobFin,
      fill: true,
    });

    rightPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    const rightPecAnchor = new Zdog.Anchor({ addTo: rightPecContainer, translate: { x: 52, y: 22, z: -8 } });
    new Zdog.Shape({
      addTo: rightPecAnchor,
      path: finShapePath,
      scale: { x: -1 },
      stroke: 12,
      color: color.blobFin,
      fill: true,
    });

    // Tail
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


    // ==========================================
    // --- Handsome Blobfish (Deep - Blue) ---
    // ==========================================
    handsomeMasterAnchor = new Zdog.Anchor({ 
      addTo: fishZone,
      scale: 1.5
    });

    // Body
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 120, color: color.blobDeepSea });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 95, color: color.blobDeepSea, translate: { z: -30 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 60, color: color.blobDeepSea, translate: { x: -28, y: 14, z: 12 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 60, color: color.blobDeepSea, translate: { x: 28, y: 14, z: 12 } });

    // Eyes
    eyeL_blue = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { x: -22, y: -12, z: 25 } });
    new Zdog.Shape({ addTo: eyeL_blue, stroke: 16, color: color.blobEyeDeep });
    new Zdog.Shape({ addTo: eyeL_blue, stroke: 7, color: color.blobPupilDeep, translate: { z: 6 } });

    eyeR_blue = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { x: 22, y: -12, z: 25 } });
    new Zdog.Shape({ addTo: eyeR_blue, stroke: 16, color: color.blobEyeDeep });
    new Zdog.Shape({ addTo: eyeR_blue, stroke: 7, color: color.blobPupilDeep, translate: { z: 6 } });

    // Mouth
    new Zdog.Shape({
      addTo: handsomeMasterAnchor,
      stroke: 5.5,
      color: color.blobMouthDeep,
      closed: false,
      path: [{ x: -18, y: 10, z: 25 }, { x: 0, y: 14, z: 27 }, { x: 18, y: 10, z: 25 }]
    });

    // Wings (Proportionally balanced by counter-scaling 1/1.5 down)
    leftWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    const leftWingFin = new Zdog.Anchor({ addTo: leftWingContainer, translate: { x: -28, y: 12, z: -8 } });
    new Zdog.Shape({
      addTo: leftWingFin,
      path: finShapePath,
      scale: { x: 0.666, y: 0.666, z: 0.666 }, // Counter-scales master anchor's 1.5x amplification
      stroke: 8, // Set to look equivalent to a standard stroke of 12
      color: color.blobDeepSeaDark,
      fill: true,
    });

    rightWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    const rightWingFin = new Zdog.Anchor({ addTo: rightWingContainer, translate: { x: 28, y: 12, z: -8 } });
    new Zdog.Shape({
      addTo: rightWingFin,
      path: finShapePath,
      scale: { x: -0.666, y: 0.666, z: 0.666 }, // Flip on x-axis and counter-scale sizing
      stroke: 8,
      color: color.blobDeepSeaDark,
      fill: true,
    });

    // Tail
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

    canvasRef.addEventListener('click', (e) => {
      handleFishClick();
    });

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

      leftPecContainer.rotate.z = Math.sin(frame / 22) * 0.14;
      rightPecContainer.rotate.z = -Math.sin(frame / 22) * 0.14;
      leftWingContainer.rotate.z = Math.sin(frame / 26) * 0.11;
      rightWingContainer.rotate.z = -Math.sin(frame / 26) * 0.11;

      if (blobTailFin) blobTailFin.rotate.y = Math.sin(frame * 0.07) * 0.18;
      if (handsomeTailFin) handsomeTailFin.rotate.y = Math.sin(frame * 0.06) * 0.16;

      const blink = frame % 280;
      let s_blink = 1;
      if (blink > 262) {
        s_blink = 0.5 + 0.5 * Math.cos(((blink - 262) / 18) * Math.PI * 2);
      }
      
      eyeL_pink.scale.y = s_blink;
      eyeR_pink.scale.y = s_blink;
      eyeL_blue.scale.y = s_blink;
      eyeR_blue.scale.y = s_blink;

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

    gsap.to(fishState, {
      duration: 1.2,
      pinkAlpha: isDeep ? 0 : 1,
      darkAlpha: isDeep ? 1 : 0,
      ease: 'power2.inOut',
      onComplete: () => {
        isAnimating = false;
        hintText = isDeep ? 'Click to surface' : 'Click to dive';
      }
    });
  }
</script>

<div class="scene-wrapper">
  <canvas bind:this={canvasRef} class="zdog-canvas"></canvas>
  <p class="hint">{hintText}</p>
</div>

<style>
  :global(body) {
    margin: 0;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background-color: #1a1a2e; 
  }

  .scene-wrapper {
    position: fixed;
    inset: 0;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .zdog-canvas {
    width: 100%;
    height: 100%;
    cursor: grab;
    touch-action: none;
  }

  .zdog-canvas:active {
    cursor: grabbing;
  }

  .hint {
    position: absolute;
    bottom: 30px;
    color: white;
    opacity: 0.6;
    pointer-events: none;
    font-size: 0.9rem;
    text-transform: uppercase;
    letter-spacing: 2px;
  }
</style>