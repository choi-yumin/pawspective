<script>
  //@ts-nocheck
  import { onMount, createEventDispatcher } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';
  import SlothBackground from './SlothBackground.svelte';

  const dispatch = createEventDispatcher();
  let canvasRef;

  export let embedded = false;

  const OPENAI_API_KEY = import.meta.env.VITE_OPENAI_API_KEY;

  let radialMenuOpen = false;
  let activeCard = null;
  let historyOpen = false;
  let thoughtBubbleVisible = false;
  let thoughtBubbleText = '';
  let replyInputValue = '';
  let thoughtTimer = null;
  let bubbleAutoHideTimer = null;
  let isApiLoading = false;
  let conversationActive = false;

  const interactionCards = [
    {
      id: 'wake',
      icon: '👀',
      label: 'Wake',
      title: 'Gently wake Mossy',
      description: 'Give the sloth a slow little nudge to blink awake.',
      hint: 'Or just drag the sloth to stir it.',
      angle: -90
    },
    {
      id: 'climb',
      icon: '🌿',
      label: 'Climb',
      title: 'Watch it climb',
      description: 'Mossy shuffles, very slowly, along the branch.',
      hint: 'Patience — sloths take their time.',
      angle: -30
    },
    {
      id: 'yawn',
      icon: '🥱',
      label: 'Yawn',
      title: 'Make it yawn',
      description: 'A long, slow stretch and the widest of yawns.',
      hint: 'Contagious. You might yawn too.',
      angle: 30
    },
    {
      id: 'snack',
      icon: '🌺',
      label: 'Snack',
      title: 'Offer a hibiscus',
      description: 'A hibiscus is a sloth\'s rare favorite treat.',
      hint: 'The flower hangs to the right of the branch.',
      angle: 90
    },
    {
      id: 'chat',
      icon: '💬',
      label: 'Chat',
      title: 'Talk with Mossy',
      description: 'The sloth has slow, gentle wisdom about rest and patience.',
      hint: 'Reply to Mossy\'s thought bubbles below.',
      angle: 150
    }
  ];

  let chatHistory = [
    { role: 'assistant', content: "Mmm… hello. 🌿 Ask me about rest, patience, or slowing down." }
  ];

  const SYSTEM_TEXT = `You are Mossy, a serene, sleepy three-toed sloth who lives high in the rainforest canopy. You speak slowly and warmly in short, unhurried messages (2-4 sentences max). You use canopy, leaf, and slowness metaphors naturally.
Your specialty is helping people think about:
- Rest and the quiet wisdom of slowing down
- Patience, and letting things unfold in their own time
- Being gentle with yourself when you feel behind
- Savoring small, ordinary moments
You give calm, insightful, slightly dreamy advice. You never rush and you never lecture. Keep it warm, short, and wise.`;

  const thoughtPrompts = [
    'The canopy is quiet today. What is on your mind?',
    'I moved one branch over since sunrise. That feels like plenty.',
    'Do you ever rush right past the good parts?',
    'Resting is not the same as falling behind, you know.',
    'I am watching a single leaf drift. Want to watch with me?',
    'What would happen if you went a little slower today?'
  ];

  let triggerWakeFn;
  let climbFn;
  let yawnFn;
  let snackFn;
  let talkingFn;

  function goHome() { dispatch('back'); }

  function toggleRadialMenu() {
    radialMenuOpen = !radialMenuOpen;
    if (radialMenuOpen) { activeCard = null; historyOpen = false; }
  }

  function openCard(id) {
    activeCard = interactionCards.find(c => c.id === id);
    radialMenuOpen = false;
  }

  function closeCard() { activeCard = null; }

  function executeCard() {
    if (!activeCard) return;
    if (activeCard.id === 'wake')  triggerWakeFn?.();
    if (activeCard.id === 'climb') climbFn?.();
    if (activeCard.id === 'yawn')  yawnFn?.();
    if (activeCard.id === 'snack') snackFn?.();
    if (activeCard.id === 'chat') {
      activeCard = null;
      return;
    }
    activeCard = null;
  }

  function toggleHistory() {
    historyOpen = !historyOpen;
    if (historyOpen) { radialMenuOpen = false; activeCard = null; }
  }

  function showThought(text, keepAlive = false) {
    if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
    thoughtBubbleText = text;
    thoughtBubbleVisible = true;
    conversationActive = keepAlive;
    if (!keepAlive) {
      bubbleAutoHideTimer = setTimeout(() => {
        thoughtBubbleVisible = false;
      }, 13000);
    }
  }

  function scheduleThought() {
    if (thoughtTimer) clearTimeout(thoughtTimer);
    // Slower cadence than the bee — Mossy is in no hurry.
    thoughtTimer = setTimeout(() => {
      const prompt = thoughtPrompts[Math.floor(Math.random() * thoughtPrompts.length)];
      showThought(prompt);
      scheduleThought();
    }, 22000 + Math.random() * 14000);
  }

  async function sendReply() {
    const msg = replyInputValue.trim();
    if (!msg || isApiLoading) return;
    replyInputValue = '';
    thoughtBubbleVisible = false;
    await askMossy(msg);
  }

  function handleReplyKey(e) {
    if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); sendReply(); }
  }

  async function askMossy(userMessage) {
    if (isApiLoading) return;
    isApiLoading = true;
    showThought('…', true);
    talkingFn?.();

    chatHistory = [...chatHistory, { role: 'user', content: userMessage }];

    try {
      const endpoint = 'https://api.openai.com/v1/chat/completions';
      const body = {
        model: 'gpt-4o-mini',
        messages: [
          { role: 'system', content: SYSTEM_TEXT },
          ...chatHistory
        ],
        max_tokens: 256,
        temperature: 0.9
      };
      const res = await fetch(endpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${OPENAI_API_KEY}`
        },
        body: JSON.stringify(body)
      });

      if (!res.ok) {
        const errorData = await res.json();
        console.error('OpenAI Internal Error JSON:', errorData);
        throw new Error(`API returned status ${res.status}`);
      }

      const data = await res.json();
      const replyText = data?.choices?.[0]?.message?.content;

      if (replyText) {
        const cleanReply = replyText.trim();
        chatHistory = [...chatHistory, { role: 'assistant', content: cleanReply }];
        showThought(cleanReply, true);
      } else {
        showThought("Mmm… my mind drifted off. Ask me again?", true);
      }
    } catch (e) {
      console.error('OpenAI Catch block triggered:', e);
      chatHistory = chatHistory.slice(0, -1);
      showThought("Mmm… my mind drifted off. Ask me again?", true);
    } finally {
      isApiLoading = false;
    }
  }

  onMount(() => {
    if (!canvasRef) return;
    const TAU = Zdog.TAU;

    const color = {
      fur:        '#8B6F4E',
      furDark:    '#6B5238',
      furLight:   '#B89B72',
      faceCream:  '#D8C29A',
      mask:       '#5A4632',
      nose:       '#3D2A1A',
      eye:        '#2A1C12',
      cheek:      '#E59A86',
      claw:       '#4A3826',
      branch:     '#6B4A2A'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      // gentle side-on view; the render loop adds a slow turntable on top
      rotate: { x: -0.10, y: -0.08, z: -0.15 }
    });

    // ─── The Sloth — slung horizontally beneath the branch, in profile ──
    const SLOTH_CONT = new Zdog.Anchor({ addTo: scene, translate: { x: -10, y: 10, z: -300 }, scale: 1 });
    const PIVOT = new Zdog.Anchor({ addTo: SLOTH_CONT });
    const SLOTH = new Zdog.Anchor({ addTo: PIVOT });

    // Body — fatter and chunkier
    new Zdog.Shape({
      addTo: SLOTH,
      path: [
        { x: -75, y: -4, z: 0 },
        { bezier: [
          { x: -30, y: 25, z: 0 }, // Pulled down slightly to maintain the sag
          { x: 30, y: 25, z: 0 },
          { x: 75, y: -4, z: 0 }
        ]}
      ],
      stroke: 120, // Increased stroke to make the body fatter
      color: color.fur, closed: false
    });

    // Belly — shifted to the top (facing the branch) instead of the side
    new Zdog.Shape({
      addTo: SLOTH,
      path: [
        { x: -40, y: -4, z: 0 },
        { bezier: [
          { x: -15, y: 8, z: 0 },
          { x: 15, y: 8, z: 0 },
          { x: 40, y: -4, z: 0 }
        ]}
      ],
      stroke: 40, 
      color: color.furLight, 
      translate: { y: -28, z: 0 }, // z: 0 perfectly centers it; y: -28 pushes it to the top surface
      closed: false
    });

    // ─── Four limbs reaching UP to grip the branch ──────────────────────
    // near pair toward camera (+z), far pair behind (-z)
    function makeLimb(baseX, baseY, baseZ, gripX, stroke) {
      const limb = new Zdog.Anchor({ addTo: SLOTH, translate: { x: baseX, y: baseY, z: baseZ } });
      const dx = gripX - baseX;
      // Adjusted reach height so limbs connect with the new background branch
      const dy = -210 - baseY; 

      new Zdog.Shape({
        addTo: limb,
        path: [
          { x: 0, y: 0, z: 0 },
          { bezier: [
            { x: dx * 0.2, y: dy * 0.4, z: 0 },
            { x: dx * 0.7, y: dy * 0.8, z: 0 },
            { x: dx,       y: dy,       z: 0 }
          ]}
        ],
        stroke, color: color.fur, fill: false
      });

      // three little claw hooks curling over the branch
      const hand = new Zdog.Anchor({ addTo: limb, translate: { x: dx, y: dy, z: 0 } });
      for (let i = 0; i < 3; i++) {
        new Zdog.Shape({
          addTo: hand,
          path: [
            { x: (i - 1) * 6, y: 0 },
            { bezier: [
              { x: (i - 1) * 6,     y: -6 },
              { x: (i - 1) * 6 - 4, y: -12 },
              { x: (i - 1) * 6 - 5, y: -16 }
            ]}
          ],
          stroke: 5, color: color.claw, closed: false
        });
      }
      return limb;
    }

    // Front arm (near) and back arm (far) — spread further left
    const leftArm  = makeLimb(-60, -10,  46, -110, 34);  
    const rightArm = makeLimb(-80, -10, -46, -170, 34); 

    // Front leg (near) and back leg (far) — spread further right
    const leftLeg  = makeLimb(60, -12,  46,  80, 34);  
    const rightLeg = makeLimb(100, -12, -46,  140, 34);

    // ─── Head — sphere with a snout poking left ─────────────────────────
    const head = new Zdog.Anchor({ addTo: SLOTH, translate: { x: -140, y: 12, z: 14 }, scale: { y: -1 } });
    new Zdog.Shape({ addTo: head, stroke: 92, color: color.fur });                                       // skull
    new Zdog.Shape({ addTo: head, stroke: 46, color: color.faceCream, translate: { x: -30, y: 10, z: 14 } }); // muzzle
    new Zdog.Shape({ addTo: head, stroke: 16, color: color.nose,      translate: { x: -54, y:  8, z: 14 } }); // nose

    // dark mask patch + single eye (profile)
    new Zdog.Ellipse({ addTo: head, width: 24, height: 40, stroke: 6, color: color.mask, fill: true, translate: { x: -14, y: -2, z: 38 } });
    new Zdog.Shape({ addTo: head, stroke: 15, color: color.eye, translate: { x: -16, y: -2, z: 42 } });
    new Zdog.Shape({ addTo: head, stroke: 5,  color: '#FFFFFF', translate: { x: -19, y: -5, z: 47 } });
    new Zdog.Shape({ addTo: head, stroke: 14, color: color.cheek, translate: { x: 2, y: 20, z: 36 } });

    // sleepy lid — driven by the render loop for blinking (two stacked, API intact)
    const lidL = new Zdog.Shape({ addTo: head, stroke: 24, color: color.faceCream, translate: { x: -16, y: -12, z: 41 } });
    const lidR = new Zdog.Shape({ addTo: head, stroke: 24, color: color.faceCream, translate: { x: -16, y: -12, z: 41 } });

    // brows — leftBrow visible; rightBrow kept (hidden) so the API stays intact
    const leftBrow  = new Zdog.Shape({ addTo: head, path: [{ x: -30, y: -18 }, { x: -6, y: -22 }], stroke: 6, color: color.furDark, translate: { z: 44 } });
    const rightBrow = new Zdog.Shape({ addTo: head, path: [{ x: 20, y: -16 }, { x: 36, y: -18 }],   stroke: 6, color: color.furDark, translate: { z: -30 } });

    // mouth on the snout (re-pathed by setExpression)
    const mouth = new Zdog.Shape({
      addTo: head, stroke: 3.5, color: color.nose, closed: false,
      path: [ { x: -46, y: 20, z: 20 }, { bezier: [{ x: -42, y: 23, z: 20 }, { x: -36, y: 23, z: 20 }, { x: -30, y: 20, z: 20 }] } ]
    });

    // ─── Expression system ─────────────────────────────────────────────
    let lidBaseL = -6, lidBaseR = -6;
    function setExpression(name) {
      let lid = -6, brow = -0.1, my = 20, c0y = 23, c1y = 23;
      if (name === 'sleepy')        { lid = -6;  brow = -0.10; my = 20; c0y = 23; c1y = 23; }
      else if (name === 'content')  { lid = -11; brow = -0.05; my = 19; c0y = 24; c1y = 24; }
      else if (name === 'happy')    { lid = -13; brow =  0.06; my = 17; c0y = 26; c1y = 26; }
      else if (name === 'surprised'){ lid = -15; brow = -0.22; my = 20; c0y = 27; c1y = 27; }
      else if (name === 'grumpy')   { lid = -8;  brow = -0.45; my = 23; c0y = 18; c1y = 18; }
      else if (name === 'yawn')     { lid = -15; brow = -0.18; my = 16; c0y = 32; c1y = 32; }
      lidBaseL = lid; lidBaseR = lid;
      leftBrow.rotate.z  = brow;
      rightBrow.rotate.z = -brow;
      mouth.path = [
        { x: -46, y: my, z: 20 },
        { bezier: [{ x: -42, y: c0y, z: 20 }, { x: -36, y: c1y, z: 20 }, { x: -30, y: my, z: 20 }] }
      ];
      mouth.updatePath();
    }
    setExpression('sleepy');

    // ─── Idle + blink loop (slow — this is a sloth) ────────────────────
    let frame = 0, isRunning = true, isBusy = false, isShaking = false, blinkTimer = 0;

    function render() {
      if (!isRunning) return;
      frame++;

      // slow turntable — parallax is what actually sells the depth
      scene.rotate.y = -0.06 + Math.sin(frame / 130) * 0.09;
      scene.rotate.x = -0.10 + Math.sin(frame / 170) * 0.03;

      if (!isBusy) {
        SLOTH_CONT.translate.y = 40 + Math.sin(frame / 55) * 5;   // slow breathing
        if (!isShaking) PIVOT.rotate.z = Math.sin(frame / 80) * 0.0000001;  // gentle sway
      }
      if (isShaking) PIVOT.rotate.z = Math.sin(frame / 4) * 0.07;

      // occasional slow blink
      blinkTimer++;
      let blink = 0;
      if (blinkTimer > 240) {
        const t = blinkTimer - 240;
        if (t < 16) blink = Math.sin((t / 16) * Math.PI) * 13;
        else blinkTimer = 0;
      }
      lidL.translate.y = lidBaseL + blink;
      lidR.translate.y = lidBaseR + blink;

      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }
    render();

    // ─── Interactions ──────────────────────────────────────────────────
    function triggerWake() {
      if (isBusy) return;
      isBusy = true; isShaking = true;
      setExpression('surprised');
      const tl = gsap.timeline({
        onComplete: () => { setExpression('sleepy'); isBusy = false; isShaking = false; PIVOT.rotate.z = 0; }
      });
      tl.to(leftArm.rotate,  { duration: 0.25, z: 0.15, ease: 'power2.out' });
      tl.to(rightArm.rotate, { duration: 0.25, z: -0.15, ease: 'power2.out' }, '<');
      tl.add(() => setExpression('grumpy'), 0.5);
      tl.to(leftArm.rotate,  { duration: 0.6, z: 0, ease: 'power1.inOut' }, 0.7);
      tl.to(rightArm.rotate, { duration: 0.6, z: 0, ease: 'power1.inOut' }, '<');
      tl.add(() => { isShaking = false; PIVOT.rotate.z = 0; setExpression('content'); }, 1.0);
      tl.to({}, { duration: 0.8 });
      askMossy("Someone just nudged me awake from my nap. Give me short, gentle sloth wisdom about waking slowly and not rushing into the day.");
    }

    function climb() {
      if (isBusy) return;
      isBusy = true;
      setExpression('content');
      const tl = gsap.timeline({ onComplete: () => { setExpression('sleepy'); isBusy = false; } });
      // reach, shuffle along the branch… pause… shuffle back. Everything unhurried.
      tl.to(rightArm.rotate,    { duration: 0.9, z: -0.35, x: 0.2, ease: 'sine.inOut' });
      tl.to(SLOTH_CONT.translate,{ duration: 1.6, x: 150,  ease: 'power1.inOut' }, '-=0.3');
      tl.to(PIVOT.rotate,       { duration: 1.6, z: -0.08, ease: 'power1.inOut' }, '<');
      tl.to(rightArm.rotate,    { duration: 0.7, z: 0, x: 0, ease: 'sine.inOut' }, '<');
      tl.to({}, { duration: 0.6 });
      tl.to(leftArm.rotate,     { duration: 0.9, z: 0.35, x: 0.2, ease: 'sine.inOut' });
      tl.to(SLOTH_CONT.translate,{ duration: 1.8, x: 0,   ease: 'power1.inOut' }, '-=0.3');
      tl.to(PIVOT.rotate,       { duration: 1.8, z: 0,    ease: 'power1.inOut' }, '<');
      tl.to(leftArm.rotate,     { duration: 0.7, z: 0, x: 0, ease: 'sine.inOut' }, '<');
    }

    function yawn() {
      if (isBusy) return;
      isBusy = true;
      const tl = gsap.timeline({ onComplete: () => { setExpression('sleepy'); isBusy = false; } });
      tl.add(() => setExpression('content'));
      tl.to(head.rotate,        { duration: 0.6, x: -0.15, ease: 'sine.inOut' });
      tl.add(() => setExpression('yawn'), 0.6);
      tl.to([leftArm.rotate, rightArm.rotate, leftLeg.rotate, rightLeg.rotate], { duration: 0.9, x: -0.4, ease: 'power2.out' }, '<');      tl.to(SLOTH_CONT.translate, { duration: 0.9, y: 28, ease: 'power2.out' }, '<');
      tl.to({}, { duration: 1.0 });                       // hold the yawn
      tl.add(() => setExpression('content'));
      tl.to([leftArm.rotate, rightArm.rotate, leftLeg.rotate, rightLeg.rotate], { duration: 1.0, x: 0, ease: 'power1.inOut' });
      tl.to(head.rotate,        { duration: 1.0, x: 0, ease: 'power1.inOut' }, '<');
      tl.to(SLOTH_CONT.translate, { duration: 1.0, y: 40, ease: 'power1.inOut' }, '<');
    }

    function snack() {
      if (isBusy) return;
      isBusy = true;
      setExpression('happy');
      const tl = gsap.timeline({ onComplete: () => { setExpression('content'); isBusy = false; } });
      // lean toward the hibiscus, reach, then slow contented munches
      tl.to(SLOTH_CONT.translate, { duration: 1.2, x: 90, y: 26, ease: 'power1.inOut' });
      tl.to(PIVOT.rotate,        { duration: 1.2, y: 0.3, z: -0.1, ease: 'power1.inOut' }, '<');
      tl.to(rightArm.rotate,     { duration: 0.6, z: -0.6, x: 0.5, ease: 'back.out(1.4)' }, '-=0.4');
      tl.to(rightArm.rotate,     { duration: 0.5, z: -0.2, x: 0.2, ease: 'sine.inOut' });
      for (let i = 0; i < 3; i++) {                         // munch, munch, munch
        tl.to(head.rotate, { duration: 0.28, x: -0.12, ease: 'sine.inOut' });
        tl.to(head.rotate, { duration: 0.28, x: 0,     ease: 'sine.inOut' });
      }
      tl.to(rightArm.rotate,     { duration: 0.6, z: 0, x: 0, ease: 'power2.in' });
      tl.to(SLOTH_CONT.translate, { duration: 1.4, x: 0, y: 40, ease: 'power1.inOut' });
      tl.to(PIVOT.rotate,        { duration: 1.4, y: 0, z: 0, ease: 'power1.inOut' }, '<');
      askMossy("I just had a sweet hibiscus flower, my favorite treat. Share a short, dreamy thought about savoring small pleasures slowly.");
    }

    function talkingAnimation() {
      if (isBusy) return;
      isBusy = true;
      setExpression('content');
      const tl = gsap.timeline({ onComplete: () => { setExpression('sleepy'); isBusy = false; } });
      tl.to(head.rotate,     { duration: 0.7, z: 0.06, ease: 'sine.inOut' });
      tl.to(leftArm.rotate,  { duration: 0.7, z: 0.18, x: 0.1, ease: 'sine.inOut' }, '<');
      tl.to(head.rotate,     { duration: 0.7, z: -0.05, ease: 'sine.inOut' });
      tl.to(leftArm.rotate,  { duration: 0.7, z: 0, x: 0, ease: 'sine.inOut' }, '<');
      tl.to(head.rotate,     { duration: 0.7, z: 0, ease: 'sine.inOut' });
    }

    triggerWakeFn = triggerWake;
    climbFn       = climb;
    yawnFn        = yawn;
    snackFn       = snack;
    talkingFn     = talkingAnimation;

    // ─── Pointer: drag to stir the sloth awake ─────────────────────────
    const hibiscusZone = { x: 250, y: -110, r: 90 };
    let isDragging = false, lastX = 0, lastY = 0, wasDragging = false, dragDist = 0;

    function onPointerDown(e) {
      isDragging = true; wasDragging = false; dragDist = 0;
      lastX = e.clientX; lastY = e.clientY;
      try { canvasRef.setPointerCapture(e.pointerId); } catch (_) {}
    }
    function onPointerMove(e) {
      if (!isDragging) return;
      const dx = e.clientX - lastX, dy = e.clientY - lastY;
      lastX = e.clientX; lastY = e.clientY;
      const dist = Math.sqrt(dx * dx + dy * dy);
      dragDist += dist;
      if (dist > 10) wasDragging = true;
      
      // Drag rotation has been removed
      
      scene.updateRenderGraph();
    }
    function onPointerUp() {
      isDragging = false;
      if (dragDist > 70 && !isBusy) triggerWake();   // a real tug wakes Mossy
    }

    function onCanvasClick(e) {
      if (wasDragging) return;
      const rect = canvasRef.getBoundingClientRect();
      const cx = (e.clientX - rect.left - rect.width  / 2);
      const cy = (e.clientY - rect.top  - rect.height / 2);

      if (Math.hypot(cx - hibiscusZone.x, cy - hibiscusZone.y) < hibiscusZone.r) {
        snack(); return;
      }
      // Tapping the sloth opens the radial menu
      if (Math.hypot(cx, cy - 10) < 200) {
        radialMenuOpen = !radialMenuOpen;
        activeCard = null;
      }
    }

    if (!embedded) {
      canvasRef.addEventListener('click',       onCanvasClick);
      canvasRef.addEventListener('pointerdown', onPointerDown);
      canvasRef.addEventListener('pointermove', onPointerMove);
      window.addEventListener('pointerup',      onPointerUp);
    }

    // First thought after a lazy pause, then keep them coming slowly
    if (!embedded) {
      setTimeout(() => showThought("Mmm… I'm thinking about rest and patience. What about you? 🌿"), 4000);
      scheduleThought();
    }

    return () => {
      isRunning = false;
      if (thoughtTimer) clearTimeout(thoughtTimer);
      if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
      canvasRef?.removeEventListener('click',       onCanvasClick);
      canvasRef?.removeEventListener('pointerdown', onPointerDown);
      canvasRef?.removeEventListener('pointermove', onPointerMove);
      window.removeEventListener('pointerup',       onPointerUp);
    };
  });
</script>

{#if !embedded}
  <div class="top-bar">
    <button class="bar-btn" on:click={goHome}>← Home</button>
    <button class="bar-btn" on:click={toggleHistory}>
      {historyOpen ? 'Close history' : '💬 History'}
    </button>
  </div>
{/if}

{#if !embedded && historyOpen}
  <div class="history-panel">
    <div class="history-header">
      <h2>Session chat</h2>
      <button class="close-btn" on:click={() => historyOpen = false}>✕</button>
    </div>
    <div class="history-scroll">
      {#each chatHistory as msg}
        <div class="history-msg {msg.role}">
          <span class="history-label">{msg.role === 'assistant' ? '🦥 Mossy' : 'You'}</span>
          <p>{msg.content}</p>
        </div>
      {/each}
    </div>
  </div>
{/if}

{#if !embedded}
  <SlothBackground />
{/if}
<canvas bind:this={canvasRef} class="scene" class:embedded></canvas>

{#if !embedded && radialMenuOpen}
  <div class="radial-overlay">
    {#each interactionCards as card, i}
      {@const angle = (i / interactionCards.length) * 360 - 90}
      {@const rad   = angle * (Math.PI / 180)}
      {@const r     = 110}
      <button
        class="radial-btn"
        style="
          left:  calc(50% + {Math.cos(rad) * r}px - 36px);
          top:   calc(50% + {Math.sin(rad) * r}px - 36px);
          animation-delay: {i * 55}ms;
        "
        on:click={() => openCard(card.id)}
      >
        <span class="radial-icon">{card.icon}</span>
        <span class="radial-label">{card.label}</span>
      </button>
    {/each}
    <button class="radial-center" on:click={() => { radialMenuOpen = false; }}>✕</button>
  </div>
{/if}

{#if activeCard}
  <div class="card-overlay" on:click|self={() => activeCard = null}>
    <div class="card">
      <div class="card-icon">{activeCard.icon}</div>
      <h2>{activeCard.title}</h2>
      <p class="card-desc">{activeCard.description}</p>
      <p class="card-hint">✦ {activeCard.hint}</p>
      <div class="card-actions">
        <button class="card-btn primary" on:click={executeCard}>Activate</button>
        <button class="card-btn ghost"   on:click={closeCard}>Cancel</button>
      </div>
    </div>
  </div>
{/if}

<div class="thought-wrap" class:visible={thoughtBubbleVisible}>
  <div class="thought-bubble">
    <div class="thought-dots">
      <span></span><span></span><span></span>
    </div>
    <p class="thought-text">{thoughtBubbleText}</p>
  </div>
  <div class="thought-reply">
    <textarea
      bind:value={replyInputValue}
      placeholder="Reply to Mossy…"
      rows="1"
      disabled={isApiLoading}
      on:keydown={handleReplyKey}
    ></textarea>
    <button class="reply-send" on:click={sendReply} disabled={isApiLoading || !replyInputValue.trim()}>
      {isApiLoading ? '…' : '➤'}
    </button>
  </div>
</div>

<p class="hint">Tap the sloth to open interactions 🌿</p>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap");

  :global(body) {
    margin: 0; padding: 0;
    width: 100vw; height: 100vh;
    background: transparent; /* Changed from #C9ECC0 to completely remove background color */
    overflow: hidden;
    font-family: 'Nunito', sans-serif;
  }

  canvas.scene {
    position: fixed; inset: 0;
    width: 100vw; height: 100vh;
    display: block; z-index: 1;
    touch-action: none;
  }

  canvas.scene.embedded {
    position: absolute;
    width: 100%;
    height: 100%;
    inset: 0;
  }

  /* ── Top bar ─────────────────────────────────────────────────────── */
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
    color: #2E3D22;
    font-family: 'Nunito', sans-serif;
    font-weight: 700; font-size: 0.85rem;
    cursor: pointer;
    box-shadow: 0 4px 18px rgba(40,55,30,.12);
    transition: background .15s, transform .1s;
  }
  .bar-btn:hover { background: #fff; transform: translateY(-1px); }
  .bar-btn:active { transform: scale(.96); }

  /* ── History panel ───────────────────────────────────────────────── */
  .history-panel {
    position: fixed;
    top: 62px; right: 16px;
    width: min(360px, calc(100vw - 32px));
    max-height: calc(100vh - 80px);
    background: rgba(255,255,255,0.97);
    border-radius: 24px;
    box-shadow: 0 20px 60px rgba(40,55,30,.18);
    z-index: 25;
    display: flex; flex-direction: column;
    overflow: hidden;
  }
  .history-header {
    display: flex; justify-content: space-between; align-items: center;
    padding: 18px 20px 12px;
    border-bottom: 1px solid rgba(93,160,82,.16);
  }
  .history-header h2 { margin: 0; font-size: 1rem; color: #2E3D22; }
  .close-btn {
    border: none; background: #e6f1dd;
    color: #3E6B2E; border-radius: 50%;
    width: 30px; height: 30px;
    cursor: pointer; font-size: 0.85rem; font-weight: 700;
  }
  .history-scroll {
    overflow-y: auto; padding: 14px 16px;
    display: flex; flex-direction: column; gap: 10px;
  }
  .history-msg {
    border-radius: 16px; padding: 12px 14px;
    font-size: 0.9rem; line-height: 1.55;
  }
  .history-msg p { margin: 4px 0 0; color: #243018; }
  .history-msg.assistant { background: #eef6e8; }
  .history-msg.user { background: #fbf7ec; }
  .history-label { font-weight: 800; font-size: 0.78rem; color: #4E7D3A; }

  /* ── Radial menu ─────────────────────────────────────────────────── */
  .radial-overlay { position: fixed; inset: 0; z-index: 20; pointer-events: none; }
  .radial-btn {
    position: fixed;
    width: 72px; height: 72px;
    border: none; border-radius: 50%;
    background: rgba(255,255,255,0.95);
    box-shadow: 0 6px 28px rgba(40,55,30,.18);
    cursor: pointer;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    gap: 2px;
    pointer-events: all;
    animation: radialPop .25s cubic-bezier(.34,1.56,.64,1) both;
    transition: transform .12s, background .12s;
  }
  .radial-btn:hover { background: #7BBE5E; transform: scale(1.12); }
  .radial-btn:hover .radial-label { color: #fff; }
  @keyframes radialPop {
    from { transform: scale(0); opacity: 0; }
    to   { transform: scale(1); opacity: 1; }
  }
  .radial-icon  { font-size: 1.5rem; line-height: 1; }
  .radial-label { font-size: 0.65rem; font-weight: 800; color: #3E6B2E; letter-spacing: .02em; }
  .radial-center {
    position: fixed;
    left:  calc(50% - 24px);
    top:   calc(50% - 24px);
    width: 48px; height: 48px;
    border: none; border-radius: 50%;
    background: rgba(123,190,94,.92);
    color: #fff;
    font-size: 1rem; font-weight: 800;
    cursor: pointer;
    box-shadow: 0 4px 20px rgba(40,55,30,.18);
    pointer-events: all;
    animation: radialPop .2s cubic-bezier(.34,1.56,.64,1) both;
    transition: background .12s, transform .1s;
    z-index: 21;
  }
  .radial-center:hover { background: #3E7D3A; transform: scale(1.08); }

  /* ── Interaction card ────────────────────────────────────────────── */
  .card-overlay {
    position: fixed; inset: 0;
    background: rgba(30, 45, 20, 0.30);
    backdrop-filter: blur(4px);
    z-index: 30;
    display: flex; align-items: center; justify-content: center;
  }
  .card {
    background: #fff;
    border-radius: 28px;
    padding: 32px 28px;
    max-width: 340px; width: calc(100vw - 48px);
    box-shadow: 0 28px 80px rgba(40,55,30,.22);
    text-align: center;
    animation: cardIn .22s cubic-bezier(.34,1.56,.64,1) both;
  }
  @keyframes cardIn {
    from { transform: scale(.88); opacity: 0; }
    to   { transform: scale(1);   opacity: 1; }
  }
  .card-icon { font-size: 2.8rem; margin-bottom: 8px; }
  .card h2   { margin: 0 0 10px; font-size: 1.1rem; color: #2E3D22; }
  .card-desc { margin: 0 0 8px; color: #4d6b3a; line-height: 1.6; font-size: 0.95rem; }
  .card-hint { margin: 0 0 24px; color: #8aa86f; font-size: 0.85rem; font-style: italic; }
  .card-actions { display: flex; gap: 10px; }
  .card-btn {
    flex: 1; border: none; border-radius: 16px;
    padding: 13px 0; font-family: 'Nunito', sans-serif;
    font-weight: 800; font-size: 0.9rem; cursor: pointer;
    transition: background .12s, transform .1s;
  }
  .card-btn:active { transform: scale(.96); }
  .card-btn.primary { background: #5DA052; color: #fff; }
  .card-btn.primary:hover { background: #3E7D3A; }
  .card-btn.ghost   { background: #e6f1dd; color: #3E6B2E; }
  .card-btn.ghost:hover { background: #d6e8c8; }

  /* ── Thought bubble ──────────────────────────────────────────────── */
  .thought-wrap {
    position: fixed;
    top: 32%;
    left: 56%;
    max-width: min(310px, 40vw);
    display: flex; flex-direction: column; gap: 10px;
    z-index: 15;
    pointer-events: none;
    opacity: 0;
    transform: translateY(8px) scale(.96);
    transition: opacity .35s ease, transform .35s ease;
  }
  .thought-wrap.visible {
    opacity: 1;
    transform: translateY(0) scale(1);
    pointer-events: all;
  }
  .thought-bubble {
    background: #fff;
    border: 2px solid rgba(93,160,82,.30);
    border-radius: 22px 22px 22px 6px;
    padding: 14px 18px;
    box-shadow: 0 8px 36px rgba(40,55,30,.14);
    position: relative;
  }
  .thought-dots { display: flex; gap: 4px; margin-bottom: 6px; }
  .thought-dots span { width: 5px; height: 5px; background: #7BBE5E; border-radius: 50%; }
  .thought-text {
    margin: 0;
    font-size: 0.92rem;
    color: #2A3A1C;
    line-height: 1.6;
    font-weight: 600;
  }
  .thought-reply { display: flex; gap: 8px; align-items: flex-end; }
  .thought-reply textarea {
    flex: 1;
    background: rgba(255,255,255,0.96);
    border: 1.5px solid rgba(93,160,82,.40);
    border-radius: 14px;
    padding: 10px 14px;
    font-family: 'Nunito', sans-serif;
    font-size: 0.9rem;
    color: #243018;
    outline: none; resize: none;
    height: 44px; min-height: 44px; max-height: 88px;
    line-height: 1.5; box-sizing: border-box;
    transition: border-color .15s, box-shadow .15s;
  }
  .thought-reply textarea::placeholder { color: #9bb487; }
  .thought-reply textarea:focus {
    border-color: #3E7D3A;
    box-shadow: 0 0 0 3px rgba(62,125,58,.18);
  }
  .reply-send {
    width: 44px; height: 44px;
    border-radius: 50%; border: none;
    background: #5DA052; color: #fff;
    font-size: 1rem; cursor: pointer;
    flex-shrink: 0;
    transition: background .12s, transform .1s;
    display: flex; align-items: center; justify-content: center;
  }
  .reply-send:hover:not(:disabled) { background: #3E7D3A; }
  .reply-send:active:not(:disabled) { transform: scale(.94); }
  .reply-send:disabled { opacity: .5; cursor: default; }

  /* ── Hint ────────────────────────────────────────────────────────── */
  .hint {
    position: fixed;
    bottom: 22px; left: 50%;
    transform: translateX(-50%);
    font-size: 0.8rem; font-weight: 600;
    color: rgba(40, 55, 30, 0.55);
    pointer-events: none;
    letter-spacing: .04em;
    z-index: 10;
  }
  .credit {
    position: fixed; bottom: 8px; left: 14px;
    font-size: 0.7rem; color: rgba(40,55,30,.42);
    z-index: 10; pointer-events: none;
  }
</style>