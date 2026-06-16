<script>
  //@ts-nocheck
  import { onMount, createEventDispatcher } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';
  import DoveBackground from './DoveBackground.svelte';

  const dispatch = createEventDispatcher();
  let canvasRef;

  export let embedded = false;

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
    { role: 'assistant', content: "Coo… peace to you. 🕊️ Ask me about calm, letting go, or finding hope." }
  ];

  const SYSTEM_TEXT = `You are Olive, a gentle, serene dove who carries a quiet sense of peace. You speak softly and warmly in short, calming messages (2-4 sentences max). You use sky, wind, feather, and olive-branch metaphors naturally.
Your specialty is helping people think about:
- Finding calm in the middle of conflict or worry
- Letting go of grudges and old resentments
- Holding onto hope after something hard
- Being gentle with yourself and with others
You give soft, reassuring, hopeful advice. You never preach. Keep it warm, short, and peaceful.`;

  const thoughtPrompts = [
    'The sky is wide and quiet today. What is weighing on you?',
    'A grudge is heavy to carry on small wings. Is there one you could set down?',
    'Storms pass. They always do. What are you waiting out right now?',
    'Peace is not the absence of noise — it is a soft place inside it.',
    'I carry an olive branch for a reason. Who might need yours?',
    'When did you last let yourself simply drift on the wind?'
  ];

  let startleFn;
  let flyFn;
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
      }, 13000);
    }
  }

  function scheduleThought() {
    if (thoughtTimer) clearTimeout(thoughtTimer);
    thoughtTimer = setTimeout(() => {
      const prompt = thoughtPrompts[Math.floor(Math.random() * thoughtPrompts.length)];
      showThought(prompt);
      scheduleThought();
    }, 18000 + Math.random() * 13000);
  }

  async function sendReply() {
    const msg = replyInputValue.trim();
    if (!msg || isApiLoading) return;
    replyInputValue = '';
    thoughtBubbleVisible = false;
    await askOlive(msg);
  }

  function handleReplyKey(e) {
    if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); sendReply(); }
  }

  async function askOlive(userMessage) {
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
        throw new Error(`API returned status ${res.status}`);
      }

      const data = await res.json();
      const replyText = data?.choices?.[0]?.message?.content;

      if (replyText) {
        const cleanReply = replyText.trim();
        chatHistory = [...chatHistory, { role: 'assistant', content: cleanReply }];
        showThought(cleanReply, true);
      } else {
        showThought("Coo… the wind carried my thought away. Ask me again?", true);
      }
    } catch (e) {
      chatHistory = chatHistory.slice(0, -1);
      showThought("Coo… the wind carried my thought away. Ask me again?", true);
    } finally {
      isApiLoading = false;
    }
  }

  onMount(() => {
    if (!canvasRef) return;
    const TAU = Zdog.TAU;

    const C = {
      white:    '#ffffff',
      mid:      '#f2f2f2',
      shade:    '#DAD6CE',
      beak:     '#D89A6E',
      beakDk:   '#C07E50',
      cheek:    '#F0A8A0',
      eye:      '#1A1410'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.05, y: 0.28, z: 0 },
      zoom: 1.3
    });

    const fg = new Zdog.Anchor({ addTo: scene });
    const DOVE_CONT = new Zdog.Anchor({ addTo: fg, scale: 1.35 });
    const PIVOT = new Zdog.Anchor({ addTo: DOVE_CONT });
    const DOVE  = new Zdog.Anchor({ addTo: PIVOT });

    // ─── 1. SMOOTH INTEGRATED BODY & NECK ──────────────────────────────
    new Zdog.Shape({ 
      addTo: DOVE, 
      path: [{ x: -10, y: 15 }, { x: 25, y: 10 }], 
      stroke: 52, 
      color: C.white 
    });

    new Zdog.Shape({ 
      addTo: DOVE, 
      path: [{ x: -5, y: 15 }, { x: -35, y: -8 }], 
      stroke: 36, 
      color: C.white 
    });

    // ─── 2. HEAD & FACE ────────────────────────────────────────────────
    const head = new Zdog.Anchor({ addTo: DOVE, translate: { x: -35, y: -10, z: 0 } });
    new Zdog.Shape({ addTo: head, stroke: 35, color: C.white });

    new Zdog.Shape({ addTo: head, stroke: 6, color: C.eye, translate: { x: -6, y: -4, z: 15 } });
    new Zdog.Shape({ addTo: head, stroke: 1.8, color: '#fff', translate: { x: -7.5, y: -5, z: 16 } });
    new Zdog.Shape({ addTo: head, stroke: 6, color: C.eye, translate: { x: -6, y: -4, z: -15 } });
    
    new Zdog.Shape({ addTo: head, stroke: 8, color: C.cheek, translate: { x: -1, y: 4, z: 15 } });
    new Zdog.Shape({ addTo: head, stroke: 8, color: C.cheek, translate: { x: -1, y: 4, z: -15 } });

    // ─── 3. GLUED ON SHARP BEAK ────────────────────────────────────────
    const beak = new Zdog.Anchor({ addTo: head, translate: { x: -14, y: 2, z: 0 } });
    
    new Zdog.Shape({ 
      addTo: beak, 
      path: [{ x: 0, y: -4 }, { x: -18, y: 0 }, { x: 0, y: 4 }], 
      stroke: 2, 
      color: C.beak, 
      fill: true 
    });
    
    const lowerBeak = new Zdog.Anchor({ addTo: beak, translate: { x: 0, y: 2 } });
    new Zdog.Shape({ 
      addTo: lowerBeak, 
      path: [{ x: 0, y: 0 }, { x: -14, y: 0 }, { x: 0, y: 2 }], 
      stroke: 1.5, 
      color: C.beakDk, 
      fill: true 
    });

    // ─── 4. TWO VOLUMETRIC WINGS ───────────────────────────────────────
    const elegantWingPath = [
      { x: -5, y: 0 },
      { bezier: [{ x: 10, y: -45 }, { x: 45, y: -50 }, { x: 65, y: -15 }] },
      { bezier: [{ x: 50, y: 5 }, { x: 20, y: 15 }, { x: -5, y: 0 }] }
    ];

    const frontWing = new Zdog.Anchor({ addTo: DOVE, translate: { x: -12, y: 10, z: 24 } });
    new Zdog.Shape({ 
      addTo: frontWing, 
      path: elegantWingPath, 
      stroke: 20, 
      color: C.mid, 
      fill: true 
    });
    
    const backWing = new Zdog.Anchor({ addTo: DOVE, translate: { x: -12, y: 10, z: -24 } });
    new Zdog.Shape({ 
      addTo: backWing, 
      path: elegantWingPath, 
      stroke: 20, 
      color: C.mid, 
      fill: true 
    });

    // ─── 5. TAIL FAN ───────────────────────────────────────────────────
    const tail = new Zdog.Anchor({ addTo: DOVE, translate: { x: 22, y: 10 } });
    new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 38, y: 4 }], stroke: 16, color: C.shade });
    new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 32, y: 2 }], stroke: 12, color: C.mid, translate: { z: 10 } });
    new Zdog.Shape({ addTo: tail, path: [{ x: 0, y: 0 }, { x: 32, y: 2 }], stroke: 12, color: C.shade, translate: { z: -10 } });

    // ─── Expression system ───────────────────────────
    function setExpression(name) {
      let beakOpen = 0;
      if (name === 'calm')         { beakOpen = 0; }
      else if (name === 'happy')   { beakOpen = 0.3; }
      else if (name === 'alarmed') { beakOpen = 0.4; }
      lowerBeak.rotate.z = beakOpen;
    }
    setExpression('calm');

    // ─── Animation Loop ───────────────────────────────────────────────
    let frame = 0, isRunning = true, isBusy = false, wingBeat = 0;
    const flap = { amp: 0.06, speed: 1 };

    function render() {
      if (!isRunning) return;
      frame++;

      wingBeat += 0.13 * flap.speed;
      frontWing.rotate.z = Math.sin(wingBeat) * flap.amp;
      backWing.rotate.z  = Math.sin(wingBeat - 0.5) * flap.amp;

      if (!isBusy) {
        DOVE_CONT.translate.y = Math.sin(frame / 40) * 4;
        head.rotate.z = Math.sin(frame / 46) * 0.05;
        tail.rotate.z = Math.sin(frame / 60) * 0.04;
      }

      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }
    render();

    // ─── Interactions ──────────────────────────────────────────────────
    function startle() {
      if (isBusy) return;
      isBusy = true;
      setExpression('alarmed');
      gsap.to(flap, { duration: 0.1, amp: 0.4, speed: 3 });
      const tl = gsap.timeline({
        onComplete: () => {
          setExpression('calm');
          gsap.to(flap, { duration: 0.7, amp: 0.06, speed: 1 });
          isBusy = false; DOVE_CONT.translate.y = 0; PIVOT.rotate.z = 0;
        }
      });
      tl.to(DOVE_CONT.translate, { duration: 0.12, y: -28, ease: 'power2.out' });
      tl.to(PIVOT.rotate,        { duration: 0.12, z: 0.14, ease: 'power2.out' }, '<');
      tl.to(DOVE_CONT.translate, { duration: 0.6, y: 0, ease: 'bounce.out' });
      tl.to(PIVOT.rotate,        { duration: 0.6, z: 0, ease: 'power1.out' }, '<');
      tl.to({}, { duration: 0.6 });
      askOlive("Someone just startled me into a flurry of feathers. Share short, gentle dove wisdom about finding calm again after a fright.");
    }

    function fly() {
      if (isBusy) return;
      isBusy = true;
      setExpression('happy');
      gsap.to(flap, { duration: 0.2, amp: 0.65, speed: 3 });
      const tl = gsap.timeline({
        onComplete: () => {
          setExpression('calm');
          gsap.to(flap, { duration: 0.7, amp: 0.06, speed: 1 });
          isBusy = false;
          DOVE_CONT.translate.x = 0; DOVE_CONT.translate.y = 0; DOVE_CONT.translate.z = 0;
          PIVOT.rotate.z = 0; PIVOT.rotate.y = 0; PIVOT.rotate.x = 0;
        }
      });
      tl.to(DOVE_CONT.translate, { duration: 0.6, x: -120, y: -90, z: 60, ease: 'power2.inOut' });
      tl.to(PIVOT.rotate,        { duration: 0.6, z: 0.3, x: 0.1, ease: 'power2.inOut' }, '<');
      tl.to(DOVE_CONT.translate, { duration: 0.6, x: 0, y: -150, z: 120, ease: 'power2.inOut' });
      tl.to(PIVOT.rotate,        { duration: 0.6, z: 0, y: -0.4, ease: 'power2.inOut' }, '<');
      tl.to(DOVE_CONT.translate, { duration: 0.6, x: 130, y: -90, z: 60, ease: 'power2.inOut' });
      tl.to(PIVOT.rotate,        { duration: 0.6, z: -0.3, y: 0, ease: 'power2.inOut' }, '<');
      tl.to(DOVE_CONT.translate, { duration: 0.6, x: 0, y: 0, z: 0, ease: 'power2.inOut' });
      tl.to(PIVOT.rotate,        { duration: 0.6, z: 0, x: 0, ease: 'power2.inOut' }, '<');
    }

    function talkingAnimation() {
      if (isBusy) return;
      isBusy = true;
      const tl = gsap.timeline({ onComplete: () => { setExpression('calm'); isBusy = false; head.rotate.z = 0; } });
      for (let i = 0; i < 3; i++) {
        tl.add(() => setExpression('happy'));
        tl.to(head.rotate, { duration: 0.25, z: 0.08, ease: 'sine.inOut' });
        tl.add(() => setExpression('calm'));
        tl.to(head.rotate, { duration: 0.25, z: 0, ease: 'sine.inOut' });
      }
    }

    startleFn  = startle;
    flyFn      = fly;
    talkingFn  = talkingAnimation;

    // ─── Click / Double Click Handlers ──────────────────────────────────────────
    let clickTimeout = null;

    function onCanvasClick(e) {
      if (clickTimeout !== null) {
        // Double click detected
        clearTimeout(clickTimeout);
        clickTimeout = null;
        startleFn();
      } else {
        // Single click detected, wait to see if a second click comes
        clickTimeout = setTimeout(() => {
          clickTimeout = null;
          flyFn();
        }, 250); // 250ms window for double click
      }
    }

    if (!embedded) {
      canvasRef.addEventListener('click', onCanvasClick);
    }

    if (!embedded) {
      setTimeout(() => showThought("Coo… the sky is calm today. 🕊️ What is on your mind?"), 3500);
      scheduleThought();
    }

    return () => {
      isRunning = false;
      if (thoughtTimer) clearTimeout(thoughtTimer);
      if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
      if (clickTimeout) clearTimeout(clickTimeout);
      if (!embedded) {
        canvasRef?.removeEventListener('click', onCanvasClick);
      }
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
          <span class="history-label">{msg.role === 'assistant' ? '🕊️ Olive' : 'You'}</span>
          <p>{msg.content}</p>
        </div>
      {/each}
    </div>
  </div>
{/if}

{#if !embedded}
  <DoveBackground />
{/if}
<canvas bind:this={canvasRef} class="scene" class:embedded></canvas>

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
      placeholder="Reply to Olive…"
      rows="1"
      disabled={isApiLoading}
      on:keydown={handleReplyKey}
    ></textarea>
    <button class="reply-send" on:click={sendReply} disabled={isApiLoading || !replyInputValue.trim()}>
      {isApiLoading ? '…' : '➤'}
    </button>
  </div>
</div>

<p class="hint">Click to fly. Double-click to startle.</p>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap");

  :global(body) {
    margin: 0; padding: 0;
    width: 100vw; height: 100vh;
    background: #BFD9EF;
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
    color: #2E3A47;
    font-family: 'Nunito', sans-serif;
    font-weight: 700; font-size: 0.85rem;
    cursor: pointer;
    box-shadow: 0 4px 18px rgba(46,58,71,.12);
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
    box-shadow: 0 20px 60px rgba(46,58,71,.18);
    z-index: 25;
    display: flex; flex-direction: column;
    overflow: hidden;
  }
  .history-header {
    display: flex; justify-content: space-between; align-items: center;
    padding: 18px 20px 12px;
    border-bottom: 1px solid rgba(111,168,220,.18);
  }
  .history-header h2 { margin: 0; font-size: 1rem; color: #2E3A47; }
  .close-btn {
    border: none; background: #e4eef8;
    color: #3A6699; border-radius: 50%;
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
  .history-msg p { margin: 4px 0 0; color: #26313b; }
  .history-msg.assistant { background: #eef4fb; }
  .history-msg.user { background: #f7f4ef; }
  .history-label { font-weight: 800; font-size: 0.78rem; color: #4A7BB0; }

  /* ── Thought bubble (Moved to bottom right) ──────────────────────────────────────────────── */
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
    align-items: flex-end; /* Align elements to the right */
  }
  .thought-wrap.visible {
    opacity: 1;
    transform: translateY(0) scale(1);
    pointer-events: all;
  }
  .thought-bubble {
    background: #fff;
    border: 2px solid rgba(111,168,220,.30);
    border-radius: 22px 22px 6px 22px; /* Adjusted tail for right side */
    padding: 14px 18px;
    box-shadow: 0 8px 36px rgba(46,58,71,.14);
    position: relative;
    max-width: 100%;
  }
  .thought-dots { display: flex; gap: 4px; margin-bottom: 6px; }
  .thought-dots span { width: 5px; height: 5px; background: #6FA8DC; border-radius: 50%; }
  .thought-text {
    margin: 0;
    font-size: 0.92rem;
    color: #26313b;
    line-height: 1.6;
    font-weight: 600;
  }
  .thought-reply { display: flex; gap: 8px; align-items: flex-end; width: 100%; }
  .thought-reply textarea {
    flex: 1;
    background: rgba(255,255,255,0.96);
    border: 1.5px solid rgba(111,168,220,.40);
    border-radius: 14px;
    padding: 10px 14px;
    font-family: 'Nunito', sans-serif;
    font-size: 0.9rem;
    color: #26313b;
    outline: none; resize: none;
    height: 44px; min-height: 44px; max-height: 88px;
    line-height: 1.5; box-sizing: border-box;
    transition: border-color .15s, box-shadow .15s;
  }
  .thought-reply textarea::placeholder { color: #9bb6cf; }
  .thought-reply textarea:focus {
    border-color: #4F8BC9;
    box-shadow: 0 0 0 3px rgba(79,139,201,.18);
  }
  .reply-send {
    width: 44px; height: 44px;
    border-radius: 50%; border: none;
    background: #6FA8DC; color: #fff;
    font-size: 1rem; cursor: pointer;
    flex-shrink: 0;
    transition: background .12s, transform .1s;
    display: flex; align-items: center; justify-content: center;
  }
  .reply-send:hover:not(:disabled) { background: #4F8BC9; }
  .reply-send:active:not(:disabled) { transform: scale(.94); }
  .reply-send:disabled { opacity: .5; cursor: default; }

  /* ── Hint ────────────────────────────────────────────────────────── */
  .hint {
    position: fixed;
    bottom: 22px; left: 50%;
    transform: translateX(-50%);
    font-size: 0.8rem; font-weight: 600;
    color: rgba(46, 58, 71, 0.55);
    pointer-events: none;
    letter-spacing: .04em;
    z-index: 10;
  }
  .credit {
    position: fixed; bottom: 8px; left: 14px;
    font-size: 0.7rem; color: rgba(46,58,71,.42);
    z-index: 10; pointer-events: none;
  }
</style>