<script>
  // @ts-nocheck
  import { onMount, onDestroy } from 'svelte';
  import Zdog from 'zdog';

  let canvas;
  let illo;
  let rafId;
  let clouds = [];

  function buildScene(illo) {
    const CLOUD_PINK = '#FFE4F0';
    const CLOUD_WHITE = '#FFF7FB';

    // Define initial cloud positions, scales, and drift speeds
    const cloudConfigs = [
      { x: -600, y: -200, z: -200, s: 1.2, speed: 0.5 },
      { x: 200, y: -100, z: -400, s: 0.9, speed: 0.3 },
      { x: 600, y: 200, z: -600, s: 1.3, speed: 0.6 },
      { x: -300, y: 300, z: -300, s: 1.1, speed: 0.4 },
      { x: 0, y: 0, z: -800, s: 1.0, speed: 0.2 },
    ];

    cloudConfigs.forEach(config => {
      const c = new Zdog.Anchor({ 
        addTo: illo, 
        translate: { x: config.x, y: config.y, z: config.z }, 
        scale: config.s 
      });
      
      new Zdog.Shape({ addTo: c, stroke: 110, color: CLOUD_WHITE });
      new Zdog.Shape({ addTo: c, stroke: 76,  color: CLOUD_PINK,  translate: { x: -58, y: 12 } });
      new Zdog.Shape({ addTo: c, stroke: 82,  color: CLOUD_WHITE, translate: { x: 56,  y: 6 } });
      new Zdog.Shape({ addTo: c, stroke: 64,  color: CLOUD_PINK,  translate: { x: -98, y: 16 } });
      new Zdog.Shape({ addTo: c, stroke: 58,  color: CLOUD_WHITE, translate: { x: 100, y: 12 } });
      
      // Save the anchor and its speed so we can animate it later
      clouds.push({ anchor: c, speed: config.speed });
    });
  }

  onMount(() => {
    // Initialize the Zdog illustration
    illo = new Zdog.Illustration({
      element: canvas,
      dragRotate: false, // Disabled to keep the camera completely static
      resize: 'window',
      zoom: 1,
    });

    buildScene(illo);

    // The animation loop
    function tick() {
      // Move each cloud to the right
      clouds.forEach(cloud => {
        cloud.anchor.translate.x += cloud.speed;
        
        // When a cloud moves entirely off the right side of the screen, 
        // snap it back to the far left to create an infinite loop.
        if (cloud.anchor.translate.x > 1200) {
          cloud.anchor.translate.x = -1200;
        }
      });

      illo.updateRenderGraph();
      rafId = requestAnimationFrame(tick);
    }
    
    tick(); // Start the loop
  });

  onDestroy(() => {
    cancelAnimationFrame(rafId);
  });
</script>

<canvas bind:this={canvas} class="sky-canvas"></canvas>

<style>
  :global(html, body) {
    height: 100%;
    margin: 0;
    overflow: hidden;
  }
  
  .sky-canvas {
    position: fixed;
    left: 0;
    top: 0;
    width: 100vw;
    height: 100vh;
    background: #FFD9EC; /* Static pink background */
    display: block;
    border: none;
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }
</style>