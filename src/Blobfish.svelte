<script>
  //@ts-nocheck
  import { onMount, createEventDispatcher } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';

  const dispatch = createEventDispatcher();

  let canvasRef;
  let uiCanvasRef;
  let overlayCanvasRef; 
  const OPENAI_API_KEY = import.meta.env.VITE_OPENAI_API_KEY;
  
  export let embedded = false;
  
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
  let historyOpen = false;
  
  let thoughtBubbleVisible = false;
  let isReplying = false; 
  let thoughtBubbleText = '';
  let replyInputValue = '';
  let thoughtTimer = null;
  let bubbleAutoHideTimer = null;
  let isApiLoading = false;
  let conversationActive = false;
  let replyInputRef;

  // Gesture Tracking for Double Tap
  let lastClickTime = 0;
  let clickTimeout;

  const interactionCards = [
    {
      id: 'float',
      icon: '🔄',
      label: 'Float',
      description: 'Orbit Swimming'
    },
    {
      id: 'jump',
      icon: '🐬',
      label: 'Jump',
      description: 'Breaching Leap'
    },
    {
      id: 'chat',
      icon: '💬',
      label: 'Chat',
      description: 'Consult the Blobfish'
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
    blobSkin: '#F4B8C2', blobNose: '#F3AEBB', blobFin: '#C97A8D', blobMouth: '#6B3A45', blobTailBody: '#D88A9D',
    blobDeepSea: '#6C8EA4', blobDeepSeaDark: '#4A6578', blobMouthDeep: '#334856', blobEyeDeep: '#94b3c7', blobPupilDeep: '#1d2a33',
    waterTransition: '#1C2541', seaweedDark: '26, 54, 39', seaweedLight: '53, 94, 59',
    bgSky: '#ff8fae', bgPinkWater: '#FAD0C4', bgMidWater: '#4A90E2', bgTransBlue1: '#3B78C4', bgTransBlue2: '#2B5FA6',
    bgDeepWater: '#1C2541', bgAbyss: '#0B132B', landGreen: '#6CA35E', landGreenDark: '#4E8241', cloudBody: 'rgba(255, 255, 255, 0.75)'
  };

  function toggleHistory() {
    historyOpen = !historyOpen;
  }

  function showThought(text, keepAlive = false) {
    if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
    thoughtBubbleText = text;
    thoughtBubbleVisible = true;
    conversationActive = keepAlive;

    if (!keepAlive && !isReplying) {
      bubbleAutoHideTimer = setTimeout(() => {
        thoughtBubbleVisible = false;
      }, 12000);
    }
  }

  function openReplyInput() {
    if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
    isReplying = true;
    thoughtBubbleVisible = true;
    setTimeout(() => replyInputRef?.focus(), 50);
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
        messages: [{ role: 'system', content: 'You are a wise blobfish' }, ...chatHistory],
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
    tl.to(fishZone.translate, { duration: 1.75, x: -160, z: -100, ease: 'sine.inOut', yoyo: true, repeat: 1 }, 0);
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
    isDragging = true; wasDragging = false;
    lastX = e.clientX; lastY = e.clientY;
    try { canvasRef.setPointerCapture(e.pointerId); } catch (_) {}
  }

  function onPointerMove(e) {
    if (!isDragging) return;
    const dx = e.clientX - lastX, dy = e.clientY - lastY;
    lastX = e.clientX; lastY = e.clientY;
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
      
      if (currentTime - lastClickTime < 300) {
        clearTimeout(clickTimeout);
        triggerJumpAnimation();
      } else {
        clearTimeout(clickTimeout);
        clickTimeout = setTimeout(() => {
          if (!isApiLoading && !isReplying) {
             showThought("Did you poke me? I'm quite squishy... Use the buttons below to interact.", false);
          }
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
    }
    resizeOverlay();
    window.addEventListener('resize', resizeOverlay);

    scene = new Zdog.Illustration({
      element: canvasRef, dragRotate: false, resize: 'window', rotate: { x: 0, y: -0.12 }, 
    });

    backgroundZone = new Zdog.Anchor({ addTo: scene, translate: { z: -450 } });

    skyBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [{ x: -1200, y: -900 }, { x: 1200, y: -900 }, { x: 1200, y: -260 }, { arc: [ { x: 0, y: -200 }, { x: -1200, y: -260 } ] }],
      stroke: 40, color: color.bgSky, fill: true
    });

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

    new Zdog.Shape({
      addTo: sceneryZone,
      path: [{ x: -800, y: 20 }, { arc: [ { x: -500, y: -200 }, { x: -200, y: 20 } ] }, { x: -200, y: 20 }, { x: -800, y: 20 }],
      stroke: 20, color: color.landGreenDark, fill: true
    });

    new Zdog.Shape({
      addTo: sceneryZone,
      path: [{ x: 200, y: 20 }, { arc: [ { x: 500, y: -240 }, { x: 800, y: 20 } ] }, { x: 800, y: 20 }, { x: 200, y: 20 }],
      stroke: 20, color: color.landGreenDark, fill: true
    });

    midBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [{ x: -1200, y: -280 }, { arc: [ { x: 0, y: -220 }, { x: 1200, y: -280 } ] }, { x: 1200, y: 50 }, { arc: [ { x: 0, y: 110 }, { x: -1200, y: 50 } ] }],
      stroke: 50, color: color.bgPinkWater, fill: true
    });

    midTransitionBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [{ x: -1200, y: 30 }, { arc: [ { x: 0, y: 90 }, { x: 1200, y: 30 } ] }, { x: 1200, y: 280 }, { arc: [ { x: 0, y: 340 }, { x: -1200, y: 280 } ] }],
      stroke: 50, color: color.bgMidWater, fill: true
    });

    deepTransitionBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [{ x: -1200, y: 260 }, { arc: [ { x: 0, y: 320 }, { x: 1200, y: 260 } ] }, { x: 1200, y: 540 }, { arc: [ { x: 0, y: 600 }, { x: -1200, y: 540 } ] }],
      stroke: 50, color: color.bgTransBlue1, fill: true
    });

    abyssBlock = new Zdog.Shape({
      addTo: backgroundZone,
      path: [{ x: -1200, y: 520 }, { arc: [ { x: 0, y: 580 }, { x: 1200, y: 520 } ] }, { x: 2500, y: 2500 }, { x: -2500, y: 2500 }],
      stroke: 50, color: color.bgTransBlue2, fill: true
    });

    fishZone = new Zdog.Anchor({ addTo: scene });

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
      addTo: blobMasterAnchor, stroke: 14, color: color.blobMouth, closed: false,
      path: [{ x: -28, y: 40, z: 44 }, { bezier: [{ x: -12, y: 28, z: 54 }, { x: 12, y: 28, z: 54 }, { x: 28, y: 40, z: 44 }] }]
    });

    const finShapePath = [{ x: 0, y: 0 }, { x: -30, y: -10 }, { x: -50, y: 15 }, { x: -40, y: 35 }, { x: -10, y: 25 }];

    leftPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    new Zdog.Shape({ addTo: new Zdog.Anchor({ addTo: leftPecContainer, translate: { x: -52, y: 22, z: -8 } }), path: finShapePath, stroke: 24, color: color.blobFin, fill: true });

    rightPecContainer = new Zdog.Anchor({ addTo: blobMasterAnchor });
    new Zdog.Shape({ addTo: new Zdog.Anchor({ addTo: rightPecContainer, translate: { x: 52, y: 22, z: -8 } }), path: finShapePath, scale: { x: -1 }, stroke: 24, color: color.blobFin, fill: true });

    const blobTail = new Zdog.Anchor({ addTo: blobMasterAnchor, translate: { y: 10, z: -68 } });
    new Zdog.Shape({ addTo: blobTail, stroke: 52, color: color.blobTailBody });
    blobTailFin = new Zdog.Anchor({ addTo: blobTail, translate: { z: -14 } });
    new Zdog.Shape({ addTo: blobTailFin, stroke: 32, color: color.blobFin, closed: false, path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -4 }, { x: -12, y: -12, z: -8 }, { x: 0, y: -18, z: -14 }] }] });

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
      addTo: handsomeMasterAnchor, stroke: 11, color: color.blobMouthDeep, closed: false,
      path: [{ x: -18, y: 10, z: 25 }, { x: 0, y: 14, z: 27 }, { x: 18, y: 10, z: 25 }]
    });

    leftWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    new Zdog.Shape({ addTo: new Zdog.Anchor({ addTo: leftWingContainer, translate: { x: -48, y: 22, z: -8 } }), path: finShapePath, scale: { x: 0.666, y: 0.666, z: 0.666 }, stroke: 16, color: color.blobDeepSeaDark, fill: true });

    rightWingContainer = new Zdog.Anchor({ addTo: handsomeMasterAnchor });
    new Zdog.Shape({ addTo: new Zdog.Anchor({ addTo: rightWingContainer, translate: { x: 48, y: 22, z: -8 } }), path: finShapePath, scale: { x: -0.666, y: 0.666, z: 0.666 }, stroke: 16, color: color.blobDeepSeaDark, fill: true });

    const handsomeTail = new Zdog.Anchor({ addTo: handsomeMasterAnchor, translate: { y: -4, z: -72 } });
    new Zdog.Shape({ addTo: handsomeTail, stroke: 60, color: color.blobDeepSea });
    handsomeTailFin = new Zdog.Anchor({ addTo: handsomeTail, translate: { z: -14 } });
    new Zdog.Shape({ addTo: handsomeTailFin, stroke: 80, color: color.blobDeepSeaDark, closed: false, path: [{ y: 0, z: 0 }, { bezier: [{ x: -6, y: -8, z: -5 }, { x: -14, y: -14, z: -9 }, { x: 0, y: -22, z: -14 }] }] });

    pinkFadeShapes = setupFadeGroup(blobMasterAnchor);
    darkFadeShapes = setupFadeGroup(handsomeMasterAnchor);
    sceneryFadeShapes = setupFadeGroup(sceneryZone);

    uiScene = new Zdog.Illustration({ element: uiCanvasRef, dragRotate: false, resize: false });

    const UIAnchor = new Zdog.Anchor({ addTo: uiScene, translate: { x: 0, y: 0 } });
    new Zdog.Cylinder({ addTo: UIAnchor, diameter: 82, length: 16, stroke: false, color: '#7f8c8d', backface: '#34495e' });
    new Zdog.Cylinder({ addTo: UIAnchor, diameter: 70, length: 2, stroke: false, color: '#FFFFFF', translate: { z: 8 } });

    for (let i = 0; i < 6; i++) {
      let angle = (i / 5) * Math.PI - Math.PI;
      new Zdog.Shape({ addTo: UIAnchor, path: [{ y: -26 }, { y: -32 }], stroke: 3, color: '#e74c3c', rotate: { z: angle }, translate: { z: 9.5 } });
    }

    gaugeNeedle = new Zdog.Anchor({ addTo: UIAnchor, translate: { z: 11 }, rotate: { z: -Math.PI * 0.75 } });
    new Zdog.Shape({ addTo: gaugeNeedle, path: [{ y: 6 }, { y: -30 }], stroke: 4, color: '#2c3e50' });
    new Zdog.Shape({ addTo: gaugeNeedle, stroke: 12, color: '#e74c3c' });

    uiScene.updateRenderGraph();
    scheduleThought();

    let frame = 0; let running = true;

    function drawSeaweed() {
      if (fishState.darkAlpha < 0.01) return; 
      const w = overlayCanvasRef.width; const h = overlayCanvasRef.height;

      seaweedStrands.forEach((strand, idx) => {
        const baseX = w * strand.xPct;
        const currentHeight = strand.height * fishState.darkAlpha;
        const greenTone = idx % 2 === 0 ? color.seaweedDark : color.seaweedLight;

        ctx.save(); ctx.beginPath();
        ctx.fillStyle = `rgba(${greenTone}, ${0.65 * fishState.darkAlpha})`;

        const segments = 7;
        let leftSidePoints = [], rightSidePoints = [];

        for (let i = 0; i <= segments; i++) {
          const segmentPct = i / segments;
          const yPos = h + 30 - (currentHeight * segmentPct);
          const waveFactor = Math.sin((frame * strand.speed) + (segmentPct * Math.PI * 1.2) + strand.delay);
          const xOffset = waveFactor * (25 * segmentPct);
          const currentWidth = strand.baseWidth * (1 - segmentPct);

          leftSidePoints.push({ x: baseX + xOffset - currentWidth / 2, y: yPos });
          rightSidePoints.unshift({ x: baseX + xOffset + currentWidth / 2, y: yPos });
        }

        ctx.moveTo(leftSidePoints[0].x, leftSidePoints[0].y);
        leftSidePoints.forEach(pt => ctx.lineTo(pt.x, pt.y));
        rightSidePoints.forEach(pt => ctx.lineTo(pt.x, pt.y));
        ctx.closePath(); ctx.fill(); ctx.restore();
      });
    }

    function drawWaterTransition() {
      ctx.clearRect(0, 0, overlayCanvasRef.width, overlayCanvasRef.height);
      drawSeaweed();

      let progress = transitionProgress.value;
      if (progress <= 0 || progress >= 1) return;

      const w = overlayCanvasRef.width; const h = overlayCanvasRef.height;
      ctx.fillStyle = color.waterTransition;

      const numBars = 3; const barThickness = Math.max(w, h) * 0.4;
      
      for (let i = 0; i < numBars; i++) {
        let barProgress = gsap.utils.clamp(0, 1, gsap.utils.mapRange(i * 0.1, 1 - (numBars - 1 - i) * 0.1, 0, 1, progress));
        ctx.save();
        ctx.translate(-barThickness + (w + barThickness * 2) * barProgress, h / 2);
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

      backgroundZone.rotate.y = -scene.rotate.y; backgroundZone.rotate.x = -scene.rotate.x;
      sceneryZone.rotate.y = -scene.rotate.y; sceneryZone.rotate.x = -scene.rotate.x;

      pinkFadeShapes.forEach(({ node, r, g, b }) => node.color = `rgba(${r},${g},${b},${fishState.pinkAlpha})`);
      darkFadeShapes.forEach(({ node, r, g, b }) => node.color = `rgba(${r},${g},${b},${fishState.darkAlpha})`);
      sceneryFadeShapes.forEach(({ node, r, g, b }) => node.color = `rgba(${r},${g},${b},${fishState.pinkAlpha})`);
      
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
      if (blink > 262) s_blink = 0.5 + 0.5 * Math.cos(((blink - 262) / 18) * Math.PI * 2);
      
      eyeL_pink.scale.y = eyeR_pink.scale.y = eyeL_blue.scale.y = eyeR_blue.scale.y = s_blink;

      scene.updateRenderGraph(); uiScene.updateRenderGraph(); drawWaterTransition();
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
      duration: 2.5, y: isDeep ? -850 : 0, ease: 'power2.inOut'
    }, 0);

    tl.to({}, {
      duration: 1.25,
      onComplete: () => {
        if (isDeep) {
          skyBlock.color = color.bgTransBlue1; midBlock.color = color.bgTransBlue2;
          midTransitionBlock.color = color.bgDeepWater; deepTransitionBlock.color = color.bgAbyss; abyssBlock.color = '#040714'; 
        } else {
          skyBlock.color = color.bgSky; midBlock.color = color.bgPinkWater;
          midTransitionBlock.color = color.bgMidWater; deepTransitionBlock.color = color.bgTransBlue1; abyssBlock.color = color.bgTransBlue2;
        }
      }
    }, 0.6);

    tl.to(gaugeNeedle.rotate, { duration: 2.5, z: isDeep ? Math.PI * 0.75 : -Math.PI * 0.75, ease: 'back.out(1.2)' }, 0);

    transitionProgress.value = 0;
    tl.to(transitionProgress, { duration: 1.4, value: 1, ease: 'power1.inOut' }, 0.55);

    tl.call(() => {
      fishState.pinkAlpha = isDeep ? 0 : 1;
      fishState.darkAlpha = isDeep ? 1 : 0;
    }, null, 1.25); 
  }
</script>

{#if !embedded}
  <div class="top-bar">
    <button class="bar-btn" on:click={() => dispatch('back')}>←</button>
    <button class="bar-btn" on:click={toggleHistory}>
      {historyOpen ? 'Close history' : '💬 History'}
    </button>
  </div>
{/if}

{#if historyOpen}
  <div class="history-panel">
    <div class="history-header">
      <h2>Session chat</h2>
      <button class="close-btn" on:click={() => historyOpen = false}>✕</button>
    </div>
    <div class="history-scroll">
      {#each chatHistory as msg}
        <div class="history-msg {msg.role}">
          <span class="history-label">{msg.role === 'assistant' ? '🐟 Blobfish' : 'You'}</span>
          <p>{msg.content}</p>
        </div>
      {/each}
    </div>
  </div>
{/if}

<canvas bind:this={overlayCanvasRef} class="overlay-scene"></canvas>
<canvas bind:this={canvasRef} class="scene" class:embedded on:pointerdown={onPointerDown} on:pointermove={onPointerMove} on:pointerup={onPointerUp} on:pointercancel={onPointerUp} on:click={onCanvasClick}></canvas>
<canvas bind:this={uiCanvasRef} class="ui-scene" on:click={handleDiveToggle}></canvas>

<div class="center-actions">

</div>

<div class="thought-wrap" class:visible={thoughtBubbleVisible || isReplying}>
  <div class="thought-bubble">
    <div class="thought-dots" style="display: {isApiLoading ? 'flex' : 'none'}">
      <span></span><span></span><span></span>
    </div>
    <p class="thought-text" style="display: {isApiLoading ? 'none' : 'block'}">{thoughtBubbleText}</p>
  </div>
  
  <div class="thought-reply">
    <textarea
      bind:this={replyInputRef}
      bind:value={replyInputValue}
      placeholder="Reply to the Blobfish…"
      rows="1"
      disabled={isApiLoading}
      on:keydown={handleReplyKey}
    ></textarea>
    <button class="reply-send" on:click={sendReply} disabled={isApiLoading || !replyInputValue.trim()}>
      {isApiLoading ? '…' : '➤'}
    </button>
  </div>
</div>

<p class="hint">{hintText}</p>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap");

  :global(body) {
    margin: 0; padding: 0;
    width: 100vw; height: 100vh;
    background: #0B132B;
    overflow: hidden;
    font-family: 'Nunito', sans-serif;
  }

  canvas.scene, canvas.overlay-scene {
    position: fixed; inset: 0;
    width: 100vw; height: 100vh;
    display: block; touch-action: none;
  }
  
  canvas.scene { z-index: 1; }
  canvas.overlay-scene { z-index: 2; pointer-events: none; }
  
  canvas.scene.embedded { position: absolute; width: 100%; height: 100%; inset: 0; }

  canvas.ui-scene {
    position: fixed;
    top: 70px; right: 16px;
    width: 120px; height: 120px;
    z-index: 10;
    cursor: pointer;
    border-radius: 50%;
  }

  /* ── Top bar & History panel ────────────────────────────────────────── */
  .top-bar {
    position: fixed;
    top: 16px; left: 16px;
    display: flex; gap: 10px;
    z-index: 30;
  }
  .bar-btn {
    padding: 9px 16px;
    border: none; border-radius: 20px;
    background: rgba(255,255,255,0.92);
    color: #2E3A47;
    font-weight: 700; font-size: 0.85rem;
    cursor: pointer;
    box-shadow: 0 4px 18px rgba(0,0,0,.15);
    transition: background .15s, transform .1s;
  }
  .bar-btn:hover { background: #fff; transform: translateY(-1px); }
  .bar-btn:active { transform: scale(.96); }

  .history-panel {
    position: fixed;
    top: 62px; left: 16px;
    width: min(360px, calc(100vw - 32px));
    max-height: calc(100vh - 80px);
    background: rgba(255,255,255,0.97);
    border-radius: 24px;
    box-shadow: 0 20px 60px rgba(0,0,0,.25);
    z-index: 25; display: flex; flex-direction: column; overflow: hidden;
  }
  .history-header {
    display: flex; justify-content: space-between; align-items: center;
    padding: 18px 20px 12px;
    border-bottom: 1px solid rgba(74,144,226,.18);
  }
  .history-header h2 { margin: 0; font-size: 1rem; color: #2E3A47; }
  .close-btn {
    border: none; background: #e4eef8; color: #3A6699; border-radius: 50%;
    width: 30px; height: 30px; cursor: pointer; font-size: 0.85rem; font-weight: 700;
  }
  .history-scroll {
    overflow-y: auto; padding: 14px 16px;
    display: flex; flex-direction: column; gap: 10px;
  }
  .history-msg { border-radius: 16px; padding: 12px 14px; font-size: 0.9rem; line-height: 1.55; }
  .history-msg p { margin: 4px 0 0; color: #26313b; }
  .history-msg.assistant { background: #eef4fb; }
  .history-msg.user { background: #f7f4ef; }
  .history-label { font-weight: 800; font-size: 0.78rem; color: #4A7BB0; }

  /* ── Center Bottom Menu ────────────────────────────────────────────── */
  .center-actions {
    position: fixed;
    bottom: 32px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 12px;
    z-index: 20;
  }
  .action-btn {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 12px 20px;
    background: rgba(255,255,255,0.95);
    border: none;
    border-radius: 30px;
    font-weight: 800;
    font-size: 0.95rem;
    color: #2E3A47;
    cursor: pointer;
    box-shadow: 0 8px 24px rgba(0,0,0,0.25);
    transition: transform 0.15s, background 0.15s;
    font-family: 'Nunito', sans-serif;
  }
  .action-btn:hover { background: #fff; transform: translateY(-3px); }
  .action-btn:active { transform: translateY(1px); }
  .btn-icon { font-size: 1.2rem; }

  /* ── Thought bubble (Locked to Bottom Right) ───────────────────────── */
  .thought-wrap {
    position: fixed;
    bottom: 90px;
    right: 32px;
    max-width: min(360px, 90vw);
    display: flex; flex-direction: column; gap: 10px;
    z-index: 15; pointer-events: none; opacity: 0;
    transform: translateY(8px) scale(.96);
    transition: opacity .35s ease, transform .35s ease;
    align-items: flex-end;
  }
  .thought-wrap.visible { opacity: 1; transform: translateY(0) scale(1); pointer-events: all; }
  
  .thought-bubble {
    background: #fff;
    border: 2px solid rgba(74,144,226,.40);
    border-radius: 22px 22px 6px 22px; 
    padding: 14px 18px;
    box-shadow: 0 8px 36px rgba(0,0,0,.25);
    position: relative; max-width: 100%;
  }
  .thought-dots { display: flex; gap: 4px; margin-bottom: 6px; }
  .thought-dots span { width: 5px; height: 5px; background: #4A90E2; border-radius: 50%; }
  .thought-text { margin: 0; font-size: 0.92rem; color: #26313b; line-height: 1.6; font-weight: 600; }
  
  .thought-reply { display: flex; gap: 8px; align-items: flex-end; width: 100%; }
  .thought-reply textarea {
    flex: 1; background: rgba(255,255,255,0.96);
    border: 1.5px solid rgba(74,144,226,.50);
    border-radius: 14px; padding: 10px 14px;
    font-family: 'Nunito', sans-serif; font-size: 0.9rem; color: #26313b;
    outline: none; resize: none;
    height: 44px; min-height: 44px; max-height: 88px;
    line-height: 1.5; box-sizing: border-box;
    transition: border-color .15s, box-shadow .15s;
  }
  .thought-reply textarea::placeholder { color: #81a4c9; }
  .thought-reply textarea:focus { border-color: #3B78C4; box-shadow: 0 0 0 3px rgba(59,120,196,.20); }
  
  .reply-send {
    width: 44px; height: 44px;
    border-radius: 50%; border: none;
    background: #4A90E2; color: #fff;
    font-size: 1rem; cursor: pointer; flex-shrink: 0;
    transition: background .12s, transform .1s;
    display: flex; align-items: center; justify-content: center;
  }
  .reply-send:hover:not(:disabled) { background: #3B78C4; }
  .reply-send:active:not(:disabled) { transform: scale(.94); }
  .reply-send:disabled { opacity: .5; cursor: default; }

  /* ── Hints ───────────────────────────────────────────────────────── */
  .hint {
    position: fixed;
    top: 22px; left: 50%;
    transform: translateX(-50%);
    font-size: 0.85rem; font-weight: 700;
    color: rgba(255, 255, 255, 0.8);
    pointer-events: none;
    letter-spacing: .04em;
    z-index: 10;
    text-shadow: 0 2px 4px rgba(0,0,0,0.4);
  }
</style>