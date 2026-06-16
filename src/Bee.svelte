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

  let historyOpen = false;
  let thoughtBubbleVisible = false;
  let thoughtBubbleText = '';
  let replyInputValue = '';
  let thoughtTimer = null;
  let bubbleAutoHideTimer = null;
  let isApiLoading = false;
  let conversationActive = false;

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

  function toggleHistory() {
    historyOpen = !historyOpen;
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

      // Bee body click → invite chat instead of opening an interaction toolkit
      if (Math.hypot(cx, cy + 100) < 130) {
        showThought('Buzz! 🌸 What is on your mind?', true);
      }
    }
    
    if (!embedded) {
      canvasRef.addEventListener('click',       onCanvasClick);
      canvasRef.addEventListener('pointerdown', onPointerDown);
      canvasRef.addEventListener('pointermove', onPointerMove);
      window.addEventListener('pointerup',      onPointerUp);
    }

    if (!embedded) {
      setTimeout(() => showThought("Buzz! 🌸 I'm thinking about effort and rest. What about you?"), 3000);
      scheduleThought();
    }

    return () => {
      isRunning = false;
      if (thoughtTimer) clearTimeout(thoughtTimer);
      if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
      if (!embedded) document.body.classList.remove(pageBodyClass);
      canvasRef?.removeEventListener('click',       onCanvasClick);
      canvasRef?.removeEventListener('pointerdown', onPointerDown);
      canvasRef?.removeEventListener('pointermove', onPointerMove);
      window.removeEventListener('pointerup',       onPointerUp);
    };
  });
</script>

{#if !embedded}
  <div class="top-bar">
    <button class="bar-btn" on:click={goHome}>←</button>
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
<canvas bind:this={canvasRef} class="scene" class:embedded></canvas>


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
  <p class="hint">Click flowers to gather pollen. Drag quickly to annoy.</p>
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

  /* ── Thought bubble ──────────────────────────────────────────────── */
  .thought-wrap {
    position: fixed;
    bottom: 32px;
    right: 32px;
    max-width: min(360px, 90vw);
    display: flex; flex-direction: column; gap: 10px;
    z-index: 15;
    pointer-events: none;
    opacity: 0;
    transform: translateY(8px) scale(.96);
    transition: opacity .35s ease, transform .35s ease;
    align-items: flex-end;
  }
  .thought-wrap.visible {
    opacity: 1;
    transform: translateY(0) scale(1);
    pointer-events: all;
  }

  .thought-bubble {
    background: #fff;
    border: 2px solid rgba(240,74,111,.25);
    border-radius: 22px 22px 6px 22px;
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
    display: flex; gap: 8px;
    align-items: flex-end;
    width: 100%;
  }
  .thought-reply textarea {
    flex: 1;
    background: rgba(255,255,255,0.96);
    border: 1.5px solid rgba(240,74,111,.25);
    border-radius: 14px;
    padding: 10px 14px;
    font-family: 'Nunito', sans-serif;
    font-size: 0.9rem;
    color: #3D1020;
    outline: none; resize: none;
    height: 44px; min-height: 44px; max-height: 88px;
    line-height: 1.5; box-sizing: border-box;
    transition: border-color .15s, box-shadow .15s;
  }
  .thought-reply textarea::placeholder { color: #C08AA0; }
  .thought-reply textarea:focus {
    border-color: #FF8FAE;
    box-shadow: 0 0 0 3px rgba(255,143,174,.18);
  }
  .reply-send {
    border: none; background: #FF8FAE; color: #fff;
    border-radius: 50%; width: 44px; height: 44px;
    cursor: pointer; font-size: 0.85rem;
    display: flex; align-items: center; justify-content: center;
    transition: background .12s;
  }
  .reply-send:hover:not(:disabled) { background: #F04A6F; }
  .reply-send:disabled { background: #f0c3d0; cursor: not-allowed; }

  .hint {
    position: fixed;
    bottom: 22px; left: 50%;
    transform: translateX(-50%);
    margin: 0;
    font-size: 0.8rem; font-weight: 600;
    color: rgba(90, 26, 48, 0.58);
    pointer-events: none;
    letter-spacing: .04em;
    z-index: 10;
    text-align: center;
  }
</style>