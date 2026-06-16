<script>
//@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';

  let canvasRef;
  let uiCanvasRef;
  let overlayCanvasRef; 
  let wrapperRef; 
  
  let { embedded = false } = $props();
  
  // --- Environment & State Management ---
  let isDeep = false;
  let isAnimating = false;
  let hintText = 'Double click the fish to jump • Click the gauge to dive';

  let fishState = { pinkAlpha: 1, darkAlpha: 0 };
  let transitionProgress = { value: 0 }; 

  // --- Zdog Global Targets ---
  let scene, uiScene;
  let backgroundZone, sceneryZone, fishZone; 
  let blobMasterAnchor, handsomeMasterAnchor;
  let leftPecContainer, rightPecContainer;
  let leftWingContainer, rightWingContainer;
  let handsomeTailFin, blobTailFin;
  let eyeL_pink, eyeR_pink, eyeL_blue, eyeR_blue;
  let mouth_pink, mouth_blue; 
  let gaugeNeedle;

  // --- Vector Background Blobs ---
  let skyBlock, midBlock, midTransitionBlock, deepTransitionBlock, abyssBlock;

  // --- Drifting Scenery Anchors ---
  let cloud1, cloud2, cloud3;

  let pinkFadeShapes = [];
  let darkFadeShapes = [];
  let sceneryFadeShapes = [];

  // --- UI Positioning States ---
  let clickCoords = { x: 0, y: 0 }; 
  let fishScreenCoords = { x: 0, y: 0 }; 

  let radialMenuOpen = false;
  let activeCard = null;
  let historyOpen = false;
  
  let thoughtBubbleVisible = false;
  let isReplying = false; 
  let thoughtBubbleText = '';
  let replyInputValue = '';
  let thoughtTimer = null;
  let bubbleAutoHideTimer = null;
  let isApiLoading = false;
  let conversationActive = false;

  // Gesture Tracking for Double Tap
  let lastClickTime = 0;
  let clickTimeout;

  const interactionCards = [
    {
      id: 'float',
      icon: '🔄',
      label: 'Float Around',
      title: 'Orbit Swimming',
      description: 'Command the blobfish to drift passively in a large horizontal tracking orbit.',
      hint: 'The fish will drift in a smooth, sweeping horizontal circle.',
      angle: -90
    },
    {
      id: 'jump',
      icon: '🐬',
      label: 'Jump',
      title: 'Breaching Leap',
      description: 'Watch the fish break dynamics to launch an airborne trajectory arc.',
      hint: 'Jumps directly if shallow. If deep, it surfaces rapidly before launching.',
      angle: 30
    },
    {
      id: 'chat',
      icon: '💬',
      label: 'Chat',
      title: 'Consult the Blobfish',
      description: 'Uncover deep sub-surface wisdom regarding atmospheric pressure and inner structural integrity.',
      hint: 'Reply to the active bubble thoughts below.',
      angle: 150
    }
  ];

  let chatHistory = [
    { role: 'assistant', content: "Mmmh... Hello down there. Let us talk about surviving the heavy pressures of life, or simply floating along." }
  ];

  const thoughtPrompts = [
    'The pressure changes up here... they alter how everyone looks at you.',
    'Down in the dark, you never need to worry about structural integrity.',
    'Floating takes absolutely no effort at all. There is a great wisdom in that.',
    'Heavy weights are easier to bear when you are built exactly for the depth you inhabit.',
    'Are you running around today, or are you letting the currents carry you?',
    'Sometimes, looking a bit deflated just means you are away from home.'
  ];

  // --- Background Seaweed Array ---
  const seaweedStrands = [
    { xPct: 0.08, baseWidth: 22, height: 320, speed: 0.024, delay: 0.0 },
    { xPct: 0.11, baseWidth: 35, height: 460, speed: 0.018, delay: 0.8 },
    { xPct: 0.14, baseWidth: 26, height: 380, speed: 0.022, delay: 0.4 },
    { xPct: 0.17, baseWidth: 40, height: 490, speed: 0.015, delay: 1.2 },
    { xPct: 0.21, baseWidth: 28, height: 340, speed: 0.026, delay: 0.2 },
    { xPct: 0.25, baseWidth: 32, height: 410, speed: 0.020, delay: 0.9 },
    { xPct: 0.72, baseWidth: 30, height: 390, speed: 0.021, delay: 1.5 },
    { xPct: 0.76, baseWidth: 42, height: 510, speed: 0.016, delay: 0.3 },
    { xPct: 0.80, baseWidth: 24, height: 350, speed: 0.025, delay: 1.0 },
    { xPct: 0.84, baseWidth: 38, height: 470, speed: 0.019, delay: 0.6 },
    { xPct: 0.88, baseWidth: 28, height: 420, speed: 0.023, delay: 1.4 },
    { xPct: 0.93, baseWidth: 20, height: 310, speed: 0.028, delay: 0.1 }
  ];

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
    waterTransition: '#1C2541',
    seaweedDark: '26, 54, 39',   
    seaweedLight: '53, 94, 59',
    
    // Multi-tier solid color ecosystem
    bgSky: '#ff8fae',
    bgPinkWater: '#FAD0C4',
    bgMidWater: '#4A90E2',
    bgTransBlue1: '#3B78C4', 
    bgTransBlue2: '#2B5FA6',
    bgDeepWater: '#1C2541',
    bgAbyss: '#0B132B',

    // Scenery color configurations
    landGreen: '#6CA35E',
    landGreenDark: '#4E8241',
    cloudBody: 'rgba(255, 255, 255, 0.75)'
  };

  function toggleRadialMenu(clientX, clientY) {
    radialMenuOpen = !radialMenuOpen;
    if (radialMenuOpen) {
      clickCoords = { x: clientX, y: clientY };
      activeCard = null; 
      historyOpen = false;
    }
  }

  function openCard(id) {
    activeCard = interactionCards.find(c => c.id === id);
    radialMenuOpen = false;
  }

  function closeCard() { activeCard = null; }

  function executeCard() {
    if (!activeCard) return;
    if (activeCard.id === 'float') triggerFloatAnimation();
    if (activeCard.id === 'jump') triggerJumpAnimation();
    if (activeCard.id === 'chat') {
      activeCard = null;
      openReplyInput(); 
      return;
    }
    activeCard = null;
  }

  function toggleHistory() {
    historyOpen = !historyOpen;
    if (historyOpen) { radialMenuOpen = false; activeCard = null; }
  }

  function updateBubblePosition() {
    if (!canvasRef || !fishZone) return;
    const rect = canvasRef.getBoundingClientRect();
    
    const centerX = rect.left + rect.width / 2;
    const centerY = rect.top + rect.height / 2;
    
    fishScreenCoords = {
      x: centerX + fishZone.translate.x,
      y: centerY + fishZone.translate.y
    };
  }

  function showThought(text, keepAlive = false) {
    if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
    thoughtBubbleText = text;
    thoughtBubbleVisible = true;
    conversationActive = keepAlive;
    isReplying = false;
    updateBubblePosition();

    if (!keepAlive && !isReplying) {
      bubbleAutoHideTimer = setTimeout(() => {
        thoughtBubbleVisible = false;
      }, 12000);
    }
  }

  function openReplyInput() {
    if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
    isReplying = true;
    updateBubblePosition();
  }

  function scheduleThought() {
    if (thoughtTimer) clearTimeout(thoughtTimer);
    thoughtTimer = setTimeout(() => {
      if (!thoughtBubbleVisible && !isReplying && !isApiLoading) {
        const prompt = thoughtPrompts[Math.floor(Math.random() * thoughtPrompts.length)];
        showThought(prompt);
      }
      scheduleThought();
    }, 18000 + Math.random() * 10000);
  }

  async function sendReply() {
    const msg = replyInputValue.trim();
    if (!msg || isApiLoading) return;
    replyInputValue = '';
    isReplying = false;
    thoughtBubbleVisible = false;
    await askFish(msg);
  }

  function handleReplyKey(e) {
    if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); sendReply(); }
  }

  async function askFish(userMessage) {
    if (isApiLoading) return;
    isApiLoading = true;
    showThought('…', true);
    triggerTalkingMotion();

    chatHistory = [...chatHistory, { role: 'user', content: userMessage }];

    try {
      const endpoint = 'https://api.openai.com/v1/chat/completions';
      const body = {
        model: 'gpt-4o-mini',
        messages: [
          { role: 'system', content: 'You are a wise blobfish' },
          ...chatHistory
        ],
        max_tokens: 256,
        temperature: 0.85
      };
      const res = await fetch(endpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${OPENAI_API_KEY}`
        },
        body: JSON.stringify(body)
      });

      if (!res.ok) throw new Error(`API error code: ${res.status}`);

      const data = await res.json();
      const replyText = data?.choices?.[0]?.message?.content;

      if (replyText) {
        const cleanReply = replyText.trim();
        chatHistory = [...chatHistory, { role: 'assistant', content: cleanReply }];
        showThought(cleanReply, true);
      } else {
        showThought("The currents are scrambled... I lost my thread of thought.", true);
      }
    } catch (e) {
      console.error(e);
      chatHistory = chatHistory.slice(0, -1);
      showThought("Too much static in the water right now. Ask me again shortly...", true);
    } finally {
      isApiLoading = false;
    }
  }

  // --- Animation Execution Methods ---
  let isMeshRunningAnimation = false;
  let customFinSpeedFactor = 1;
  let customTailSpeedFactor = 1;
  let expressionModifierActive = false;

  function triggerFloatAnimation() {
    if (isMeshRunningAnimation) return;
    isMeshRunningAnimation = true;
    customTailSpeedFactor = 2.0;

    const tl = gsap.timeline({
      onComplete: () => {
        isMeshRunningAnimation = false;
        customTailSpeedFactor = 1;
        fishZone.translate.x = 0;
        fishZone.translate.z = 0;
        fishZone.rotate.y = -0.12;
      }
    });

    tl.to(fishZone.rotate, { duration: 3.5, y: `-=${Zdog.TAU}`, ease: 'none' });
    tl.to(fishZone.translate, { 
      duration: 1.75, 
      x: -160, 
      z: -100, 
      ease: 'sine.inOut', 
      yoyo: true, 
      repeat: 1 
    }, 0);
  }

  function triggerJumpAnimation() {
    if (isMeshRunningAnimation) return;
    isMeshRunningAnimation = true;
    customFinSpeedFactor = 3.5;

    const tl = gsap.timeline({
      onComplete: () => {
        isMeshRunningAnimation = false;
        customFinSpeedFactor = 1;
        fishZone.translate.y = 0;
        fishZone.translate.z = 0;
        fishZone.rotate.x = 0;
      }
    });

    if (isDeep) {
      tl.to(fishZone.translate, { duration: 1.2, y: -260, z: 40, ease: 'power2.out' });
      tl.to(fishZone.rotate, { duration: 0.8, x: -0.4 }, 0);
      
      tl.to(fishZone.translate, { duration: 0.5, y: -500, z: 120, ease: 'power1.out' }, '+=0.1');
      tl.to(fishZone.rotate, { duration: 1.0, x: `+=${Zdog.TAU}`, ease: 'power1.inOut' }, '-=0.2');
      tl.to(fishZone.translate, { duration: 0.6, y: 0, z: 0, ease: 'power2.in' });
    } else {
      tl.to(fishZone.translate, { duration: 0.5, y: -240, z: 80, ease: 'power1.out' });
      tl.to(fishZone.rotate, { duration: 0.9, x: `+=${Zdog.TAU}`, ease: 'power1.inOut' }, 0);
      tl.to(fishZone.translate, { duration: 0.5, y: 0, z: 0, ease: 'power1.in' }, '-=0.4');
    }
  }

  function triggerTalkingMotion() {
    const tl = gsap.timeline();
    tl.to([leftPecContainer.rotate, leftWingContainer.rotate], { duration: 0.3, y: 0.4, ease: 'sine.inOut', repeat: 3, yoyo: true });
    tl.to([rightPecContainer.rotate, rightWingContainer.rotate], { duration: 0.3, y: -0.4, ease: 'sine.inOut', repeat: 3, yoyo: true }, 0);
  }

  // --- Core Mechanical Helpers ---
  function hexToRgb(hex) {
    if(hex.startsWith('rgba')) return { r:255, g:255, b:255 }; 
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

  // --- Canvas Pointer Action Hit Detection ---
  let isDragging = false, lastX = 0, lastY = 0, wasDragging = false;
  const sensitivity = 0.008;

  function onPointerDown(e) {
    isDragging = true; 
    wasDragging = false;
    lastX = e.clientX; 
    lastY = e.clientY;
    try { canvasRef.setPointerCapture(e.pointerId); } catch (_) {}
  }

  function onPointerMove(e) {
    if (!isDragging) return;
    const dx = e.clientX - lastX, dy = e.clientY - lastY;
    lastX = e.clientX; 
    lastY = e.clientY;
    if (Math.hypot(dx, dy) > 8) { wasDragging = true; }
    
    fishZone.rotate.y += dx * sensitivity;
    fishZone.rotate.x += dy * sensitivity;
    fishZone.rotate.x  = Math.max(-Zdog.TAU / 8, Math.min(Zdog.TAU / 8, fishZone.rotate.x));
  }

  function onPointerUp() { isDragging = false; }

  function onCanvasClick(e) {
    if (wasDragging) return;
    const rect = canvasRef.getBoundingClientRect();
    const cx = (e.clientX - rect.left - rect.width / 2);
    const cy = (e.clientY - rect.top - rect.height / 2);

    const fishRadiusThreshold = 145; 
    if (Math.hypot(cx, cy) < fishRadiusThreshold) {
      const currentTime = Date.now();
      
      // Check for Double Click / Double Tap (interval < 300ms)
      if (currentTime - lastClickTime < 300) {
        clearTimeout(clickTimeout);
        radialMenuOpen = false;
        triggerJumpAnimation();
      } else {
        // Queue single click menu switch safely to rule out double tap intents
        clearTimeout(clickTimeout);
        clickTimeout = setTimeout(() => {
          toggleRadialMenu(fishScreenCoords.x, fishScreenCoords.y);
        }, 250);
      }
      lastClickTime = currentTime;
    }
  }

  onMount(() => {
    const ctx = overlayCanvasRef.getContext('2d');
    function resizeOverlay() {
      overlayCanvasRef.width = window.innerWidth;
      overlayCanvasRef.height = window.innerHeight;
      updateBubblePosition();
    }
    resizeOverlay();
    window.addEventListener('resize', resizeOverlay);

    // SCENE GENERATION
    scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: 0, y: -0.12 }, 
    });

    // Layer 1: Background Sky Vector Objects Group
    backgroundZone = new Zdog.Anchor({ addTo: scene, translate: { z: -450 } });

    skyBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [
        { x: -1200, y: -900 },
        { x: 1200, y: -900 },
        { x: 1200, y: -260 },
        { arc: [ { x: 0, y: -200 }, { x: -1200, y: -260 } ] }
      ],
      stroke: 40,
      color: color.bgSky,
      fill: true
    });

    // Layer 1.5: Clouds high up in the sky space
    sceneryZone = new Zdog.Anchor({ addTo: scene, translate: { z: -420 } });

    cloud1 = new Zdog.Anchor({ addTo: sceneryZone, translate: { x: -350, y: -450 } });
    new Zdog.Shape({ addTo: cloud1, stroke: 50, color: color.cloudBody });
    new Zdog.Shape({ addTo: cloud1, stroke: 36, color: color.cloudBody, translate: { x: -30, y: 8 } });
    new Zdog.Shape({ addTo: cloud1, stroke: 40, color: color.cloudBody, translate: { x: 34, y: 4 } });

    cloud2 = new Zdog.Anchor({ addTo: sceneryZone, translate: { x: 180, y: -520 } });
    new Zdog.Shape({ addTo: cloud2, stroke: 64, color: color.cloudBody });
    new Zdog.Shape({ addTo: cloud2, stroke: 46, color: color.cloudBody, translate: { x: -44, y: 10 } });
    new Zdog.Shape({ addTo: cloud2, stroke: 42, color: color.cloudBody, translate: { x: 44, y: 6 } });

    cloud3 = new Zdog.Anchor({ addTo: sceneryZone, translate: { x: 500, y: -380 } });
    new Zdog.Shape({ addTo: cloud3, stroke: 38, color: color.cloudBody });
    new Zdog.Shape({ addTo: cloud3, stroke: 28, color: color.cloudBody, translate: { x: -24, y: 4 } });

    // Islands preserved right where they are
    let leftIsland = new Zdog.Shape({
      addTo: sceneryZone,
      path: [
        { x: -800, y: 20 },
        { arc: [ { x: -500, y: -200 }, { x: -200, y: 20 } ] },
        { x: -200, y: 20 },
        { x: -800, y: 20 }
      ],
      stroke: 20,
      color: color.landGreenDark,
      fill: true
    });

    let rightIsland = new Zdog.Shape({
      addTo: sceneryZone,
      path: [
        { x: 200, y: 20 },
        { arc: [ { x: 500, y: -240 }, { x: 800, y: 20 } ] },
        { x: 800, y: 20 },
        { x: 200, y: 20 }
      ],
      stroke: 20,
      color: color.landGreenDark,
      fill: true
    });

    // Layer 1.8: Water Volumes Group
    midBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [
        { x: -1200, y: -280 },
        { arc: [ { x: 0, y: -220 }, { x: 1200, y: -280 } ] },
        { x: 1200, y: 50 },
        { arc: [ { x: 0, y: 110 }, { x: -1200, y: 50 } ] }
      ],
      stroke: 50,
      color: color.bgPinkWater,
      fill: true
    });

    midTransitionBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [
        { x: -1200, y: 30 },
        { arc: [ { x: 0, y: 90 }, { x: 1200, y: 30 } ] },
        { x: 1200, y: 280 },
        { arc: [ { x: 0, y: 340 }, { x: -1200, y: 280 } ] }
      ],
      stroke: 50,
      color: color.bgMidWater,
      fill: true
    });

    deepTransitionBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [
        { x: -1200, y: 260 },
        { arc: [ { x: 0, y: 320 }, { x: 1200, y: 260 } ] },
        { x: 1200, y: 540 },
        { arc: [ { x: 0, y: 600 }, { x: -1200, y: 540 } ] }
      ],
      stroke: 50,
      color: color.bgTransBlue1,
      fill: true
    });

      abyssBlock = new Zdog.Shape({
        addTo: backgroundZone,
        path: [
          { x: -1200, y: 520 },
          { arc: [ { x: 0, y: 580 }, { x: 1200, y: 520 } ] },
          { x: 1200, y: 2500 }, // Increased from 1200 to 2500 to cover lower viewports
          { x: -1200, y: 2500 } // Increased from 1200 to 2500
        ],
        stroke: 50,
        color: color.bgTransBlue2,
        fill: true
      });

    // Layer 2: Fish Interactive Space Group
    fishZone = new Zdog.Anchor({ addTo: scene });

    // --- Blobfish (Surface - Pink) ---
    blobMasterAnchor = new Zdog.Anchor({ addTo: fishZone, scale: 2 });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 260, color: color.blobSkin });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 180, color: color.blobSkin, translate: { y: -40, z: 6 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 100, color: color.blobNose, translate: { y: 14, z: 62 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 120, color: color.blobSkin, translate: { x: -38, y: 24, z: 18 } });
    new Zdog.Shape({ addTo: blobMasterAnchor, stroke: 120, color: color.blobSkin, translate: { x: 38, y: 24, z: 18 } });

    eyeL_pink = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { x: -18, y: -14, z: 52 } });
    new Zdog.Shape({ addTo: eyeL_pink, stroke: 30, color: '#FFFFFF' });
    new Zdog.Shape({ addTo: eyeL_pink, stroke: 12, color: '#111111', translate: { z: 6 } });

    eyeR_pink = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { x: 18, y: -14, z: 52 } });
    new Zdog.Shape({ addTo: eyeR_pink, stroke: 30, color: '#FFFFFF' });
    new Zdog.Shape({ addTo: eyeR_pink, stroke: 12, color: '#111111', translate: { z: 6 } });

    mouth_pink = new Zdog.Shape({
      addTo: blobMasterAnchor,
      stroke: 14,
      color: color.blobMouth,
      closed: false,
      path: [{ x: -28, y: 40, z: 44 }, { bezier: [{ x: -12, y: 28, z: 54 }, { x: 12, y: 28, z: 54 }, { x: 28, y: 40, z: 44 }] }]
    });

    const finShapePath = [{ x: 0, y: 0 }, { x: -30, y: -10 }, { x: -50, y: 15 }, { x: -40, y: 35 }, { x: -10, y: 25 }];

    leftPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    const leftPecAnchor = new Zdog.Anchor({ addTo: leftPecContainer, translate: { x: -52, y: 22, z: -8 } });
    new Zdog.Shape({ addTo: leftPecAnchor, path: finShapePath, stroke: 24, color: color.blobFin, fill: true });

    rightPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    const rightPecAnchor = new Zdog.Anchor({ addTo: rightPecContainer, translate: { x: 52, y: 22, z: -8 } });
    new Zdog.Shape({ addTo: rightPecAnchor, path: finShapePath, scale: { x: -1 }, stroke: 24, color: color.blobFin, fill: true });

    const blobTail = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { y: 10, z: -68 } });
    new Zdog.Shape({ addTo: blobTail, stroke: 52, color: color.blobTailBody });
    blobTailFin = new Zdog.Anchor({ addTo: blobTail, translate: { z: -14 } });
    new Zdog.Shape({ addTo: blobTailFin, stroke: 32, color: color.blobFin, closed: false, path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -4 }, { x: -12, y: -12, z: -8 }, { x: 0, y: -18, z: -14 }] }] });

    // --- Handsome Blobfish (Deep - Blue) ---
    handsomeMasterAnchor = new Zdog.Anchor({ addTo: fishZone, scale: 3 });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 240, color: color.blobDeepSea });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 190, color: color.blobDeepSea, translate: { z: -30 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 120, color: color.blobDeepSea, translate: { x: -28, y: 14, z: 12 } });
    new Zdog.Shape({ addTo: handsomeMasterAnchor, stroke: 120, color: color.blobDeepSea, translate: { x: 28, y: 14, z: 12 } });

    eyeL_blue = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { x: -22, y: -12, z: 25 } });
    new Zdog.Shape({ addTo: eyeL_blue, stroke: 32, color: color.blobEyeDeep });
    new Zdog.Shape({ addTo: eyeL_blue, stroke: 14, color: color.blobPupilDeep, translate: { z: 6 } });

    eyeR_blue = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { x: 22, y: -12, z: 25 } });
    new Zdog.Shape({ addTo: eyeR_blue, stroke: 32, color: color.blobEyeDeep });
    new Zdog.Shape({ addTo: eyeR_blue, stroke: 14, color: color.blobPupilDeep, translate: { z: 6 } });

    mouth_blue = new Zdog.Shape({
      addTo: handsomeMasterAnchor,
      stroke: 11,
      color: color.blobMouthDeep,
      closed: false,
      path: [{ x: -18, y: 10, z: 25 }, { x: 0, y: 14, z: 27 }, { x: 18, y: 10, z: 25 }]
    });

    leftWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    const leftWingFin = new Zdog.Anchor({ addTo: leftWingContainer, translate: { x: -48, y: 22, z: -8 } });
    new Zdog.Shape({ addTo: leftWingFin, path: finShapePath, scale: { x: 0.666, y: 0.666, z: 0.666 }, stroke: 16, color: color.blobDeepSeaDark, fill: true });

    rightWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    const rightWingFin = new Zdog.Anchor({ addTo: rightWingContainer, translate: { x: 48, y: 22, z: -8 } });
    new Zdog.Shape({ addTo: rightWingFin, path: finShapePath, scale: { x: -0.666, y: 0.666, z: 0.666 }, stroke: 16, color: color.blobDeepSeaDark, fill: true });

    const handsomeTail = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { y: -4, z: -72 } });
    new Zdog.Shape({ addTo: handsomeTail, stroke: 60, color: color.blobDeepSea });
    handsomeTailFin = new Zdog.Anchor({ addTo: handsomeTail, translate: { z: -14 } });
    new Zdog.Shape({ addTo: handsomeTailFin, stroke: 80, color: color.blobDeepSeaDark, closed: false, path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -5 }, { x: -14, y: -14, z: -9 }, { x: 0, y: -22, z: -14 }] }] });

    pinkFadeShapes = setupFadeGroup(blobMasterAnchor);
    darkFadeShapes = setupFadeGroup(handsomeMasterAnchor);
    sceneryFadeShapes = setupFadeGroup(sceneryZone);

    // FIXED UI GAUGE
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
    scheduleThought();

    // RENDERING MECHANICS LOOP
    let frame = 0;
    let running = true;

    function drawSeaweed() {
      if (fishState.darkAlpha < 0.01) return; 
      const w = overlayCanvasRef.width;
      const h = overlayCanvasRef.height;

      seaweedStrands.forEach((strand, idx) => {
        const baseX = w * strand.xPct;
        const currentHeight = strand.height * fishState.darkAlpha;
        const greenTone = idx % 2 === 0 ? color.seaweedDark : color.seaweedLight;

        ctx.save();
        ctx.beginPath();
        ctx.fillStyle = `rgba(${greenTone}, ${0.65 * fishState.darkAlpha})`;

        const segments = 7;
        let leftSidePoints = [];
        let rightSidePoints = [];

        for (let i = 0; i <= segments; i++) {
          const segmentPct = i / segments;
          const yPos = h + 30 - (currentHeight * segmentPct);
          const waveFactor = Math.sin((frame * strand.speed) + (segmentPct * Math.PI * 1.2) + strand.delay);
          const xOffset = waveFactor * (25 * segmentPct);
          const centerX = baseX + xOffset;
          const currentWidth = strand.baseWidth * (1 - segmentPct);

          leftSidePoints.push({ x: centerX - currentWidth / 2, y: yPos });
          rightSidePoints.unshift({ x: centerX + currentWidth / 2, y: yPos });
        }

        ctx.moveTo(leftSidePoints[0].x, leftSidePoints[0].y);
        leftSidePoints.forEach(pt => ctx.lineTo(pt.x, pt.y));
        rightSidePoints.forEach(pt => ctx.lineTo(pt.x, pt.y));
        ctx.closePath();
        ctx.fill();
        ctx.restore();
      });
    }

    function drawWaterTransition() {
      ctx.clearRect(0, 0, overlayCanvasRef.width, overlayCanvasRef.height);
      drawSeaweed();

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

      if (!isMeshRunningAnimation) {
        fishZone.translate.y = Math.sin(frame * 0.035) * 14;
        fishZone.rotate.z = Math.cos(frame * 0.025) * 0.04;
      }

      cloud1.translate.x += 0.28; if (cloud1.translate.x > 950) cloud1.translate.x = -950;
      cloud2.translate.x += 0.16; if (cloud2.translate.x > 950) cloud2.translate.x = -950;
      cloud3.translate.x += 0.22; if (cloud3.translate.x > 950) cloud3.translate.x = -950;

      backgroundZone.rotate.y = -scene.rotate.y;
      backgroundZone.rotate.x = -scene.rotate.x;
      sceneryZone.rotate.y = -scene.rotate.y;
      sceneryZone.rotate.x = -scene.rotate.x;

      updateBubblePosition();

      pinkFadeShapes.forEach(({ node, r, g, b }) => {
        node.color = `rgba(${r},${g},${b},${fishState.pinkAlpha})`;
      });
      darkFadeShapes.forEach(({ node, r, g, b }) => {
        node.color = `rgba(${r},${g},${b},${fishState.darkAlpha})`;
      });

      sceneryFadeShapes.forEach(({ node, r, g, b }) => {
        node.color = `rgba(${r},${g},${b},${fishState.pinkAlpha})`;
      });
      
      cloud1.children.forEach(c => c.color = `rgba(255,255,255,${0.75 * fishState.pinkAlpha})`);
      cloud2.children.forEach(c => c.color = `rgba(255,255,255,${0.75 * fishState.pinkAlpha})`);
      cloud3.children.forEach(c => c.color = `rgba(255,255,255,${0.75 * fishState.pinkAlpha})`);

      blobMasterAnchor.visible = fishState.pinkAlpha > 0.01;
      handsomeMasterAnchor.visible = fishState.darkAlpha > 0.01;
      sceneryZone.visible = fishState.pinkAlpha > 0.01;

      leftPecContainer.rotate.z = Math.sin(frame / (22 / customFinSpeedFactor)) * 0.14;
      rightPecContainer.rotate.z = -Math.sin(frame / (22 / customFinSpeedFactor)) * 0.14;
      leftWingContainer.rotate.z = Math.sin(frame / (26 / customFinSpeedFactor)) * 0.11;
      rightWingContainer.rotate.z = -Math.sin(frame / (26 / customFinSpeedFactor)) * 0.11;

      if (blobTailFin) blobTailFin.rotate.y = Math.sin(frame * (0.07 * customTailSpeedFactor)) * 0.18;
      if (handsomeTailFin) handsomeTailFin.rotate.y = Math.sin(frame * (0.06 * customTailSpeedFactor)) * 0.16;

      const blink = frame % 280;
      let s_blink = 1;
      if (blink > 262) {
        s_blink = 0.5 + 0.5 * Math.cos(((blink - 262) / 18) * Math.PI * 2);
      }
      
      if (!expressionModifierActive) {
        eyeL_pink.scale.y = s_blink;
        eyeR_pink.scale.y = s_blink;
        eyeL_blue.scale.y = s_blink;
        eyeR_blue.scale.y = s_blink;
      }

      scene.updateRenderGraph();
      uiScene.updateRenderGraph();
      drawWaterTransition();
      
      requestAnimationFrame(render);
    }

    render();

    return () => { 
      running = false; 
      if (thoughtTimer) clearTimeout(thoughtTimer);
      if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
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
        hintText = isDeep ? 'Click the gauge to surface' : 'Double click the fish to jump • Click the gauge to dive';
      }
    });

    tl.to([skyBlock.translate, midBlock.translate, midTransitionBlock.translate, deepTransitionBlock.translate, abyssBlock.translate, sceneryZone.translate], {
      duration: 2.5,
      y: isDeep ? -850 : 0,
      ease: 'power2.inOut'
    }, 0);

    tl.to({}, {
      duration: 1.25,
      onComplete: () => {
        if (isDeep) {
          skyBlock.color = color.bgTransBlue1;
          midBlock.color = color.bgTransBlue2;
          midTransitionBlock.color = color.bgDeepWater;
          deepTransitionBlock.color = color.bgAbyss;
          abyssBlock.color = '#040714'; 
        } else {
          skyBlock.color = color.bgSky;
          midBlock.color = color.bgPinkWater;
          midTransitionBlock.color = color.bgMidWater;
          deepTransitionBlock.color = color.bgTransBlue1;
          abyssBlock.color = color.bgTransBlue2;
        }
      }
    }, 0.6);

    tl.to(gaugeNeedle.rotate, {
      duration: 2.5,
      z: isDeep ? Math.PI * 0.75 : -Math.PI * 0.75,
      ease: 'back.out(1.2)'
    }, 0);

    transitionProgress.value = 0;
    tl.to(transitionProgress, {
      duration: 1.4,
      value: 1,
      ease: 'power1.inOut'
    }, 0.55);

    tl.to({}, { duration: 0 }, 1.25); 
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

<style>
  @import url("https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap");
  
  /* Локальные стили для кнопок навигации */
  .custom-top-bar {
    position: absolute;
    top: 16px; 
    left: 16px;
    display: flex; 
    gap: 10px;
    z-index: 50;
  }

  .custom-bar-btn {
    padding: 9px 16px;
    border: none; 
    border-radius: 20px;
    background: rgba(255, 255, 255, 0.92);
    color: #5A1A30;
    font-family: 'Nunito', sans-serif;
    font-weight: 700; 
    font-size: 0.85rem;
    cursor: pointer;
    box-shadow: 0 4px 18px rgba(90, 26, 48, 0.10);
    transition: background 0.15s, transform 0.1s;
  }
  .custom-bar-btn:hover { 
    background: #ffffff; 
    transform: translateY(-1px); 
  }
  .custom-bar-btn:active { 
    transform: scale(0.96); 
  }
</style>

<div 
  bind:this={wrapperRef} 
  style="position: fixed; inset: 0; display: flex; justify-content: center; align-items: center; user-select: none; background-color: #ff8fae; font-family: 'Nunito', sans-serif;"
>
  <div class="custom-top-bar">
    <button 
      class="custom-bar-btn"
      onclick={() => window.location.href = '/land'} 
    >
      ←
    </button>

    <button 
      class="custom-bar-btn"
      onclick={toggleHistory}
    >
      {historyOpen ? 'Close history' : '💬 History'}
    </button>
  </div>
  
  <button 
    style="position: absolute; top: 25px; right: 25px; background: none; border: none; padding: 0; cursor: pointer; z-index: 10; width: 150px; height: 150px;" 
    onclick={handleDiveToggle} 
    aria-label="Toggle Sea Depth"
  >
    <canvas bind:this={uiCanvasRef} width="150" height="150" style="display: block; width: 150px; height: 150px;"></canvas>
  </button>

  <canvas 
    bind:this={canvasRef} 
    style="width: 100%; height: 100%; cursor: grab; touch-action: none; z-index: 1; {embedded ? 'position: absolute; inset: 0;' : ''}"
    onpointerdown={onPointerDown}
    onpointermove={onPointerMove}
    onpointerup={onPointerUp}
    onclick={onCanvasClick}
  ></canvas>
  
  <canvas bind:this={overlayCanvasRef} style="position: absolute; inset: 0; width: 100%; height: 100%; pointer-events: none; z-index: 2;"></canvas>

  {#if !embedded && radialMenuOpen}
    <div style="position: absolute; z-index: 30; left: {clickCoords.x}px; top: {clickCoords.y}px;">
      {#each interactionCards as card}
        <button 
          style="position: absolute; transform: rotate({card.angle}deg) translate(140px) rotate(-{card.angle}deg);"
          onclick={() => openCard(card.id)}
        >
          <span>{card.icon}</span>
          <span>{card.label}</span>
        </button>
      {/each}
    </div>
  {/if}

  {#if activeCard}
    <div 
      style="position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 40; display: flex; align-items: center; justify-content: center;" 
      onclick={closeCard}
    >
      <div 
        style="background: white; padding: 20px; max-width: 400px; border-radius: 8px;" 
        onclick={(e) => e.stopPropagation()}
      >
        <h3>{activeCard.title}</h3>
        <p>{activeCard.description}</p>
        <p>💡 {activeCard.hint}</p>
        <div>
          <button onclick={closeCard}>Back</button>
          <button onclick={executeCard}>Execute</button>
        </div>
      </div>
    </div>
  {/if}

  {#if thoughtBubbleVisible}
    <div style="position: absolute; z-index: 25; background: white; border-radius: 12px; padding: 15px; width: 260px; left: {fishScreenCoords.x + 130}px; top: {fishScreenCoords.y - 45}px;">
      <div>
        <p style="margin: 0 0 10px 0; color: #333;">{thoughtBubbleText}</p>

        {#if !isReplying && !isApiLoading}
          <div>
            <button onclick={openReplyInput}>Reply</button>
          </div>
        {:else}
          <div style="display: flex; gap: 5px;">
            <input
              type="text"
              placeholder="Send a ripple back..."
              bind:value={replyInputValue}
              onkeydown={handleReplyKey}
              disabled={isApiLoading}
              style="flex: 1;"
              autofocus
            />
            <button onclick={sendReply} disabled={isApiLoading}>→</button>
          </div>
        {/if}
      </div>
    </div>
  {/if}

  <p style="position: absolute; bottom: 25px; color: white; opacity: 0.8; font-size: 0.9rem; pointer-events: none; z-index: 5;">{hintText}</p>
</div>