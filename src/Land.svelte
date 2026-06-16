<script>
// @ts-nocheck
  import { onMount, onDestroy } from 'svelte';
  import Zdog from 'zdog';

  const TAU = Zdog.TAU;

  // tweakables
  const MIN_ZOOM = 0.15;
  const MAX_ZOOM = 16;
  const START_ZOOM = 0.6;
  const ZOOM_EASE = 0.12;
  const ZOOM_STEP = 1.06;
  const SPIN_SPEED = 0.003;
  const LAND_SCALE = 0.82;

  // Asset Placements (Kept mostly identical to preserve hitboxes, minor tweaks for aesthetics)
  const DOVE_POS   = { x: 220, y: -280, z: -360 };
  const DOVE_SCALE = 1.35;

  const BEE_POS   = { x: -200, y: 20, z: -140 };
  const BEE_SCALE = 0.2;

  const CHICKEN_POS   = { x: -300, y: 30, z: 200 }; 
  const CHICKEN_SCALE = 0.6;

  const BLOB_POS = { x: 300, y: 150, z: -200 }; 
  const BLOB_SCALE = 0.5;

  const SLOTH_POS = { x: -20, y: 0, z: -120 }; 
  const SLOTH_SCALE = 1; 

  const SIGN_POS = { x: 0, y: 170, z: 800 }; // Front edge of the island

  let { onSelectDove, onSelectBee, onSelectChicken, onSelectFish, onSelectSloth } = $props();

  // Reactive and lifecycle handles
  let canvas = $state(null);   
  let illo;
  let rafId;
  let spin = true;
  let targetZoom = START_ZOOM;
  
  let dove;                  
  let bee;                   
  let wingBeat = 0;          
  let beeFrame = 0;          
  let downX = 0;             
  let downY = 0;
  let prevDragX = 0; // Tracks X movement for single-axis rotation
  let chickenHandle;         
  let fish;
  
  // Sloth handles
  let slothPivot;
  let slothHitPos = null;

  const lerp = (a, b, t) => a + (b - a) * t;

  function buildScene(illo) {
    const GRASS      = '#5ab030';
    const GRASS_DARK = '#3a8018';
    const DIRT       = '#9A6438';   
    const DIRT_DARK  = '#6E4424';   
    const POND       = '#73BFF5';
    const POND_EDGE  = '#5EA7DC';
    const BRANCH     = '#6B4A2A';
    const BRANCH_DK  = '#4E3318';
    const TREE       = '#4A7B28';
    const TREE_DARK  = '#36601A';
    const CLOUD_PINK  = '#FFE4F0';
    const CLOUD_WHITE = '#FFF7FB';

    const SURFACE_Y = 170;
    const ISLAND_DEPTH = 260;        
    const ISLAND_Z = -310;           

    const r = new Zdog.Anchor({ addTo: illo });
    const land = new Zdog.Anchor({ addTo: r, scale: LAND_SCALE });

    // ── Island slab (Made slightly wider to accommodate more nature) ──
    new Zdog.Cylinder({
      addTo: land,
      diameter: 2500,
      length: ISLAND_DEPTH,
      color: DIRT,            
      frontFace: GRASS,       
      backface: DIRT_DARK,    
      stroke: false,
      rotate: { x: TAU / 4 },
      translate: { y: SURFACE_Y + ISLAND_DEPTH / 2, z: ISLAND_Z },
    });

    new Zdog.Ellipse({
      addTo: land, diameter: 2300, stroke: 0, fill: true, color: GRASS_DARK,
      rotate: { x: TAU / 4 }, translate: { y: SURFACE_Y - 1, z: ISLAND_Z },
    });

    // ── Ambiguous Organic Pond ──
    const pondPath = [
      { x: -150, y: -200 },
      { bezier: [ {x: -150, y: -450}, {x: 250, y: -400}, {x: 350, y: -150} ] },
      { bezier: [ {x: 450, y: 100}, {x: 200, y: 350}, {x: 0, y: 300} ] },
      { bezier: [ {x: -200, y: 250}, {x: -150, y: 50}, {x: -150, y: -200} ] }
    ];

    new Zdog.Shape({
      addTo: land,
      path: pondPath,
      rotate: { x: TAU / 4 },
      translate: { x: 150, y: SURFACE_Y - 4, z: ISLAND_Z + 150 },
      stroke: 40, color: POND_EDGE, fill: true
    });

    new Zdog.Shape({
      addTo: land,
      path: pondPath,
      rotate: { x: TAU / 4 },
      translate: { x: 150, y: SURFACE_Y - 2, z: ISLAND_Z + 150 },
      stroke: 0, color: POND, fill: true
    });

    // ── Bigger, Fuller Trees ──
    function createTree(parent, x, z, scale = 1) {
      const t = new Zdog.Anchor({ addTo: parent, translate: { x, y: SURFACE_Y, z }, scale });
      // Trunk
      new Zdog.Shape({ addTo: t, path: [{ x: 0, y: 0 }, { x: 0, y: -240 }], stroke: 50, color: BRANCH });
      // Dense Canopy
      new Zdog.Shape({ addTo: t, stroke: 260, color: TREE, translate: { y: -280 } });
      new Zdog.Shape({ addTo: t, stroke: 200, color: TREE_DARK, translate: { x: -80, y: -240, z: 40 } });
      new Zdog.Shape({ addTo: t, stroke: 210, color: TREE,      translate: { x: 90,  y: -230, z: 30 } });
      new Zdog.Shape({ addTo: t, stroke: 190, color: TREE_DARK, translate: { x: 0,   y: -360, z: -50 } });
      new Zdog.Shape({ addTo: t, stroke: 180, color: TREE,      translate: { x: -50, y: -330, z: 70 } });
      new Zdog.Shape({ addTo: t, stroke: 170, color: TREE_DARK, translate: { x: 70,  y: -300, z: -70 } });
    }

    createTree(land, -650, -500, 1.3);
    createTree(land,  650, -300, 1.4);
    createTree(land,  300, -800, 1.1);
    createTree(land, -550, -100, 1.5);
    createTree(land,  750, -700, 1.2);
    createTree(land, -750, -350, 1.2);
    createTree(land, -100, -900, 1.3);
    createTree(land,  800,  100, 1.1);

    // ── Flower Garden Scatter ──
    function createFlower(parent, x, z, scale = 1) {
      const f = new Zdog.Anchor({ addTo: parent, translate: { x, y: SURFACE_Y, z }, scale });
      new Zdog.Shape({ addTo: f, path: [{y:0}, {y:-40}], stroke: 8, color: '#6A9A38' });
      
      const pColor = ['#FF6B8B', '#FFD93D', '#6BCBFF', '#D488FF', '#FFFFFF'][Math.floor(Math.random()*5)];
      const petals = new Zdog.Anchor({ addTo: f, translate: { y: -40 }, rotate: { x: TAU/8 } });
      
      new Zdog.Shape({ addTo: petals, stroke: 22, color: pColor, translate: {x:-12, y:-12} });
      new Zdog.Shape({ addTo: petals, stroke: 22, color: pColor, translate: {x:12, y:-12} });
      new Zdog.Shape({ addTo: petals, stroke: 22, color: pColor, translate: {x:-12, y:12} });
      new Zdog.Shape({ addTo: petals, stroke: 22, color: pColor, translate: {x:12, y:12} });
      
      new Zdog.Shape({ addTo: petals, stroke: 16, color: '#FFF176', translate: {z: 8} });
    }

    // Procedurally plant 60 flowers across the island
    for(let i = 0; i < 60; i++) {
      const angle = Math.random() * TAU;
      const radius = 300 + Math.random() * 800; // Keep away from exact center
      createFlower(land, Math.cos(angle) * radius, (Math.sin(angle) * radius) + ISLAND_Z, 0.4 + Math.random() * 0.5);
    }

    const slothTree = new Zdog.Anchor({
      addTo: land,
      translate: { x: SLOTH_POS.x-100, y: SURFACE_Y-320, z: SLOTH_POS.z+50 }
    });

    // ── Clouds ──
    function createCloud(parent, x, y, z, s = 1) {
      const c = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale: s });
      new Zdog.Shape({ addTo: c, stroke: 110, color: CLOUD_WHITE });
      new Zdog.Shape({ addTo: c, stroke: 76,  color: CLOUD_PINK,  translate: { x: -58, y: 12 } });
      new Zdog.Shape({ addTo: c, stroke: 82,  color: CLOUD_WHITE, translate: { x: 56,  y: 6 } });
      new Zdog.Shape({ addTo: c, stroke: 64,  color: CLOUD_PINK,  translate: { x: -98, y: 16 } });
      new Zdog.Shape({ addTo: c, stroke: 58,  color: CLOUD_WHITE, translate: { x: 100, y: 12 } });
    }

    createCloud(r, -680, -360, -500, 1.25);
    createCloud(r, -180, -450, -960, 1.15);
    createCloud(r,  320, -360, -600, 1.10);

    // ── Dove ──
    function createDove(parent, x, y, z, scale = 1) {
      const C = { white: '#ffffff', mid: '#f2f2f2', shade: '#DAD6CE', beak: '#D89A6E', beakDk: '#C07E50', cheek: '#F0A8A0', eye: '#1A1410' };
      const d = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale });
      new Zdog.Shape({ addTo: d, path: [{ x: -10, y: 15 }, { x: 25, y: 10 }], stroke: 52, color: C.white });
      new Zdog.Shape({ addTo: d, path: [{ x: -5, y: 15 }, { x: -35, y: -8 }], stroke: 36, color: C.white });
      const head = new Zdog.Anchor({ addTo: d, translate: { x: -35, y: -10, z: 0 } });
      new Zdog.Shape({ addTo: head, stroke: 35, color: C.white });
      new Zdog.Shape({ addTo: head, stroke: 6,   color: C.eye,   translate: { x: -6, y: -4, z: 15 } });
      new Zdog.Shape({ addTo: head, stroke: 1.8, color: '#fff',  translate: { x: -7.5, y: -5, z: 16 } });
      new Zdog.Shape({ addTo: head, stroke: 6,   color: C.eye,   translate: { x: -6, y: -4, z: -15 } });
      new Zdog.Shape({ addTo: head, stroke: 8,   color: C.cheek, translate: { x: -1, y: 4, z: 15 } });
      new Zdog.Shape({ addTo: head, stroke: 8,   color: C.cheek, translate: { x: -1, y: 4, z: -15 } });
      const beak = new Zdog.Anchor({ addTo: head, translate: { x: -14, y: 2, z: 0 } });
      new Zdog.Shape({ addTo: beak, path: [{ x: 0, y: -4 }, { x: -18, y: 0 }, { x: 0, y: 4 }], stroke: 2, color: C.beak, fill: true });
      new Zdog.Shape({ addTo: beak, path: [{ x: 0, y: 0 }, { x: -14, y: 0 }, { x: 0, y: 2 }], stroke: 1.5, color: C.beakDk, fill: true, translate: { y: 2 } });
      const wingPath = [{ x: -5, y: 0 }, { bezier: [{ x: 10, y: -45 }, { x: 45, y: -50 }, { x: 65, y: -15 }] }, { bezier: [{ x: 50, y: 5 }, { x: 20, y: 15 }, { x: -5, y: 0 }] }];
      const frontWing = new Zdog.Anchor({ addTo: d, translate: { x: -12, y: 10, z: 24 } });
      new Zdog.Shape({ addTo: frontWing, path: wingPath, stroke: 20, color: C.mid, fill: true });
      const backWing = new Zdog.Anchor({ addTo: d, translate: { x: -12, y: 10, z: -24 } });
      new Zdog.Shape({ addTo: backWing, path: wingPath, stroke: 20, color: C.mid, fill: true });
      const tail = new Zdog.Anchor({ addTo: d, translate: { x: 22, y: 10 } });
      new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 38, y: 4 }], stroke: 16, color: C.shade });
      new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 32, y: 2 }], stroke: 12, color: C.mid,   translate: { z: 10 } });
      new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 32, y: 2 }], stroke: 12, color: C.shade, translate: { z: -10 } });
      return { dove: d, frontWing, backWing };
    }
    dove = createDove(r, DOVE_POS.x, DOVE_POS.y, DOVE_POS.z, DOVE_SCALE);

    // ── Bee ──
    function createBee(parent, x, y, z, scale = 1) {
      const B = { yellow: '#FCD116', black: '#3D2620', white: '#FFFDF0', cheek: '#F26B50' };
      const cont = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale });
      const b    = new Zdog.Anchor({ addTo: cont });
      const head = new Zdog.Shape({ addTo: b, stroke: 28, color: B.yellow });
      new Zdog.Shape({ addTo: head, stroke: 3, color: B.black, translate: { x: -22, y: 4, z: 36 } });
      new Zdog.Shape({ addTo: head, stroke: 3, color: B.black, translate: { x:  14, y: 4, z: 40 } });
      new Zdog.Shape({ addTo: head, stroke: 2, color: B.black, closed: false, path: [{ x: -4, y: 10, z: 42 }, { bezier: [{ x: -2, y: 13, z: 42 }, { x: 2, y: 13, z: 42 }, { x: 4, y: 10, z: 42 }] }] });
      const antler = new Zdog.Anchor({ addTo: head, translate: { y: -44, z: 10 } });
      new Zdog.Shape({ addTo: antler, path: [{ y: 0, x: -10 }, { y: -22, x: -24, z: 8 }],  stroke: 1, color: B.black });
      new Zdog.Shape({ addTo: antler, path: [{ y: 0, x:  10 }, { y: -22, x:  -4, z: 12 }], stroke: 1, color: B.black });

      const leftArm  = new Zdog.Anchor({ addTo: b, translate: { x: -18, y: 45, z: 32 } });
      const rightArm = new Zdog.Anchor({ addTo: b, translate: { x:  18, y: 45, z: 32 } });
      new Zdog.Shape({ addTo: leftArm,  path: [{ y: 0 }, { y: 22 }], stroke: 2, color: B.black });
      new Zdog.Shape({ addTo: rightArm, path: [{ y: 0 }, { y: 22 }], stroke: 2, color: B.black });

      const body = new Zdog.Anchor({ addTo: b, translate: { y: 32, z: -35 } });
      const seg = new Zdog.Shape({ addTo: body, stroke: 27, color: B.yellow });
      seg.copy({ addTo: body, stroke: 40.5, color: B.black,  translate: { z: -32 } });
      seg.copy({ addTo: body, stroke: 42, color: B.yellow, translate: { z: -64 } });
      seg.copy({ addTo: body, stroke: 39, color: B.black,  translate: { z: -96 } });
      new Zdog.Shape({ addTo: body, stroke: 27, color: B.black, translate: { z: -123 } });

      const rightWing = new Zdog.Anchor({ addTo: body, translate: { z: -43, y: -65, x:  29 } });
      const leftWing  = new Zdog.Anchor({ addTo: body, translate: { z: -43, y: -65, x: -29 } });
      new Zdog.Ellipse({ addTo: rightWing, width: 80, height: 160, color: B.white, fill: true, rotate: { x: TAU/5, z:  TAU/5 }, translate: { x:  65 }, stroke: 0 });
      new Zdog.Ellipse({ addTo: leftWing,  width: 80, height: 160, color: B.white, fill: true, rotate: { x: TAU/5, z: -TAU/5 }, translate: { x: -65 }, stroke: 0 });
      return { cont, leftWing, rightWing, antler };
    }
    bee = createBee(r, BEE_POS.x, BEE_POS.y, BEE_POS.z, BEE_SCALE);

    // ── Chicken ──
    function createChicken(parent, x, y, z, scale = 1) {
      const C = { body: '#FFF1E6', headBase: '#FFF5E8', eye: '#FFFFFF', pupil: '#111111', comb: '#FF3B5C', tail: '#F8D7C7', leg: '#FAA353' };
      const chicken = new Zdog.Anchor({ addTo: parent, translate: { x, y, z }, scale, rotate: { x: -0.05 } });
      const chickenBody = new Zdog.Anchor({ addTo: chicken });

      new Zdog.Shape({ addTo: chickenBody, stroke: 162, color: C.body, translate: { y: 75 } });
      new Zdog.Cylinder({ addTo: chickenBody, diameter: 72, length: 150, stroke: 21.6, color: '#F6D8C8', rotate: { x: TAU / 4 }, translate: { y: -45, z: 18 } });

      const head = new Zdog.Anchor({ addTo: chickenBody, translate: { y: -150, z: 42 } });
      new Zdog.Shape({ addTo: head, stroke: 81, color: C.headBase });
      new Zdog.Cone({ addTo: head, diameter: 42, length: 66, stroke: 7.2, color: '#F4A63A', translate: { y: 30, z: 72 } });

      new Zdog.Shape({ addTo: head, path: [{ y: 0 }, { y: 48 }], stroke: 18, color: C.comb, translate: { x: -18, y: 42, z: 48 } }).copy({ translate: { x: 18, y: 42, z: 48 } });
      new Zdog.Shape({ addTo: head, stroke: 21.6, color: C.comb, path: [{ x: -18, y: -66, z: 6 }, { x: -30, y: -108, z: 18 }, { x: -6, y: -84, z: 2 }, { x: 6, y: -126, z: -6 }, { x: 24, y: -78, z: -12 }] });

      const eyeL = new Zdog.Anchor({ addTo: head, translate: { x: -60, y: -12, z: 42 } });
      new Zdog.Shape({ addTo: eyeL, stroke: 50.4, color: C.eye, fill: true });
      const pupilL = new Zdog.Shape({ addTo: eyeL, stroke: 18, color: C.pupil, fill: true, translate: { x: -12, y: -12, z: 36 } });

      const eyeR = new Zdog.Anchor({ addTo: head, translate: { x: 60, y: -6, z: 36 } });
      new Zdog.Shape({ addTo: eyeR, stroke: 43.2, color: C.eye, fill: true });
      const pupilR = new Zdog.Shape({ addTo: eyeR, stroke: 16.2, color: C.pupil, fill: true, translate: { x: 9, y: 6, z: 30 } });

      const createWing = (parent, xDir) => {
        const wing = new Zdog.Anchor({ addTo: chickenBody, translate: { x: 105 * xDir, y: 30, z: 15 }, rotate: { y: 0.2 * xDir, z: 0.3 * xDir } });
        for (let i = 0; i < 3; i++) {
          new Zdog.Shape({ 
            addTo: wing, 
            stroke: 16 - (i * 2), 
            color: C.body, 
            closed: false, 
            translate: { y: i * 10 },
            path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: -35 * xDir, y: -30, z: -10 }, { x: -50 * xDir, y: 20, z: -20 }, { x: -20 * xDir, y: 55, z: -10 }] }] 
          });
        }
      };
      createWing(chickenBody, -1); 
      createWing(chickenBody, 1);  

      const tailBase = new Zdog.Anchor({ addTo: chickenBody, translate: { x: 0, y: 36, z: -75 } });
      for (let i = 0; i < 5; i++) {
        const angle = (i - 2) * 0.2; 
        new Zdog.Shape({ 
          addTo: tailBase, 
          stroke: 18, 
          color: C.tail, 
          rotate: { y: angle },
          path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: -30, y: -60, z: -10 }, { x: -50, y: -120, z: -40 }, { x: -20, y: -150, z: -50 }] }] 
        });
      }

      const legStroke = 16.2;
      const legLeg = new Zdog.Shape({ path: [{ y: 60 }, { y: 120 }], stroke: legStroke, color: C.leg });
      function createFoot(parent, rotationY) {
        const footAnchor = new Zdog.Anchor({ addTo: parent, translate: { y: 120 }, rotate: { y: rotationY } });
        new Zdog.Shape({ addTo: footAnchor, path: [{ x: 0, z: 0 }, { x: 0, z: 45 }], stroke: legStroke, color: C.leg });
        const toeAngles = [-0.8, 0, 0.8]; 
        toeAngles.forEach(angle => {
          new Zdog.Shape({ addTo: footAnchor, path: [{ x: 0, z: 0 }, { x: Math.sin(angle) * 36, z: Math.cos(angle) * 30 }], stroke: legStroke * 0.8, color: C.leg });
        });
      }
      
      const leftLegAnchor = new Zdog.Anchor({ addTo: chicken, translate: { x: -42, y: 150, z: 0 } });
      legLeg.copy({ addTo: leftLegAnchor });
      createFoot(leftLegAnchor, 0.3);

      const rightLegAnchor = new Zdog.Anchor({ addTo: chicken, translate: { x: 42, y: 150, z: 0 } });
      legLeg.copy({ addTo: rightLegAnchor });
      createFoot(rightLegAnchor, -0.3);

      return { chicken, eyeL, eyeR, head };
    }
    chickenHandle = createChicken(land, CHICKEN_POS.x, CHICKEN_POS.y, CHICKEN_POS.z, CHICKEN_SCALE);

    // ── Blobfish ──
    function createBlobfish(parent, x, y, z) {
      const C = { skin: '#F4B8C2', nose: '#F3AEBB', fin: '#C97A8D', mouth: '#6B3A45', tail: '#D88A9D' };
      const blob = new Zdog.Anchor({ addTo: parent, scale: 2 * BLOB_SCALE, translate: { x, y, z } });

      new Zdog.Shape({ addTo: blob, stroke: 260 * BLOB_SCALE, color: C.skin });
      new Zdog.Shape({ addTo: blob, stroke: 180 * BLOB_SCALE, color: C.skin, translate: { y: -40, z: 6 } });
      new Zdog.Shape({ addTo: blob, stroke: 100 * BLOB_SCALE, color: C.nose, translate: { y: 14, z: 62 } });
      new Zdog.Shape({ addTo: blob, stroke: 120 * BLOB_SCALE, color: C.skin, translate: { x: -38, y: 24, z: 18 } });
      new Zdog.Shape({ addTo: blob, stroke: 120 * BLOB_SCALE, color: C.skin, translate: { x: 38, y: 24, z: 18 } });

      const eyeL = new Zdog.Anchor({ addTo: blob, translate: { x: -18, y: -14, z: 52 } });
      new Zdog.Shape({ addTo: eyeL, stroke: 30 * BLOB_SCALE, color: '#FFFFFF' });
      new Zdog.Shape({ addTo: eyeL, stroke: 12 * BLOB_SCALE, color: '#111111', translate: { z: 6 } });

      const eyeR = new Zdog.Anchor({ addTo: blob, translate: { x: 18, y: -14, z: 52 } });
      new Zdog.Shape({ addTo: eyeR, stroke: 30 * BLOB_SCALE, color: '#FFFFFF' });
      new Zdog.Shape({ addTo: eyeR, stroke: 12 * BLOB_SCALE, color: '#111111', translate: { z: 6 } });

      new Zdog.Shape({
        addTo: blob, stroke: 14 * BLOB_SCALE, color: C.mouth, closed: false,
        path: [{ x: -28, y: 40, z: 44 }, { bezier: [{ x: -12, y: 28, z: 54 }, { x: 12, y: 28, z: 54 }, { x: 28, y: 40, z: 44 }] }]
      });

      const finPath = [{ x: 0, y: 0 }, { x: -30, y: -10 }, { x: -50, y: 15 }, { x: -40, y: 35 }, { x: -10, y: 25 }];
      const leftPec = new Zdog.Anchor({ addTo: blob, translate: { x: -52, y: 22, z: -8 } });
      new Zdog.Shape({ addTo: leftPec, path: finPath, stroke: 24 * BLOB_SCALE, color: C.fin, fill: true });
      
      const rightPec = new Zdog.Anchor({ addTo: blob, translate: { x: 52, y: 22, z: -8 } });
      new Zdog.Shape({ addTo: rightPec, path: finPath, scale: { x: -1 }, stroke: 24 * BLOB_SCALE, color: C.fin, fill: true });

      const blobTail = new Zdog.Anchor({ addTo: blob, translate: { y: 10, z: -68 } });
      new Zdog.Shape({ addTo: blobTail, stroke: 52 * BLOB_SCALE, color: C.tail });
      const tailFin = new Zdog.Anchor({ addTo: blobTail, translate: { z: -14 } });
      new Zdog.Shape({ addTo: tailFin, stroke: 32 * BLOB_SCALE, color: C.fin, closed: false, path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -4 }, { x: -12, y: -12, z: -8 }, { x: 0, y: -18, z: -14 }] }] });

      return { blob, leftPec, rightPec, tailFin, eyeL, eyeR };
    }
    fish = createBlobfish(land, BLOB_POS.x, BLOB_POS.y, BLOB_POS.z);

    // ── Sloth ──
    function createSloth(parent) {
      const color = {
        fur: '#8B6F4E', furDark: '#6B5238', furLight: '#B89B72', faceCream: '#D8C29A',
        mask: '#5A4632', nose: '#3D2A1A', eye: '#2A1C12', cheek: '#E59A86', claw: '#4A3826'
      };

      const SLOTH_CONT = new Zdog.Anchor({ addTo: parent, translate: { x: -30, y: 200, z: -500 }, scale: SLOTH_SCALE });
      slothHitPos = { x: parent.translate.x - 30, y: parent.translate.y + 200, z: parent.translate.z - 500 };
      slothPivot = new Zdog.Anchor({ addTo: SLOTH_CONT });
      const SLOTH = new Zdog.Anchor({ addTo: slothPivot });

      new Zdog.Shape({ addTo: SLOTH, path: [{ x: -75, y: -4, z: 0 }, { bezier: [{ x: -30, y: 25, z: 0 }, { x: 30, y: 25, z: 0 }, { x: 75, y: -4, z: 0 }] }], stroke: 120, color: color.fur, closed: false });
      new Zdog.Shape({ addTo: SLOTH, path: [{ x: -40, y: -4, z: 0 }, { bezier: [{ x: -15, y: 8, z: 0 }, { x: 15, y: 8, z: 0 }, { x: 40, y: -4, z: 0 }] }], stroke: 40, color: color.furLight, translate: { y: -28, z: 0 }, closed: false });

      function makeLimb(baseX, baseY, baseZ, gripX, stroke) {
        const limb = new Zdog.Anchor({ addTo: SLOTH, translate: { x: baseX, y: baseY, z: baseZ } });
        const dx = gripX - baseX; const dy = -210 - baseY;
        new Zdog.Shape({ addTo: limb, path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: dx * 0.2, y: dy * 0.4, z: 0 }, { x: dx * 0.7, y: dy * 0.8, z: 0 }, { x: dx, y: dy, z: 0 }] }], stroke, color: color.fur, fill: false });
        const hand = new Zdog.Anchor({ addTo: limb, translate: { x: dx, y: dy, z: 0 } });
        for (let i = 0; i < 3; i++) {
          new Zdog.Shape({ addTo: hand, path: [{ x: (i - 1) * 6, y: 0 }, { bezier: [{ x: (i - 1) * 6, y: -6 }, { x: (i - 1) * 6 - 4, y: -12 }, { x: (i - 1) * 6 - 5, y: -16 }] }], stroke: 5, color: color.claw, closed: false });
        }
      }

      makeLimb(-60, -10,  46, -110, 34);
      makeLimb(-80, -10, -46, -170, 34);
      makeLimb( 60, -12,  46,   80, 34);
      makeLimb(100, -12, -46,  140, 34);

      const head = new Zdog.Anchor({ addTo: SLOTH, translate: { x: -140, y: 12, z: 14 }, scale: { y: -1 } });
      new Zdog.Shape({ addTo: head, stroke: 92, color: color.fur });
      new Zdog.Shape({ addTo: head, stroke: 46, color: color.faceCream, translate: { x: -30, y: 10, z: 14 } });
      new Zdog.Shape({ addTo: head, stroke: 16, color: color.nose, translate: { x: -54, y: 8, z: 14 } });

      const eye = new Zdog.Ellipse({ addTo: head, width: 14, height: 14, fill: true, stroke: 1, color: color.eye, translate: { x: -16, y: -2, z: 42 } });
      new Zdog.Ellipse({ addTo: eye, width: 14.5, height: 14.5, fill: true, stroke: 1, color: color.mask, translate: { z: 2 }, quarters: 4 });

      new Zdog.Shape({ addTo: head, path: [{ x: -30, y: -18 }, { x: -6, y: -22 }], stroke: 6, color: color.furDark, translate: { z: 44 } });
      new Zdog.Shape({ addTo: head, path: [{ x: 20, y: -16 }, { x: 36, y: -18 }],   stroke: 6, color: color.furDark, translate: { z: -30 } });
      new Zdog.Shape({ addTo: head, stroke: 3.5, color: color.nose, closed: false, path: [ { x: -46, y: 20, z: 20 }, { bezier: [{ x: -42, y: 23, z: 20 }, { x: -36, y: 23, z: 20 }, { x: -30, y: 20, z: 20 }] } ] });
    }
    createSloth(slothTree);

    // ── Chunky 3D Wooden Letter Signpost: "PAWSPECTIVE" ──
    function createSignpost(parent, sx, sy, sz) {
      const WoodLight = '#A3724C';
      const WoodDark  = '#784E2F';
      const TextColor = '#FFFFFF';

      const signGroup = new Zdog.Anchor({ addTo: parent, translate: { x: sx, y: sy, z: sz }, rotate: { y: 0 } });

      // Twin mounting support posts planted into the grass
      new Zdog.Cylinder({ addTo: signGroup, diameter: 24, length: 180, color: WoodDark, frontFace: WoodLight, stroke: false, rotate: { x: TAU/4 }, translate: { x: -250, y: -90, z: 0 } });
      new Zdog.Cylinder({ addTo: signGroup, diameter: 24, length: 180, color: WoodDark, frontFace: WoodLight, stroke: false, rotate: { x: TAU/4 }, translate: { x: 250, y: -90, z: 0 } });

      // Main thick billboard backplate block
      new Zdog.Box({
        addTo: signGroup,
        width: 760,
        height: 110,
        depth: 30,
        color: WoodLight,
        leftFace: WoodDark, rightFace: WoodDark, topFace: WoodDark, bottomFace: WoodDark,
        translate: { y: -160, z: 10 }
      });

      // Character layout vector path dictionary definitions
      const font = {
        P: [{x:0,y:20},{x:0,y:-20},{x:12,y:-20},{x:15,y:-14},{x:15,y:-6},{x:12,y:0},{x:0,y:0}],
        A: [{x:-14,y:20},{x:-3,y:-20},{x:3,y:-20},{x:14,y:20}, {move:{x:-8,y:4}},{x:8,y:4}],
        W: [{x:-15,y:-20},{x:-8,y:20},{x:0,y:-4},{x:8,y:20},{x:15,y:-20}],
        S: [{x:14,y:-14},{x:10,y:-20},{x:-10,y:-20},{x:-14,y:-12},{x:14,y:4},{x:10,y:20},{x:-10,y:20},{x:-14,y:12}],
        E: [{x:14,y:-20},{x:0,y:-20},{x:0,y:20},{x:14,y:20}, {move:{x:0,y:0}},{x:11,y:0}],
        C: [{x:14,y:-12},{x:10,y:-20},{x:-10,y:-20},{x:-14,y:0},{x:-10,y:20},{x:14,y:12}],
        T: [{x:-14,y:-20},{x:14,y:-20}, {move:{x:0,y:-20}},{x:0,y:20}],
        I: [{x:0,y:-20},{x:0,y:20}, {move:{x:-10,y:-20}},{x:10,y:-20}, {move:{x:-10,y:20}},{x:10,y:20}],
        V: [{x:-14,y:-20},{x:0,y:20},{x:14,y:-20}]
      };

      const word = ["P","A","W","S","P","E","C","T","I","V","E"];
      const spacing = 62; // Center offset multiplier step spacing per letter 
      const startX = -((word.length - 1) * spacing) / 2;

      word.forEach((char, index) => {
        const letterAnchor = new Zdog.Anchor({
          addTo: signGroup,
          translate: { x: startX + (index * spacing), y: -160, z: 26 }
        });

        // 3D Text extrusion stack layers
        for (let layer = 0; font[char] && layer < 4; layer++) {
          new Zdog.Shape({
            addTo: letterAnchor,
            path: font[char],
            closed: false,
            stroke: layer === 3 ? 7 : 9,
            color: layer === 3 ? TextColor : WoodDark,
            translate: { z: -layer * 2.5 }, // Increased extrusion depth
            backface: true                  // Ensure back is always rendered
          });
        }
      });
    }
    createSignpost(land, SIGN_POS.x, SIGN_POS.y, SIGN_POS.z);
  }

  // Projection Screen Positions
  function doveScreenPos() { const v = new Zdog.Vector(DOVE_POS); v.rotate(illo.rotate); const rect = canvas.getBoundingClientRect(); return { x: rect.width / 2 + illo.zoom * (v.x + illo.translate.x), y: rect.height / 2 + illo.zoom * (v.y + illo.translate.y) }; }
  function beeScreenPos() { const src = bee ? bee.cont.translate : BEE_POS; const v = new Zdog.Vector(src); v.rotate(illo.rotate); const rect = canvas.getBoundingClientRect(); return { x: rect.width / 2 + illo.zoom * (v.x + illo.translate.x), y: rect.height / 2 + illo.zoom * (v.y + illo.translate.y) }; }
  function chickenScreenPos() { const v = new Zdog.Vector(CHICKEN_POS); v.rotate(illo.rotate); const rect = canvas.getBoundingClientRect(); return { x: rect.width / 2 + illo.zoom * (v.x + illo.translate.x), y: rect.height / 2 + illo.zoom * (v.y + illo.translate.y) }; }
  function fishScreenPos() { const src = fish ? fish.blob.translate : BLOB_POS; const v = new Zdog.Vector(src); v.rotate(illo.rotate); const rect = canvas.getBoundingClientRect(); return { x: rect.width / 2 + illo.zoom * (v.x + illo.translate.x), y: rect.height / 2 + illo.zoom * (v.y + illo.translate.y) }; }
  function slothScreenPos() { if (!slothHitPos) return { x: -9999, y: -9999 }; const v = new Zdog.Vector(slothHitPos); v.rotate(illo.rotate); const rect = canvas.getBoundingClientRect(); return { x: rect.width / 2 + illo.zoom * (v.x + illo.translate.x), y: rect.height / 2 + illo.zoom * (v.y + illo.translate.y) }; }

  // Target Boundaries Hover Check
  function isOnFish(e) { if (!illo || !fish) return false; const rect = canvas.getBoundingClientRect(); const p = fishScreenPos(); return Math.hypot((e.clientX - rect.left) - p.x, (e.clientY - rect.top) - p.y) <= 120 * BLOB_SCALE * illo.zoom; }
  function isOnChicken(e) { if (!illo) return false; const rect = canvas.getBoundingClientRect(); const p = chickenScreenPos(); return Math.hypot((e.clientX - rect.left) - p.x, (e.clientY - rect.top) - p.y) <= 100 * CHICKEN_SCALE * illo.zoom; }
  function isOnDove(e) { if (!illo) return false; const rect = canvas.getBoundingClientRect(); const p = doveScreenPos(); return Math.hypot((e.clientX - rect.left) - p.x, (e.clientY - rect.top) - p.y) <= 80 * DOVE_SCALE * illo.zoom; }
  function isOnBee(e) { if (!illo || !bee) return false; const rect = canvas.getBoundingClientRect(); const p = beeScreenPos(); return Math.hypot((e.clientX - rect.left) - p.x, (e.clientY - rect.top) - p.y) <= 90 * BEE_SCALE * illo.zoom; }
  function isOnSloth(e) { if (!illo || !slothHitPos) return false; const rect = canvas.getBoundingClientRect(); const p = slothScreenPos(); return Math.hypot((e.clientX - rect.left) - p.x, (e.clientY - rect.top) - p.y) <= 160 * illo.zoom; }

  // Locked center zooming (no panning allowed)
  function handleWheel(e) {
    e.preventDefault();
    const factor = e.deltaY < 0 ? ZOOM_STEP : 1 / ZOOM_STEP;
    targetZoom = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, targetZoom * factor));
  }

  function handlePointerDown(e) {
    downX = e.clientX; 
    downY = e.clientY; 
    prevDragX = e.clientX;
    spin = false;
    canvas.style.cursor = 'grabbing';
  }

  function handlePointerUp(e) {
    if (canvas) canvas.style.cursor = 'grab';
    const moved = Math.hypot(e.clientX - downX, e.clientY - downY);

    if (moved < 6) {
      if (isOnDove(e))         onSelectDove?.();
      else if (isOnBee(e))     onSelectBee?.();
      else if (isOnChicken(e)) onSelectChicken?.();
      else if (isOnFish(e))    onSelectFish?.();
      else if (isOnSloth(e))   onSelectSloth?.();
    }
  }

  function handlePointerMove(e) {
    if (!canvas) return;

    // Custom single-axis drag mapping (Y-axis turntable spin restricted to horizontal mouse drags)
    if (canvas.style.cursor === 'grabbing') {
      let deltaX = e.clientX - prevDragX;
      illo.rotate.y += deltaX * 0.008; 
      prevDragX = e.clientX;
      return;
    }
    
    canvas.style.cursor = (isOnDove(e) || isOnBee(e) || isOnChicken(e) || isOnFish(e) || isOnSloth(e)) ? 'pointer' : 'grab';
  }

  onMount(() => {
    illo = new Zdog.Illustration({
      element: canvas,
      dragRotate: false, // Disabled to lock axis
      resize: 'window',
      zoom: START_ZOOM,
      rotate: { x: -0.1, y: TAU / 10 },
    });

    buildScene(illo);

    function tick() {
      rafId = requestAnimationFrame(tick);
      if (spin) illo.rotate.y += SPIN_SPEED;

      wingBeat += 0.13;
      if (dove) {
        dove.frontWing.rotate.z = Math.sin(wingBeat) * 0.18;
        dove.backWing.rotate.z  = Math.sin(wingBeat - 0.5) * 0.18;
      }

      beeFrame++;
      if (bee) {
        const f = beeFrame;
        bee.rightWing.rotate.z = -TAU / 6 + (TAU / 10) * Math.sin(f / 0.9);
        bee.leftWing.rotate.z  =  TAU / 6 - (TAU / 10) * Math.sin(f / 0.9);
        bee.antler.rotate.x    = (TAU / 40) * Math.sin(f / 10);
        bee.cont.translate.y   = BEE_POS.y + Math.sin(f / 14) * 2; 
        bee.cont.translate.x   = BEE_POS.x + Math.cos(f / 28) * 2;
      }

      if (chickenHandle) {
        const blinkCycle = beeFrame % 340; 
        const sY = (blinkCycle > 320) ? 0.5 + 0.5 * Math.cos(((blinkCycle - 320) / 20) * TAU) : 1;
        chickenHandle.eyeL.scale.y = sY; chickenHandle.eyeR.scale.y = sY;
      }

      if (fish) {
        const f = beeFrame; 
        fish.blob.translate.y = BLOB_POS.y + Math.sin(f * 0.035) * 14;
        fish.leftPec.rotate.z = Math.sin(f / 22) * 0.14;
        fish.rightPec.rotate.z = -Math.sin(f / 22) * 0.14;
        fish.tailFin.rotate.y = Math.sin(f * 0.07) * 0.18;
        const blink = f % 280;
        let s_blink = (blink > 262) ? 0.5 + 0.5 * Math.cos(((blink - 262) / 18) * Math.PI * 2) : 1;
        fish.eyeL.scale.y = s_blink; fish.eyeR.scale.y = s_blink;
      }

      if (slothPivot) {
        slothPivot.rotate.z = Math.sin(beeFrame * 0.02) * 0.08; 
      }

      // Smooth zoom execution, locked precisely on 0,0,0 coordinates
      illo.translate = { x: 0, y: 0, z: 0 };
      illo.zoom = lerp(illo.zoom, targetZoom, ZOOM_EASE);
      
      illo.updateRenderGraph();
    }
    tick();

    canvas.addEventListener('wheel', handleWheel, { passive: false });
    window.addEventListener('pointerup', handlePointerUp);
  });

  onDestroy(() => {
    cancelAnimationFrame(rafId);
    if (canvas) canvas.removeEventListener('wheel', handleWheel);
    if (typeof window !== 'undefined') window.removeEventListener('pointerup', handlePointerUp);
  });
</script>

<canvas
  bind:this={canvas}
  on:pointerdown={handlePointerDown}
  on:pointermove={handlePointerMove}
  class="land-canvas"
></canvas>

<style>
  :global(html, body) { height: 100%; margin: 0; overflow: hidden; }
  .land-canvas {
    position: fixed; left: 0; top: 0; width: 100vw; height: 100vh;
    background: #FFD9EC; cursor: grab; touch-action: none; display: block; border: none;
  }
</style>