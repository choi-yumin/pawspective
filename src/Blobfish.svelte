<script>
  //@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';

  let canvasRef;
  let uiCanvasRef;
  let overlayCanvasRef; 
  let wrapperRef; 
  
  let isDeep = false;
  let isAnimating = false;
  let hintText = 'Drag to rotate • Click the gauge to dive';

  // These variables now act as immediate binary switches (1 or 0) triggered only behind the wave cover
  let fishState = { pinkAlpha: 1, darkAlpha: 0 };
  let transitionProgress = { value: 0 }; 

  let scene, uiScene;
  let blobMasterAnchor, handsomeMasterAnchor;
  let leftPecContainer, rightPecContainer;
  let leftWingContainer, rightWingContainer;
  let handsomeTailFin, blobTailFin;
  let eyeL_pink, eyeR_pink, eyeL_blue, eyeR_blue;
  let gaugeNeedle;

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
    blobPupilDeep: '#1d2a33',
    waterTransition: '#4A90E2' 
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
    wrapperRef.style.setProperty('--sky-pos', '45%');
    wrapperRef.style.setProperty('--mid-pos', '75%');
    wrapperRef.style.setProperty('--abyss-pos', '100%');

    const ctx = overlayCanvasRef.getContext('2d');
    function resizeOverlay() {
      overlayCanvasRef.width = window.innerWidth;
      overlayCanvasRef.height = window.innerHeight;
    }
    resizeOverlay();
    window.addEventListener('resize', resizeOverlay);

    // ----------------------------------------------------
    // MAIN FISH SCENE (Rotatable)
    // ----------------------------------------------------
    scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: true,
      resize: 'window',
      rotate: { x: -0.2, y: -0.12 },
    });

    const fishZone = new Zdog.Anchor({ addTo: scene });

    // --- Blobfish (Surface - Pink) ---
    blobMasterAnchor = new Zdog.Anchor({ addTo: fishZone });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 130, color: color.blobSkin });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 90, color: color.blobSkin, translate: { y: -40, z: 6 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 50, color: color.blobNose, translate: { y: 14, z: 62 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 60, color: color.blobSkin, translate: { x: -38, y: 24, z: 18 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 60, color: color.blobSkin, translate: { x: 38, y: 24, z: 18 } });

    eyeL_pink = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { x: -18, y: -14, z: 52 } });
    new Zdog.Shape({ addTo: eyeL_pink, stroke: 15, color: '#FFFFFF' });
    new Zdog.Shape({ addTo: eyeL_pink, stroke: 6, color: '#111111', translate: { z: 6 } });
    new Zdog.Shape({ addTo: eyeL_pink, stroke: 5, color: color.blobNose, closed: false, path: [{ x: -10, y: 4 }, { x: 0, y: 8 }, { x: 10, y: 4 }] });

    eyeR_pink = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { x: 18, y: -14, z: 52 } });
    new Zdog.Shape({ addTo: eyeR_pink, stroke: 15, color: '#FFFFFF' });
    new Zdog.Shape({ addTo: eyeR_pink, stroke: 6, color: '#111111', translate: { z: 6 } });
    new Zdog.Shape({ addTo: eyeR_pink, stroke: 5, color: color.blobNose, closed: false, path: [{ x: -10, y: 4 }, { x: 0, y: 8 }, { x: 10, y: 4 }] });

    new Zdog.Shape({
      addTo: blobMasterAnchor,
      stroke: 7,
      color: color.blobMouth,
      closed: false,
      path: [{ x: -28, y: 40, z: 44 }, { bezier: [{ x: -12, y: 28, z: 54 }, { x: 12, y: 28, z: 54 }, { x: 28, y: 40, z: 44 }] }]
    });

    const finShapePath = [{ x: 0, y: 0 }, { x: -30, y: -10 }, { x: -50, y: 15 }, { x: -40, y: 35 }, { x: -10, y: 25 }];

    leftPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    const leftPecAnchor = new Zdog.Anchor({ addTo: leftPecContainer, translate: { x: -52, y: 22, z: -8 } });
    new Zdog.Shape({ addTo: leftPecAnchor, path: finShapePath, stroke: 12, color: color.blobFin, fill: true });

    rightPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    const rightPecAnchor = new Zdog.Anchor({ addTo: rightPecContainer, translate: { x: 52, y: 22, z: -8 } });
    new Zdog.Shape({ addTo: rightPecAnchor, path: finShapePath, scale: { x: -1 }, stroke: 12, color: color.blobFin, fill: true });

    const blobTail = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { y: 10, z: -68 } });
    new Zdog.Shape({ addTo: blobTail, stroke: 26, color: color.blobTailBody });
    blobTailFin = new Zdog.Anchor({ addTo: blobTail, translate: { z: -14 } });
    new Zdog.Shape({ addTo: blobTailFin, stroke: 18, color: color.blobFin, closed: false, path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -4 }, { x: -12, y: -12, z: -8 }, { x: 0, y: -18, z: -14 }] }] });

    // --- Handsome Blobfish (Deep - Blue) ---
    handsomeMasterAnchor = new Zdog.Anchor({ addTo: fishZone, scale: 1.5 });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 120, color: color.blobDeepSea });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 95, color: color.blobDeepSea, translate: { z: -30 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 60, color: color.blobDeepSea, translate: { x: -28, y: 14, z: 12 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 60, color: color.blobDeepSea, translate: { x: 28, y: 14, z: 12 } });

    eyeL_blue = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { x: -22, y: -12, z: 25 } });
    new Zdog.Shape({ addTo: eyeL_blue, stroke: 16, color: color.blobEyeDeep });
    new Zdog.Shape({ addTo: eyeL_blue, stroke: 7, color: color.blobPupilDeep, translate: { z: 6 } });

    eyeR_blue = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { x: 22, y: -12, z: 25 } });
    new Zdog.Shape({ addTo: eyeR_blue, stroke: 16, color: color.blobEyeDeep });
    new Zdog.Shape({ addTo: eyeR_blue, stroke: 7, color: color.blobPupilDeep, translate: { z: 6 } });

    new Zdog.Shape({
      addTo: handsomeMasterAnchor,
      stroke: 5.5,
      color: color.blobMouthDeep,
      closed: false,
      path: [{ x: -18, y: 10, z: 25 }, { x: 0, y: 14, z: 27 }, { x: 18, y: 10, z: 25 }]
    });

    leftWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    const leftWingFin = new Zdog.Anchor({ addTo: leftWingContainer, translate: { x: -48, y: 22, z: -8 } });
    new Zdog.Shape({ addTo: leftWingFin, path: finShapePath, scale: { x: 0.666, y: 0.666, z: 0.666 }, stroke: 8, color: color.blobDeepSeaDark, fill: true });

    rightWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    const rightWingFin = new Zdog.Anchor({ addTo: rightWingContainer, translate: { x: 48, y: 22, z: -8 } });
    new Zdog.Shape({ addTo: rightWingFin, path: finShapePath, scale: { x: -0.666, y: 0.666, z: 0.666 }, stroke: 8, color: color.blobDeepSeaDark, fill: true });

    const handsomeTail = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { y: -4, z: -72 } });
    new Zdog.Shape({ addTo: handsomeTail, stroke: 30, color: color.blobDeepSea });
    handsomeTailFin = new Zdog.Anchor({ addTo: handsomeTail, translate: { z: -14 } });
    new Zdog.Shape({ addTo: handsomeTailFin, stroke: 40, color: color.blobDeepSeaDark, closed: false, path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -5 }, { x: -14, y: -14, z: -9 }, { x: 0, y: -22, z: -14 }] }] });

    pinkFadeShapes = setupFadeGroup(blobMasterAnchor);
    darkFadeShapes = setupFadeGroup(handsomeMasterAnchor);

    // ----------------------------------------------------
    // FIXED BACKGROUND UI GAUGE
    // ----------------------------------------------------
    uiScene = new Zdog.Illustration({
      element: uiCanvasRef,
      dragRotate: false,
      resize: false
    });

    const UIAnchor = new Zdog.Anchor({ addTo: uiScene, translate: { x: 0, y: 0 } });

    new Zdog.Cylinder({ addTo: UIAnchor, diameter: 82, length: 16, stroke: false, color: '#7f8c8d', backface: '#34495e' });
    new Zdog.Cylinder({ addTo: UIAnchor, diameter: 70, length: 2, stroke: false, color: '#FFFFFF', translate: { z: 8 } });

    for (let i = 0; i < 6; i++) {
      let angle = (i / 5) * Math.PI - Math.PI;
      new Zdog.Shape({
        addTo: UIAnchor,
        path: [{ y: -26 }, { y: -32 }],
        stroke: 3,
        color: '#e74c3c',
        rotate: { z: angle },
        translate: { z: 9.5 }
      });
    }

    gaugeNeedle = new Zdog.Anchor({ addTo: UIAnchor, translate: { z: 11 }, rotate: { z: -Math.PI * 0.75 } });
    new Zdog.Shape({ addTo: gaugeNeedle, path: [{ y: 6 }, { y: -30 }], stroke: 4, color: '#2c3e50' });
    new Zdog.Shape({ addTo: gaugeNeedle, stroke: 12, color: '#e74c3c' });

    uiScene.updateRenderGraph();

    // ----------------------------------------------------
    // MAIN RENDERING LOOP
    // ----------------------------------------------------
    let frame = 0;
    let running = true;

    function drawWaterTransition() {
      ctx.clearRect(0, 0, overlayCanvasRef.width, overlayCanvasRef.height);
      
      let progress = transitionProgress.value;
      if (progress <= 0 || progress >= 1) return;

      const w = overlayCanvasRef.width;
      const h = overlayCanvasRef.height;

      ctx.fillStyle = color.waterTransition;

      const numBars = 3;
      const barThickness = Math.max(w, h) * 0.4;
      
      for (let i = 0; i < numBars; i++) {
        let barProgress = gsap.utils.mapRange(i * 0.1, 1 - (numBars - 1 - i) * 0.1, 0, 1, progress);
        barProgress = gsap.utils.clamp(0, 1, barProgress);

        ctx.save();
        let centerX = -barThickness + (w + barThickness * 2) * barProgress;
        ctx.translate(centerX, h / 2);
        ctx.rotate(-Math.PI / 6); 

        ctx.fillRect(-barThickness / 2, -h * 1.5, barThickness, h * 3);
        ctx.restore();
      }
    }

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
      uiScene.updateRenderGraph();
      drawWaterTransition();
      
      requestAnimationFrame(render);
    }

    render();

    return () => { 
      running = false; 
      window.removeEventListener('resize', resizeOverlay);
    };
  });

  function handleDiveToggle() {
    if (isAnimating) return;
    isAnimating = true;
    isDeep = !isDeep;

    const tl = gsap.timeline({
      onComplete: () => {
        isAnimating = false;
        hintText = isDeep ? 'Click the gauge to surface' : 'Click the gauge to dive';
      }
    });

    // Environmental backdrop sliding movement
    tl.to(wrapperRef, {
      duration: 2.5,
      '--sky-pos': isDeep ? '-120%' : '45%',
      '--mid-pos': isDeep ? '-40%' : '75%',
      '--abyss-pos': isDeep ? '0%' : '100%',
      ease: 'power2.inOut'
    }, 0);

    // Indicator needle response
    tl.to(gaugeNeedle.rotate, {
      duration: 2.5,
      z: isDeep ? Math.PI * 0.75 : -Math.PI * 0.75,
      ease: 'back.out(1.2)'
    }, 0);

    // Track water wave position modifier cleanly
    transitionProgress.value = 0;
    tl.to(transitionProgress, {
      duration: 1.4,
      value: 1,
      ease: 'power1.inOut'
    }, 0.55);

    // INSTEAD OF TWEENING OVER TIME: We insert a single zero-duration swap 
    // precisely when the center thick water band completely masks the center of the viewport (progress = 0.5)
    tl.to({}, { duration: 0 }, 1.25); // Timeline position marker alignment anchor
    tl.call(() => {
      if (isDeep) {
        fishState.pinkAlpha = 0;
        fishState.darkAlpha = 1;
      } else {
        fishState.pinkAlpha = 1;
        fishState.darkAlpha = 0;
      }
    }, null, 1.25); 
  }
</script>

<div bind:this={wrapperRef} class="scene-wrapper">
  
  <button class="gauge-button" on:click={handleDiveToggle} aria-label="Toggle Sea Depth">
    <canvas bind:this={uiCanvasRef} width="150" height="150" class="ui-canvas"></canvas>
  </button>

  <canvas bind:this={canvasRef} class="zdog-canvas"></canvas>
  
  <canvas bind:this={overlayCanvasRef} class="overlay-canvas"></canvas>
  
  <p class="hint">{hintText}</p>
</div>

<style>
  :global(body) {
    margin: 0;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }

  .scene-wrapper {
    position: fixed;
    inset: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    background: linear-gradient(
      to bottom,
      #FAD0C4 0%,
      #FAD0C4 var(--sky-pos),
      #4A90E2 var(--sky-pos),
      #1C2541 var(--mid-pos),
      #0B132B var(--abyss-pos)
    );
  }

  .zdog-canvas {
    width: 100%;
    height: 100%;
    cursor: grab;
    touch-action: none;
    z-index: 1;
  }
  .zdog-canvas:active {
    cursor: grabbing;
  }

  .overlay-canvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; 
    z-index: 2;
  }

  .gauge-button {
    position: absolute;
    top: 25px;
    right: 25px;
    background: none;
    border: none;
    padding: 0;
    margin: 0;
    cursor: pointer;
    z-index: 10; 
    width: 150px;
    height: 150px;
    display: block;
    overflow: visible; 
    filter: drop-shadow(0px 8px 16px rgba(0,0,0,0.35));
    transition: transform 0.2s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  }
  
  .ui-canvas {
    display: block;
    width: 150px;
    height: 150px;
  }

  .gauge-button:hover {
    transform: scale(1.05);
  }
  .gauge-button:active {
    transform: scale(0.95);
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
    text-shadow: 0 2px 4px rgba(0,0,0,0.6);
    z-index: 3;
  }
</style>