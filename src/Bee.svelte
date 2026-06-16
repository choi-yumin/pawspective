<script>
  //@ts-nocheck
  import { onMount, onDestroy, createEventDispatcher } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';
  import BeeBackground from './BeeBackground.svelte';

  const dispatch = createEventDispatcher();

  export let embedded = false;
  let canvasRef;
  let pageBodyClass = 'bee-page';

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
      id: 'annoy',
      icon: '😤',
      label: 'Annoy',
      title: 'Pull the bee\'s tail',
      description: 'Grab and drag the bee hard to make it flash red with anger.',
      hint: 'Drag the bee quickly to trigger annoyance.',
      angle: -90
    },
    {
      id: 'pollen',
      icon: '🌸',
      label: 'Pollen',
      title: 'Gather pollen',
      description: 'Click any of the flowers scattered around the garden to send the bee collecting.',
      hint: 'Tap the flowers in the foreground.',
      angle: 0
    },
    {
      id: 'chat',
      icon: '💬',
      label: 'Chat',
      title: 'Speak to the bee',
      description: 'The bee has thoughts and wisdom about effort, rest, and how others see you.',
      hint: 'Reply to the bee\'s thought bubbles below.',
      angle: 90
    }
  ];

  let chatHistory = [
    { role: 'assistant', content: "Buzz! 🌸 Ask me about effort, rest, or what others think of you!" }
  ];

  const SYSTEM_TEXT = `You are Buzz, an adorable wise bee who lives in a flower garden. You speak in short, warm, whimsical messages (2-4 sentences max). You use flower and bee metaphors naturally.
Your specialty is helping people think about:
- How others perceive them (social perception, first impressions, reputation)
- Diligence and hard work (like a bee — constant, purposeful effort)
- Laziness and rest (the importance of pause vs. the trap of avoidance)
You give gentle, insightful, slightly playful advice. You don't lecture. Keep it warm, short, and wise.`;

  const thoughtPrompts = [
    'The garden is asking a lot today. What do you think?',
    'I am thinking about hard work and sweet rest.',
    'If I fly too much, I wonder if the flowers miss me.',
    'Does being busy make us feel better or more tired?',
    'Every drop of nectar matters — even the ones you can\'t see yet.',
    'What do you think the other bees say about me?'
  ];

  let triggerAnnoyanceFn;
  let gatherPollenFn;
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
    if (activeCard.id === 'annoy') triggerAnnoyanceFn?.();
    if (activeCard.id === 'pollen') {
      // Pick a random flower zone when executed from the generic overlay action menu
      const randomFlower = flowerZones[Math.floor(Math.random() * flowerZones.length)];
      gatherPollenFn?.(randomFlower);
    }
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
      }, 12000);
    }
  }

  function scheduleThought() {
    if (thoughtTimer) clearTimeout(thoughtTimer);
    thoughtTimer = setTimeout(() => {
      const prompt = thoughtPrompts[Math.floor(Math.random() * thoughtPrompts.length)];
      showThought(prompt);
      scheduleThought();
    }, 16000 + Math.random() * 12000);
  }

  async function sendReply() {
    const msg = replyInputValue.trim();
    if (!msg || isApiLoading) return;
    replyInputValue = '';
    thoughtBubbleVisible = false;
    await askBee(msg);
  }

  function handleReplyKey(e) {
    if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); sendReply(); }
  }

  async function askBee(userMessage) {
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
        showThought("Sorry, I can't think of anything to answer. Try again?", true);
      }
    } catch (e) {
      console.error('OpenAI Catch block triggered:', e);
      chatHistory = chatHistory.slice(0, -1);
      showThought("Sorry, I can't think of anything to answer. Try again?", true);
    } finally {
      isApiLoading = false;
    }
  }

  // Exact 3D target coordinates from your background layout grid.
  // cx and cy map 2D click spaces relative to screen projection factors.
  const flowerZones = [
    { x: -600, y: 120, z: -150, cx: -570, cy: 150, r: 85 },
    { x: -350, y: 140, z:   80, cx: -350, cy: 110, r: 90 },
    { x: -120, y:  95, z: -280, cx: -100, cy: 160, r: 80 },
    { x:  100, y: 135, z:   20, cx:  100, cy: 120, r: 90 },
    { x:  320, y: 100, z: -320, cx:  340, cy: 170, r: 80 },
    { x:  520, y: 145, z:   90, cx:  510, cy: 110, r: 90 },
    { x:  700, y: 110, z: -180, cx:  680, cy: 150, r: 85 }
  ];

  onMount(() => {
    if (!embedded) document.body.classList.add(pageBodyClass);
    if (!canvasRef) return;
    const TAU = Zdog.TAU;

    const color = {
      beeYellow:   '#FCD116',
      beeBlack:    '#3D2620',
      beeWhite:    '#FFFDF0',
      beeCheek:    '#F26B50',
      leafGreen:   '#5DA020',
      darkGreen:   '#3A5C18',
      land:        '#7ab535',
      landLight:   '#9fd95c',
      trunkBrown:  '#8B6340',
      cloudPink:   '#FFE4F0',
      cloudWhite:  '#FFF7FB',
      daisyWhite:  '#FFFFFF',
      daisyYellow: '#F9D342',
      roseRed:     '#F04A6F',
      rosePink:    '#FF8FAE',
      lavPurple:   '#C07EE0',
      lavLight:    '#E2B8F5',
      sunflowerY:  '#FFB800',
      sunflowerC:  '#7A3B10',
      tulipPink:   '#FF85A2',
      tulipOrange: '#FF6B35',
      honeyGold:   '#F2C94C',
      honeyDark:   '#D4A017',
      beeAngryRed: '#FF3B30'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.28, y: -0.12, z: 0 }
    });

    const fgGroup = new Zdog.Anchor({ addTo: scene, translate: { x: 0, y: 50, z: 120 } });

    let BEE_CONT  = new Zdog.Anchor({ addTo: fgGroup, translate: { x: 0, y: -100, z: 0 }, scale: 1.55 });
    let BEE_PIVOT = new Zdog.Anchor({ addTo: BEE_CONT, translate: { x: 0, y: 32, z: -35 }, rotate: { y: 0 } });
    let BEE       = new Zdog.Anchor({ addTo: BEE_PIVOT, translate: { x: 0, y: -32, z: 35 } });

    let beeHead = new Zdog.Shape({ addTo: BEE, stroke: 130, color: color.beeYellow });
    new Zdog.Shape({ addTo: beeHead, stroke: 10, color: color.beeBlack, translate: { x: -22, y:  4, z: 36 } });
    new Zdog.Shape({ addTo: beeHead, stroke: 10, color: color.beeBlack, translate: { x:  14, y:  4, z: 40 } });

    let beeMouth = new Zdog.Shape({
      addTo: beeHead, stroke: 3.5, color: color.beeBlack, closed: false,
      path: [{ x: -4, y: 10, z: 42 }, { bezier: [{ x: -2, y: 13, z: 42 }, { x: 2, y: 13, z: 42 }, { x: 4, y: 10, z: 42 }] }]
    });

    let leftBrow = new Zdog.Shape({
      addTo: beeHead,
      path: [{ x: -34, y: -14, z: 44 }, { x: -10, y: -22, z: 44 }],
      stroke: 8, color: color.beeBlack,
      translate: { x: -18, y: -8, z: 20 }, rotate: { z: -0.18 }
    });
    let rightBrow = new Zdog.Shape({
      addTo: beeHead,
      path: [{ x: 10, y: -22, z: 44 }, { x: 34, y: -14, z: 44 }],
      stroke: 8, color: color.beeBlack,
      translate: { x: 18, y: -8, z: 20 }, rotate: { z: 0.18 }
    });

    new Zdog.Shape({ addTo: beeHead, stroke: 14, color: color.beeCheek, translate: { x: -32, y: 14, z: 24 } });
    new Zdog.Shape({ addTo: beeHead, stroke: 14, color: color.beeCheek, translate: { x:  24, y: 14, z: 30 } });

    let antlerAnchor = new Zdog.Anchor({ addTo: beeHead, translate: { y: -44, z: 10 } });
    new Zdog.Shape({ addTo: antlerAnchor, path: [{ y: 0, x: -10 }, { y: -22, x: -24, z: 8 }], stroke: 4.5, color: color.beeBlack });
    new Zdog.Shape({ addTo: antlerAnchor, path: [{ y: 0, x:  10 }, { y: -22, x:  -4, z: 12}], stroke: 4.5, color: color.beeBlack });

    let leftArm  = new Zdog.Anchor({ addTo: BEE, translate: { x: -18, y: 45, z: 32 } });
    let rightArm = new Zdog.Anchor({ addTo: BEE, translate: { x:  18, y: 45, z: 32 } });
    new Zdog.Shape({ addTo: leftArm,  path: [{ y: 0 }, { y: 22 }], stroke: 7, color: color.beeBlack });
    new Zdog.Shape({ addTo: rightArm, path: [{ y: 0 }, { y: 22 }], stroke: 7, color: color.beeBlack });

    let bodyAnchor = new Zdog.Anchor({ addTo: BEE, translate: { y: 32, z: -35 } });
    let p1 = new Zdog.Shape({ addTo: bodyAnchor, stroke: 140, color: color.beeYellow });
    let p2 = p1.copy({ addTo: bodyAnchor, stroke: 162, color: color.beeBlack,   translate: { z: -32  } });
    let p3 = p1.copy({ addTo: bodyAnchor, stroke: 168, color: color.beeYellow,  translate: { z: -64  } });
    let p4 = p1.copy({ addTo: bodyAnchor, stroke: 156, color: color.beeBlack,   translate: { z: -96  } });
    new Zdog.Shape({ addTo: bodyAnchor, stroke: 108, color: color.beeBlack, translate: { z: -123 } });

    let rightWing = new Zdog.Anchor({ addTo: bodyAnchor, translate: { z: -43, y: -65, x:  29 } });
    let leftWing  = new Zdog.Anchor({ addTo: bodyAnchor, translate: { z: -43, y: -65, x: -29 } });

    new Zdog.Ellipse({ addTo: rightWing, width: 80, height: 160, color: color.beeWhite, fill: true, rotate: { x: TAU/5, z:  TAU/5 }, translate: { x:  65 }, stroke: 0 });
    new Zdog.Ellipse({ addTo: leftWing,  width: 80, height: 160, color: color.beeWhite, fill: true, rotate: { x: TAU/5, z: -TAU/5 }, translate: { x: -65 }, stroke: 0 });

    function setExpression(name) {
      if (name === 'neutral') {
        leftBrow.rotate.z  = -0.18; rightBrow.rotate.z = 0.18;
        beeMouth.path = [{ x: -4, y: 10, z: 42 }, { bezier: [{ x: -2, y: 13, z: 42 }, { x: 2, y: 13, z: 42 }, { x: 4, y: 10, z: 42 }] }];
      } else if (name === 'annoyed') {
        leftBrow.rotate.z  = -0.55; rightBrow.rotate.z = 0.55;
        beeMouth.path = [{ x: -4, y: 14, z: 42 }, { bezier: [{ x: -2, y: 10, z: 42 }, { x: 2, y: 10, z: 42 }, { x: 4, y: 14, z: 42 }] }];
      } else if (name === 'angry') {
        leftBrow.rotate.z  = -0.8;  rightBrow.rotate.z = 0.8;
        beeMouth.path = [{ x: -5, y: 15, z: 42 }, { bezier: [{ x: -2, y: 8, z: 42 }, { x: 2, y: 8, z: 42 }, { x: 5, y: 15, z: 42 }] }];
      } else if (name === 'excited') {
        leftBrow.rotate.z  = -0.05; rightBrow.rotate.z = 0.05;
        beeMouth.path = [{ x: -6, y: 7, z: 42 }, { bezier: [{ x: -2, y: 18, z: 42 }, { x: 2, y: 18, z: 42 }, { x: 6, y: 7, z: 42 }] }];
      } else if (name === 'happy') {
        leftBrow.rotate.z  = -0.1;  rightBrow.rotate.z = 0.1;
        beeMouth.path = [{ x: -5, y: 8, z: 42 }, { bezier: [{ x: -2, y: 16, z: 42 }, { x: 2, y: 16, z: 42 }, { x: 5, y: 8, z: 42 }] }];
      }
      beeMouth.updatePath();
    }
    setExpression('neutral');

    let frame = 0, isRunning = true, isGathering = false, isAnnoyed = false, isShaking = false;

    function render() {
      if (!isRunning) return;
      frame++;
      const wingSpd = isAnnoyed ? 2.5 : 1.0;
      rightWing.rotate.z = -TAU / 6 + (TAU / 10) * Math.sin(frame / (0.9 / wingSpd));
      leftWing.rotate.z  =  TAU / 6 - (TAU / 10) * Math.sin(frame / (0.9 / wingSpd));
      antlerAnchor.rotate.x = (TAU / 40) * Math.sin(frame / 10);
      if (isShaking) BEE_PIVOT.rotate.z = Math.sin(frame / 3) * 0.18;
      if (!isGathering) {
        BEE_CONT.translate.y = -100 + Math.sin(frame / 14) * 5;
        BEE_CONT.translate.x = Math.cos(frame / 28) * 4;
      }
      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }
    render();

    function triggerAnnoyance() {
      if (isAnnoyed) return;
      isAnnoyed = true; isShaking = true;
      setExpression('angry');
      const tl = gsap.timeline({
        onComplete: () => {
          setExpression('neutral');
          isAnnoyed = false; isShaking = false;
          BEE_PIVOT.rotate.z = 0;
        }
      });
      tl.to([beeHead, p1, p3], { duration: 0.12, color: color.beeAngryRed, ease: 'power2.out' });
      tl.to([beeHead, p1, p3], { duration: 0.7,  color: color.beeYellow,   ease: 'power1.inOut', delay: 0.9 });
      askBee("Someone just pulled on my tail — I'm cross! Give me short playful bee wisdom about keeping cool.");
    }

    // Dynamic pollen gather routing based on target flower parameters
    function gatherPollen(target) {
      if (isGathering) return;
      isGathering = true;
      setExpression('happy');

      // Adjust relative hover positioning directly relative to flower location offsets
      const targetX = target.x;
      const targetY = target.y - 45;
      const targetZ = target.z + 40;

      const tl = gsap.timeline({ onComplete: () => { setExpression('neutral'); isGathering = false; } });
      
      tl.to(BEE_CONT.translate, { duration: 0.65, x: targetX, y: targetY, z: targetZ, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.45, x: 0.35, y: targetX > 0 ? 0.25 : -0.25, ease: 'power2.out' }, '-=0.45');
      tl.to(leftArm.rotate,     { duration: 0.2, z: 0.2, x: 0.1,      ease: 'back.out(1.5)'}, '-=0.15');
      tl.to(rightArm.rotate,    { duration: 0.2, z: -0.2, x: 0.1,     ease: 'back.out(1.5)'}, '<');
      
      // Gathering contact wiggles
      tl.to(BEE_CONT.translate, { duration: 0.12, y: targetY + 12, ease: 'power1.inOut' });
      tl.to(BEE_CONT.translate, { duration: 0.12, y: targetY - 4,  ease: 'power1.inOut' });
      tl.to(BEE_CONT.translate, { duration: 0.12, y: targetY + 6,  ease: 'power1.inOut' });
      
      tl.to(leftArm.rotate,     { duration: 0.2, z: 0, x: 0,          ease: 'power2.in' });
      tl.to(rightArm.rotate,    { duration: 0.2, z: 0, x: 0,          ease: 'power2.in' }, '<');
      
      // Return flight paths home
      tl.to(BEE_CONT.translate, { duration: 0.6, x: 0, y: -100, z: 0, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.45, x: 0, y: 0,          ease: 'power2.inOut' }, '-=0.45');
    }

    function talkingAnimation() {
      if (isGathering) return;
      isGathering = true;
      setExpression('happy');
      const tl = gsap.timeline({ onComplete: () => { setExpression('neutral'); isGathering = false; } });
      tl.to(leftArm.rotate,     { duration: 0.4, z: 0.3, x: 0.15, ease: 'sine.inOut' });
      tl.to(rightArm.rotate,    { duration: 0.4, z: -0.3, x: 0.15, ease: 'sine.inOut' }, '<');
      tl.to(leftArm.rotate,     { duration: 0.4, z: 0, x: 0, ease: 'sine.inOut' }, '-=0.1');
      tl.to(rightArm.rotate,    { duration: 0.4, z: 0, x: 0, ease: 'sine.inOut' }, '<');
      tl.to(leftArm.rotate,     { duration: 0.35, z: 0.25, x: 0.12, ease: 'sine.inOut' });
      tl.to(rightArm.rotate,    { duration: 0.35, z: -0.25, x: 0.12, ease: 'sine.inOut' }, '<');
      tl.to(leftArm.rotate,     { duration: 0.35, z: 0, x: 0, ease: 'sine.inOut' });
      tl.to(rightArm.rotate,    { duration: 0.35, z: 0, x: 0, ease: 'sine.inOut' }, '<');
    }

    triggerAnnoyanceFn = triggerAnnoyance;
    gatherPollenFn     = gatherPollen;
    talkingFn          = talkingAnimation;

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
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (dist > 10) { wasDragging = true; }
      if (dist > 28) triggerAnnoyance();
      BEE_PIVOT.rotate.y += dx * sensitivity;
      BEE_PIVOT.rotate.x += dy * sensitivity;
      BEE_PIVOT.rotate.x  = Math.max(-TAU / 8, Math.min(TAU / 8, BEE_PIVOT.rotate.x));
      scene.updateRenderGraph();
    }

    function onPointerUp() { isDragging = false; }

    function onCanvasClick(e) {
      if (wasDragging) return;
      const rect   = canvasRef.getBoundingClientRect();
      const cx     = (e.clientX - rect.left  - rect.width  / 2);
      const cy     = (e.clientY - rect.top   - rect.height / 2);

      // Flower projection click boundaries matching your back scene
      for (const fz of flowerZones) {
        if (Math.hypot(cx - fz.cx, cy - fz.cy) < fz.r) {
          gatherPollen(fz); return;
        }
      }

      // Bee body click → radial menu
      if (Math.hypot(cx, cy + 100) < 130) {
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

    setTimeout(() => showThought("Buzz! 🌸 I'm thinking about effort and rest. What about you?"), 3000);
    scheduleThought();

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

<div class="top-bar">
  <button class="bar-btn" on:click={goHome}>← Home</button>
  <button class="bar-btn" on:click={toggleHistory}>
    {historyOpen ? 'Close history' : '💬 History'}
  </button>
</div>

{#if historyOpen}
  <div class="history-panel">
    <div class="history-header">
      <h2>Session chat</h2>
      <button class="close-btn" on:click={() => historyOpen = false}>✕</button>
    </div>
    <div class="history-scroll">
      {#each chatHistory as msg}
        <div class="history-msg {msg.role}">
          <span class="history-label">{msg.role === 'assistant' ? '🐝 Buzz' : 'You'}</span>
          <p>{msg.content}</p>
        </div>
      {/each}
    </div>
  </div>
{/if}

{#if !embedded}
  <BeeBackground />
{/if}
<canvas bind:this={canvasRef} class="scene"></canvas>

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

{#if !embedded}
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
        placeholder="Reply to Buzz…"
        rows="1"
        disabled={isApiLoading}
        on:keydown={handleReplyKey}
      ></textarea>
      <button class="reply-send" on:click={sendReply} disabled={isApiLoading || !replyInputValue.trim()}>
        {isApiLoading ? '…' : '➤'}
      </button>
    </div>
  </div>
{/if}

{#if !embedded}
  <p class="hint">Tap the bee to open interactions ✿</p>
{/if}

<style>
  @import url("https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap");

  :global(body) {
    margin: 0; padding: 0;
    width: 100vw; height: 100vh;
    background: #FFB7D5;
    overflow: hidden;
    font-family: 'Nunito', sans-serif;
  }

  canvas.scene {
    position: fixed; inset: 0;
    width: 100vw; height: 100vh;
    display: block; z-index: 1;
    touch-action: none;
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
    color: #5A1A30;
    font-family: 'Nunito', sans-serif;
    font-weight: 700; font-size: 0.85rem;
    cursor: pointer;
    box-shadow: 0 4px 18px rgba(90,26,48,.10);
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
    box-shadow: 0 20px 60px rgba(90,26,48,.18);
    z-index: 25;
    display: flex; flex-direction: column;
    overflow: hidden;
  }
  .history-header {
    display: flex; justify-content: space-between; align-items: center;
    padding: 18px 20px 12px;
    border-bottom: 1px solid rgba(240,74,111,.12);
  }
  .history-header h2 {
    margin: 0; font-size: 1rem; color: #5A1A30;
  }
  .close-btn {
    border: none; background: #fde6ef;
    color: #B73058; border-radius: 50%;
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
  .history-msg p { margin: 4px 0 0; color: #3D1020; }
  .history-msg.assistant { background: #fff0f6; }
  .history-msg.user { background: #fffcf3; }
  .history-label { font-weight: 800; font-size: 0.78rem; color: #B73058; }

  /* ── Radial menu ─────────────────────────────────────────────────── */
  .radial-overlay {
    position: fixed; inset: 0;
    z-index: 20;
    pointer-events: none;
  }

  .radial-btn {
    position: fixed;
    width: 72px; height: 72px;
    border: none; border-radius: 50%;
    background: rgba(255,255,255,0.95);
    box-shadow: 0 6px 28px rgba(90,26,48,.16);
    cursor: pointer;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    gap: 2px;
    pointer-events: all;
    animation: radialPop .25s cubic-bezier(.34,1.56,.64,1) both;
    transition: transform .12s, background .12s;
  }
  .radial-btn:hover {
    background: #FF8FAE;
    transform: scale(1.12);
  }
  .radial-btn:hover .radial-label { color: #fff; }

  @keyframes radialPop {
    from { transform: scale(0); opacity: 0; }
    to   { transform: scale(1); opacity: 1; }
  }

  .radial-icon  { font-size: 1.5rem; line-height: 1; }
  .radial-label { font-size: 0.65rem; font-weight: 800; color: #7A2D44; letter-spacing: .02em; }

  .radial-center {
    position: fixed;
    left:  calc(50% - 24px);
    top:   calc(50% - 24px);
    width: 48px; height: 48px;
    border: none; border-radius: 50%;
    background: rgba(255,143,174,.9);
    color: #fff;
    font-size: 1rem; font-weight: 800;
    cursor: pointer;
    box-shadow: 0 4px 20px rgba(90,26,48,.18);
    pointer-events: all;
    animation: radialPop .2s cubic-bezier(.34,1.56,.64,1) both;
    transition: background .12s, transform .1s;
    z-index: 21;
  }
  .radial-center:hover { background: #F04A6F; transform: scale(1.08); }

  /* ── Interaction card ────────────────────────────────────────────── */
  .card-overlay {
    position: fixed; inset: 0;
    background: rgba(90, 26, 48, 0.28);
    backdrop-filter: blur(4px);
    z-index: 30;
    display: flex; align-items: center; justify-content: center;
  }
  .card {
    background: #fff;
    border-radius: 28px;
    padding: 32px 28px;
    max-width: 340px; width: calc(100vw - 48px);
    box-shadow: 0 28px 80px rgba(90,26,48,.22);
    text-align: center;
    animation: cardIn .22s cubic-bezier(.34,1.56,.64,1) both;
  }
  @keyframes cardIn {
    from { transform: scale(.88); opacity: 0; }
    to   { transform: scale(1);   opacity: 1; }
  }
  .card-icon { font-size: 2.8rem; margin-bottom: 8px; }
  .card h2   { margin: 0 0 10px; font-size: 1.1rem; color: #5A1A30; }
  .card-desc { margin: 0 0 8px; color: #7a3050; line-height: 1.6; font-size: 0.95rem; }
  .card-hint { margin: 0 0 24px; color: #b07090; font-size: 0.85rem; font-style: italic; }
  .card-actions { display: flex; gap: 10px; }
  .card-btn {
    flex: 1; border: none; border-radius: 16px;
    padding: 13px 0; font-family: 'Nunito', sans-serif;
    font-weight: 800; font-size: 0.9rem; cursor: pointer;
    transition: background .12s, transform .1s;
  }
  .card-btn:active { transform: scale(.96); }
  .card-btn.primary { background: #FF8FAE; color: #fff; }
  .card-btn.primary:hover { background: #F04A6F; }
  .card-btn.ghost   { background: #fde6ef; color: #B73058; }
  .card-btn.ghost:hover { background: #f9cad8; }

  /* ── Thought bubble ──────────────────────────────────────────────── */
  .thought-wrap {
    position: fixed;
    top: 48%;
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
    border: 2px solid rgba(240,74,111,.25);
    border-radius: 22px 22px 22px 6px;
    padding: 14px 18px;
    box-shadow: 0 8px 36px rgba(90,26,48,.12);
    position: relative;
  }
  .thought-dots {
    display: flex; gap: 4px; margin-bottom: 6px;
  }
  .thought-dots span {
    width: 5px; height: 5px;
    background: #FF8FAE; border-radius: 50%;
  }
  .thought-text {
    margin: 0;
    font-size: 0.92rem;
    color: #4B1528;
    line-height: 1.6;
  }

  .thought-reply {
    display: flex; gap: 6px;
    background: #fff; padding: 6px;
    border-radius: 20px;
    border: 1.5px solid rgba(240,74,111,.15);
    box-shadow: 0 6px 20px rgba(90,26,48,.06);
  }
  .thought-reply textarea {
    flex: 1; border: none; resize: none;
    font-family: 'Nunito', sans-serif; font-size: 0.88rem;
    padding: 6px 10px; color: #3D1020; outline: none;
    background: transparent;
  }
  .reply-send {
    border: none; background: #FF8FAE; color: #fff;
    border-radius: 50%; width: 32px; height: 32px;
    cursor: pointer; font-size: 0.85rem;
    display: flex; align-items: center; justify-content: center;
    transition: background .12s;
  }
  .reply-send:hover:not(:disabled) { background: #F04A6F; }
  .reply-send:disabled { background: #f0c3d0; cursor: not-allowed; }

  .hint {
    position: fixed; bottom: 20px; left: 24px;
    margin: 0; font-size: 0.85rem; font-weight: 700;
    color: #A05070; z-index: 10;
    pointer-events: none;
    background: rgba(255,255,255,0.4);
    padding: 4px 12px; border-radius: 12px;
  }
</style>