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
      moss:       '#7E8B5A',
      branch:     '#6B4A2A',
      branchDk:   '#4E3318',
      leafGreen:  '#5DA052',
      leafDark:   '#3A6A28',
      leafLight:  '#7FC04A',
      canopy1:    '#A7D88C',
      canopy2:    '#84C268',
      canopyDk:   '#4E7D3A',
      skyTop:     '#C9ECC0',
      skyLow:     '#E7F6DF',
      hibiscusR:  '#F0584C',
      hibiscusC:  '#FFC94D',
      vine:       '#4E7D3A'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.16, y: -0.18, z: 0 }   // off-axis so the form has a near and far side
    });

    // background is rendered in SlothBackground.svelte

    // ─── The Sloth ─────────────────────────────────────────────────────
    const SLOTH_CONT = new Zdog.Anchor({ addTo: scene, translate: { x: 0, y: 30, z: 80 }, scale: 1.4 });
    const PIVOT = new Zdog.Anchor({ addTo: SLOTH_CONT });
    const SLOTH = new Zdog.Anchor({ addTo: PIVOT });

    // Long arms reaching up to grip the branch, with three claws each
    // Four limbs reaching up to grip the branch, with three claws each
    // Body — longer, pear-shaped hanging torso
    const bodyAnchor = new Zdog.Anchor({
      addTo: SLOTH,
      translate: { y: 38, z: -28 },
      rotate: { z: -0.04 }
    });

    // ── Torso from overlapping spheres → real volume front-to-back ──
    new Zdog.Shape({ addTo: bodyAnchor, stroke: 138, color: color.fur });
    new Zdog.Shape({ addTo: bodyAnchor, stroke: 120, color: color.furDark, translate: { y: 20, z: -34 } }); // back bulge
    new Zdog.Shape({ addTo: bodyAnchor, stroke: 108, color: color.furDark, translate: { y: 46, z: -20 } }); // rump

    // Belly that actually bulges toward the viewer
    [
      { y: -34, z: 46, s: 60 },
      { y:   2, z: 54, s: 74 },
      { y:  34, z: 50, s: 66 },
      { y:  60, z: 38, s: 48 }
    ].forEach(b => {
      new Zdog.Shape({ addTo: bodyAnchor, stroke: b.s, color: color.furLight, translate: { y: b.y, z: b.z } });
    });
    new Zdog.Shape({ addTo: bodyAnchor, stroke: 50, color: color.faceCream, translate: { y: 10, z: 66 } });

    // fur/algae tufts as little spheres on the belly surface (not flat patches)
    [
      { x: -30, y: -30, z: 72, s: 17, c: color.moss },
      { x:  28, y:   6, z: 76, s: 14, c: color.moss },
      { x: -16, y:  46, z: 66, s: 15, c: color.furDark }
    ].forEach(p => {
      new Zdog.Shape({ addTo: bodyAnchor, stroke: p.s, color: p.c, translate: { x: p.x, y: p.y, z: p.z } });
    });

  // Natural curved limbs: arms longer, back legs slightly behind and lower
  function makeLimb(sx, baseX, baseY, midX, endX, endY, z, stroke = 20) {
    const limb = new Zdog.Anchor({
      addTo: SLOTH,
      translate: { x: baseX * sx, y: baseY, z }
    });

    new Zdog.Shape({
      addTo: limb,
      path: [
        { x: 0, y: 0, z: 0 },
        {
          bezier: [
            { x: midX * sx, y: -55, z: -8 },
            { x: endX * sx, y: -150, z: -18 },
            { x: endX * sx, y: endY, z: -28 }
          ]
        }
      ],
      stroke,
      color: color.fur,
      fill: false
    });

    const claw = new Zdog.Anchor({
      addTo: limb,
      translate: { x: endX * sx, y: endY - 4, z: -28 },
      rotate: { z: -0.18 * sx }
    });

    for (let i = 0; i < 3; i++) {
      new Zdog.Shape({
        addTo: claw,
        path: [
          { x: (i - 1) * 8, y: 0 },
          { bezier: [
            { x: (i - 1) * 8 - 3 * sx, y: 8 },
            { x: (i - 1) * 8 - 6 * sx, y: 15 },
            { x: (i - 1) * 8 - 4 * sx, y: 23 }
          ]}
        ],
        stroke: 6,
        color: color.claw,
        closed: false
      });
    }

    return limb;
  }

  const leftArm  = makeLimb(-1, 46, 8, 20, 24, -244, 20, 22);
  const rightArm = makeLimb( 1, 46, 8, 20, 24, -244, 20, 22);

  const leftLeg  = makeLimb(-1, 34, 88, 30, 68, -236, -24, 19);
  const rightLeg = makeLimb( 1, 34, 88, 30, 68, -236, -24, 19);

  // Head
  const head = new Zdog.Anchor({ addTo: SLOTH, translate: { x: 0, y: -78, z: 10 } });
  new Zdog.Shape({ addTo: head, stroke: 128, color: color.fur });
  new Zdog.Ellipse({ addTo: head, width: 96, height: 84, stroke: 16, color: color.faceCream, fill: true, translate: { z: 50 } });

  // Dark eye-mask stripes (the classic sloth look)
  new Zdog.Ellipse({ addTo: head, width: 30, height: 50, stroke: 6, color: color.mask, fill: true, translate: { x: -26, y: -4, z: 58 }, rotate: { z:  0.35 } });
  new Zdog.Ellipse({ addTo: head, width: 30, height: 50, stroke: 6, color: color.mask, fill: true, translate: { x:  26, y: -4, z: 58 }, rotate: { z: -0.35 } });

  // Eyes + highlights
  new Zdog.Shape({ addTo: head, stroke: 15, color: color.eye, translate: { x: -24, y: -2, z: 66 } });
  new Zdog.Shape({ addTo: head, stroke: 15, color: color.eye, translate: { x:  24, y: -2, z: 66 } });
  new Zdog.Shape({ addTo: head, stroke: 5,  color: '#FFFFFF', translate: { x: -27, y: -5, z: 72 } });
  new Zdog.Shape({ addTo: head, stroke: 5,  color: '#FFFFFF', translate: { x:  21, y: -5, z: 72 } });

  // Sleepy lids — cream caps we slide down over the eyes to blink / look drowsy
  const lidL = new Zdog.Shape({ addTo: head, stroke: 16, color: color.faceCream, translate: { x: -24, y: -12, z: 67 } });
  const lidR = new Zdog.Shape({ addTo: head, stroke: 16, color: color.faceCream, translate: { x:  24, y: -12, z: 67 } });

  // Cheeks + nose
  // Rounder cheeks + a muzzle that bulges forward, with a protruding nose
  new Zdog.Shape({ addTo: head, stroke: 18, color: color.cheek, translate: { x: -40, y: 14, z: 44 } });
  new Zdog.Shape({ addTo: head, stroke: 18, color: color.cheek, translate: { x:  40, y: 14, z: 44 } });
  new Zdog.Shape({ addTo: head, stroke: 50, color: color.faceCream, translate: { x: 0, y: 13, z: 52 } }); // muzzle
  new Zdog.Shape({ addTo: head, stroke: 16, color: color.nose,  translate: { x: 0, y: 8, z: 74 } });        // nose tip
  new Zdog.Shape({ addTo: head, stroke: 6,  color: '#000',      translate: { x: -4, y: 9, z: 80 } });        // nostrils
  new Zdog.Shape({ addTo: head, stroke: 6,  color: '#000',      translate: { x:  4, y: 9, z: 80 } });
  // Brows + mouth (mouth is a bezier we re-path for expressions)
  const leftBrow  = new Zdog.Shape({ addTo: head, path: [{ x: -36, y: -26 }, { x: -14, y: -30 }], stroke: 7, color: color.furDark, translate: { z: 58 }, rotate: { z: -0.1 } });
  const rightBrow = new Zdog.Shape({ addTo: head, path: [{ x:  14, y: -30 }, { x:  36, y: -26 }], stroke: 7, color: color.furDark, translate: { z: 58 }, rotate: { z:  0.1 } });
  const mouth = new Zdog.Shape({
    addTo: head, stroke: 3.5, color: color.nose, closed: false,
    path: [{ x: -6, y: 22, z: 60 }, { bezier: [{ x: -3, y: 25, z: 60 }, { x: 3, y: 25, z: 60 }, { x: 6, y: 22, z: 60 }] }]
  });

  // ─── Expression system ─────────────────────────────────────────────
  // lidBase is the resting lid height for the current mood; the blink in the
  // render loop is added on top so blinking works in every expression.
  let lidBaseL = -6, lidBaseR = -6;
  function setExpression(name) {
    let lid = -6, brow = -0.1, mL = -6, mR = 6, my = 22, c0y = 25, c1y = 25;
    if (name === 'sleepy')        { lid = -6;  brow = -0.1;  my = 22; c0y = 25; c1y = 25; }
    else if (name === 'content')  { lid = -12; brow = -0.05; my = 20; c0y = 27; c1y = 27; }
    else if (name === 'happy')    { lid = -14; brow =  0.05; my = 19; c0y = 29; c1y = 29; mL = -8; mR = 8; }
    else if (name === 'surprised'){ lid = -16; brow = -0.25; my = 22; c0y = 30; c1y = 30; mL = -5; mR = 5; }
    else if (name === 'grumpy')   { lid = -8;  brow = -0.5;  my = 25; c0y = 20; c1y = 20; }
    else if (name === 'yawn')     { lid = -16; brow = -0.2;  my = 18; c0y = 38; c1y = 38; mL = -9; mR = 9; }
    lidBaseL = lid; lidBaseR = lid;
    leftBrow.rotate.z  = brow;
    rightBrow.rotate.z = -brow;
    mouth.path = [
      { x: mL, y: my, z: 60 },
      { bezier: [{ x: -3, y: c0y, z: 60 }, { x: 3, y: c1y, z: 60 }, { x: mR, y: my, z: 60 }] }
    ];
    mouth.updatePath();
  }
  setExpression('sleepy');

  // ─── Idle + blink loop (slow — this is a sloth) ────────────────────
  let frame = 0, isRunning = true, isBusy = false, isShaking = false, blinkTimer = 0;

  function render() {
    if (!isRunning) return;
    frame++;

    if (!isBusy) {
      SLOTH_CONT.translate.y = 30 + Math.sin(frame / 55) * 5;   // slow breathing
      if (!isShaking) PIVOT.rotate.z = Math.sin(frame / 80) * 0.04;  // gentle sway
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
    // reach, shuffle right… pause… shuffle back. Everything unhurried.
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
    tl.to([leftArm.rotate, rightArm.rotate, leftLeg.rotate, rightLeg.rotate], { duration: 0.9, x: -0.4, ease: 'power2.out' }, '<');      tl.to(SLOTH_CONT.translate, { duration: 0.9, y: 18, ease: 'power2.out' }, '<');
    tl.to({}, { duration: 1.0 });                       // hold the yawn
    tl.add(() => setExpression('content'));
    tl.to([leftArm.rotate, rightArm.rotate, leftLeg.rotate, rightLeg.rotate], { duration: 1.0, x: 0, ease: 'power1.inOut' });
    tl.to(head.rotate,        { duration: 1.0, x: 0, ease: 'power1.inOut' }, '<');
    tl.to(SLOTH_CONT.translate, { duration: 1.0, y: 30, ease: 'power1.inOut' }, '<');
  }

  function snack() {
    if (isBusy) return;
    isBusy = true;
    setExpression('happy');
    const tl = gsap.timeline({ onComplete: () => { setExpression('content'); isBusy = false; } });
    // lean toward the hibiscus, reach, then slow contented munches
    tl.to(SLOTH_CONT.translate, { duration: 1.2, x: 90, y: 16, ease: 'power1.inOut' });
    tl.to(PIVOT.rotate,        { duration: 1.2, y: 0.3, z: -0.1, ease: 'power1.inOut' }, '<');
    tl.to(rightArm.rotate,     { duration: 0.6, z: -0.6, x: 0.5, ease: 'back.out(1.4)' }, '-=0.4');
    tl.to(rightArm.rotate,     { duration: 0.5, z: -0.2, x: 0.2, ease: 'sine.inOut' });
    for (let i = 0; i < 3; i++) {                         // munch, munch, munch
      tl.to(head.rotate, { duration: 0.28, x: -0.12, ease: 'sine.inOut' });
      tl.to(head.rotate, { duration: 0.28, x: 0,     ease: 'sine.inOut' });
    }
    tl.to(rightArm.rotate,     { duration: 0.6, z: 0, x: 0, ease: 'power2.in' });
    tl.to(SLOTH_CONT.translate, { duration: 1.4, x: 0, y: 30, ease: 'power1.inOut' });
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
  const sensitivity = 0.006;

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
    PIVOT.rotate.y += dx * sensitivity;
    PIVOT.rotate.x += dy * sensitivity;
    PIVOT.rotate.x  = Math.max(-TAU / 9, Math.min(TAU / 9, PIVOT.rotate.x));
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
    if (Math.hypot(cx, cy - 10) < 170) {
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

<!-- ─── Top bar ──────────────────────────────────────────────────────── -->
{#if !embedded}
  <div class="top-bar">
    <button class="bar-btn" on:click={goHome}>← Home</button>
    <button class="bar-btn" on:click={toggleHistory}>
      {historyOpen ? 'Close history' : '💬 History'}
    </button>
  </div>
{/if}

<!-- ─── Chat history panel ─────────────────────────────────────────── -->
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

<!-- ─── Canvas ─────────────────────────────────────────────────────── -->
{#if !embedded}
  <SlothBackground />
{/if}
<canvas bind:this={canvasRef} class="scene" class:embedded></canvas>

<!-- ─── Radial menu ────────────────────────────────────────────── -->
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

<!-- ─── Interaction card ────────────────────────────────────────────── -->
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

<!-- ─── Thought bubble ─────────────────────────────────────────────── -->
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

<!-- ─── Hint ───────────────────────────────────────────────────────── -->
<p class="hint">Tap the sloth to open interactions 🌿</p>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap");

  :global(body) {
    margin: 0; padding: 0;
    width: 100vw; height: 100vh;
    background: #C9ECC0;
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