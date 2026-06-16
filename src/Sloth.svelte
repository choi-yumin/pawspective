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

  let historyOpen = false;
  let thoughtBubbleVisible = false;
  let thoughtBubbleText = '';
  let replyInputValue = '';
  let thoughtTimer = null;
  let bubbleAutoHideTimer = null;
  let isApiLoading = false;
  let conversationActive = false;

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
      claw:       '#4A3826'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.10, y: -0.08, z: -0.15 }
    });

    const SLOTH_CONT = new Zdog.Anchor({ addTo: scene, translate: { x: -30, y: 200, z: -500 }, scale: 1 });
    const PIVOT = new Zdog.Anchor({ addTo: SLOTH_CONT });
    const SLOTH = new Zdog.Anchor({ addTo: PIVOT });

    // Body
    new Zdog.Shape({
      addTo: SLOTH,
      path: [
        { x: -75, y: -4, z: 0 },
        { bezier: [{ x: -30, y: 25, z: 0 }, { x: 30, y: 25, z: 0 }, { x: 75, y: -4, z: 0 }] }
      ],
      stroke: 120, color: color.fur, closed: false
    });

    // Belly
    new Zdog.Shape({
      addTo: SLOTH,
      path: [
        { x: -40, y: -4, z: 0 },
        { bezier: [{ x: -15, y: 8, z: 0 }, { x: 15, y: 8, z: 0 }, { x: 40, y: -4, z: 0 }] }
      ],
      stroke: 40, color: color.furLight, translate: { y: -28, z: 0 }, closed: false
    });

    // Limbs
    function makeLimb(baseX, baseY, baseZ, gripX, stroke) {
      const limb = new Zdog.Anchor({ addTo: SLOTH, translate: { x: baseX, y: baseY, z: baseZ } });
      const dx = gripX - baseX;
      const dy = -210 - baseY; 

      new Zdog.Shape({
        addTo: limb,
        path: [
          { x: 0, y: 0, z: 0 },
          { bezier: [{ x: dx * 0.2, y: dy * 0.4, z: 0 }, { x: dx * 0.7, y: dy * 0.8, z: 0 }, { x: dx, y: dy, z: 0 }] }
        ],
        stroke, color: color.fur, fill: false
      });

      const hand = new Zdog.Anchor({ addTo: limb, translate: { x: dx, y: dy, z: 0 } });
      for (let i = 0; i < 3; i++) {
        new Zdog.Shape({
          addTo: hand,
          path: [
            { x: (i - 1) * 6, y: 0 },
            { bezier: [{ x: (i - 1) * 6, y: -6 }, { x: (i - 1) * 6 - 4, y: -12 }, { x: (i - 1) * 6 - 5, y: -16 }] }
          ],
          stroke: 5, color: color.claw, closed: false
        });
      }
      return limb;
    }

    const leftArm  = makeLimb(-60, -10,  46, -110, 34);  
    const rightArm = makeLimb(-80, -10, -46, -170, 34); 
    const leftLeg  = makeLimb(60, -12,  46,  80, 34);  
    const rightLeg = makeLimb(100, -12, -46,  140, 34);

    // Head
    const head = new Zdog.Anchor({ addTo: SLOTH, translate: { x: -140, y: 12, z: 14 }, scale: { y: -1 } });
    new Zdog.Shape({ addTo: head, stroke: 92, color: color.fur });
    new Zdog.Shape({ addTo: head, stroke: 46, color: color.faceCream, translate: { x: -30, y: 10, z: 14 } });
    new Zdog.Shape({ addTo: head, stroke: 16, color: color.nose,      translate: { x: -54, y:  8, z: 14 } });

  const eye = new Zdog.Ellipse({ 
  addTo: head, width: 14, height: 14, 
  fill: true, stroke: 1, color: color.eye, 
  translate: { x: -16, y: -2, z: 42 } 
  });

  const lid = new Zdog.Ellipse({ 
    addTo: eye, width: 14.5, height: 14.5, 
    fill: true, stroke: 1, color: color.mask, 
    translate: { z: 2 }, // Pulled to the absolute front
    quarters: 4 
  });

  const pupil = new Zdog.Ellipse({ 
    addTo: eye, width: 4.5, height: 4.5,
    fill: true, stroke: 1, color: '#FFFFFF', 
    // Changed y to 2.5 to move it to the visual bottom half
    translate: { x: -0.5, y: 0.5, z: 1 } 
});

    const leftBrow  = new Zdog.Shape({ addTo: head, path: [{ x: -30, y: -18 }, { x: -6, y: -22 }], stroke: 6, color: color.furDark, translate: { z: 44 } });
    const rightBrow = new Zdog.Shape({ addTo: head, path: [{ x: 20, y: -16 }, { x: 36, y: -18 }],   stroke: 6, color: color.furDark, translate: { z: -30 } });

    const mouth = new Zdog.Shape({
      addTo: head, stroke: 3.5, color: color.nose, closed: false,
      path: [ { x: -46, y: 20, z: 20 }, { bezier: [{ x: -42, y: 23, z: 20 }, { x: -36, y: 23, z: 20 }, { x: -30, y: 20, z: 20 }] } ]
    });

    // ─── Expression system ───
    let lidState = 4; // 4 = closed, 2 = half open

    function setExpression(name) {
      let brow = -0.1, my = 20, c0y = 23, c1y = 23;
      
      if (name === 'sleepy')        { lidState = 4; brow = -0.10; my = 20; c0y = 23; c1y = 23; } 
      else if (name === 'content')  { lidState = 2; brow = -0.05; my = 19; c0y = 24; c1y = 24; } 
      else if (name === 'surprised'){ lidState = 0; brow = -0.22; my = 20; c0y = 27; c1y = 27; } 
      
      if (lidState === 4) {
        lid.quarters = 4;
        lid.rotate.z = 0;
        lid.visible = true;
        pupil.visible = false; // FORCE HIDING THE PUPIL
      } else if (lidState === 2) {
        lid.quarters = 2;
        lid.rotate.z = -Zdog.TAU / 4; // PUTS THE LID ON THE TOP HALF
        lid.visible = true;
        pupil.visible = true;
      } else {
        lid.visible = false;
        pupil.visible = true;
      }
      lid.updatePath();

      leftBrow.rotate.z  = brow;
      rightBrow.rotate.z = -brow;
      mouth.path = [
        { x: -46, y: my, z: 20 },
        { bezier: [{ x: -42, y: c0y, z: 20 }, { x: -36, y: c1y, z: 20 }, { x: -30, y: my, z: 20 }] }
      ];
      mouth.updatePath();
    }
    setExpression('sleepy');

    // ─── Animation Loop ───
    // ─── Animation Loop ───
    let frame = 0, isRunning = true, isBusy = false, blinkTimer = 0;

    function render() {
      if (!isRunning) return;
      frame++;

      scene.rotate.y = -0.06 + Math.sin(frame / 1200) * 0.04;
      scene.rotate.x = -0.10 + Math.sin(frame / 1600) * 0.015;

      if (!isBusy) {
        SLOTH_CONT.translate.y = 40 + Math.sin(frame / 100) * 0.5;
      }

      // --- NEW STEP 3 GOES HERE ---
     // If awake, add occasional blinks
      // If awake, add occasional blinks
      if (lidState < 4) {
        blinkTimer++;
        if (blinkTimer > 240) {
          const t = blinkTimer - 240;
          if (t < 6) {
            // Blink shut
            if (lid.quarters !== 4) {
               lid.quarters = 4;
               lid.rotate.z = 0;
               lid.visible = true;
               pupil.visible = false; // HIDE PUPIL DURING BLINK
               lid.updatePath();
            }
          } else {
            // Snap back open
            if (lid.quarters !== lidState) {
               lid.quarters = lidState;
               lid.rotate.z = lidState === 2 ? -Zdog.TAU / 4 : 0;
               lid.visible = (lidState > 0);
               pupil.visible = true; // SHOW PUPIL AFTER BLINK
               lid.updatePath();
            }
            if (t >= 12) blinkTimer = 0;
          }
        }
      } else {
         // Failsafe for when it's fully asleep
         pupil.visible = false;
      }
      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }
    render();

    // ─── Interactions ───
    function triggerWake() {
      if (isBusy) return;
      isBusy = true;
      setExpression('surprised');
      
      const tl = gsap.timeline({
        onComplete: () => {
          // Stay awake for a little while, then gently go back to sleep
          setTimeout(() => {
            isBusy = false;
            // Only fall asleep if the user isn't currently dragging it
            if (!isDragging) {
              setExpression('sleepy');
            }
          }, 5000);
        }
      });
      // A gentle surprised startle
      tl.to(PIVOT.translate, { y: -8, duration: 0.15, yoyo: true, repeat: 1, ease: "sine.inOut" });
      
      askMossy("Oh... someone just woke me up from my nap. Say a short, sleepy greeting.");
    }

    function talkingAnimation() {
      if (isBusy) return;
      const tl = gsap.timeline();
      tl.to(head.rotate,     { duration: 0.7, z: 0.06, ease: 'sine.inOut' });
      tl.to(head.rotate,     { duration: 0.7, z: -0.05, ease: 'sine.inOut' });
      tl.to(head.rotate,     { duration: 0.7, z: 0, ease: 'sine.inOut' });
    }
    talkingFn = talkingAnimation;

    // ─── Drag to Climb & Double Click to Wake ─────────────────────────
    let isDragging = false, lastX = 0, lastY = 0, wasDragging = false, dragDist = 0;
    let clickTimeout = null;

    function onPointerDown(e) {
      isDragging = true; wasDragging = false; dragDist = 0;
      lastX = e.clientX; lastY = e.clientY;
      try { canvasRef.setPointerCapture(e.pointerId); } catch (_) {}
      
      // Open eyes upon touch
      if (!isBusy) setExpression('content');
    }
    
    function onPointerMove(e) {
      if (!isDragging) return;
      const dx = e.clientX - lastX, dy = e.clientY - lastY;
      lastX = e.clientX; lastY = e.clientY;
      const dist = Math.hypot(dx, dy);
      dragDist += dist;
      
      if (dragDist > 10) {
        wasDragging = true;
        
        // Calculate the proposed new X position based on drag
        let nextX = SLOTH_CONT.translate.x + (dx * 0.4);
        
        // Clamp the movement to an invisible bounding box
        const MIN_X = -100; // Far left boundary
        const MAX_X = 160;  // Far right boundary
        nextX = Math.max(MIN_X, Math.min(MAX_X, nextX));
        
        // Only update the procedural animation if the sloth actually moved 
        // (prevents arms moving while stuck at the boundary)
        if (SLOTH_CONT.translate.x !== nextX) {
          SLOTH_CONT.translate.x = nextX;
          
          let walkCycle = SLOTH_CONT.translate.x * 0.025;
          leftArm.rotate.z  = Math.sin(walkCycle) * 0.35;
          rightArm.rotate.z = Math.cos(walkCycle) * 0.35;
          leftLeg.rotate.z  = Math.cos(walkCycle) * 0.35;
          rightLeg.rotate.z = Math.sin(walkCycle) * 0.35;
          PIVOT.rotate.z    = Math.sin(walkCycle * 2) * 0.05; // Gentle body sway
        }
      }
      scene.updateRenderGraph();
    }
    
    function onPointerUp() {
      isDragging = false;
      
      // Fall asleep when let go, unless a "wake cycle" (double click) is actively running
      if (!isBusy) {
        setExpression('sleepy');
      }

      // Gentle settle back into natural resting position after dragging
      if (wasDragging) {
        gsap.to([leftArm.rotate, rightArm.rotate, leftLeg.rotate, rightLeg.rotate, PIVOT.rotate], { 
          z: 0, duration: 1.5, ease: 'power1.inOut' 
        });
      }
    }

    function onCanvasClick(e) {
      if (wasDragging) return;
      
      if (clickTimeout !== null) {
        // Double click detected
        clearTimeout(clickTimeout);
        clickTimeout = null;
        triggerWake();
      } else {
        // Single click detected, wait to see if a second click comes
        clickTimeout = setTimeout(() => {
          clickTimeout = null;
          // Single clicks do nothing by design
        }, 250); 
      }
    }

    if (!embedded) {
      canvasRef.addEventListener('click',       onCanvasClick);
      canvasRef.addEventListener('pointerdown', onPointerDown);
      canvasRef.addEventListener('pointermove', onPointerMove);
      window.addEventListener('pointerup',      onPointerUp);
    }

    return () => {
      isRunning = false;
      if (thoughtTimer) clearTimeout(thoughtTimer);
      if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
      if (clickTimeout) clearTimeout(clickTimeout);
      if (!embedded) {
        canvasRef?.removeEventListener('click',       onCanvasClick);
        canvasRef?.removeEventListener('pointerdown', onPointerDown);
        canvasRef?.removeEventListener('pointermove', onPointerMove);
        window.removeEventListener('pointerup',       onPointerUp);
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

<p class="hint">Drag the sloth to help it climb. Double-click to wake it up.</p>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap");

  :global(body) {
    margin: 0; padding: 0;
    width: 100vw; height: 100vh;
    background: transparent; 
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

  /* ── Thought bubble (Anchored Right) ──────────────────────────────────────────────── */
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
    border: 2px solid rgba(93,160,82,.30);
    border-radius: 22px 22px 6px 22px; 
    padding: 14px 18px;
    box-shadow: 0 8px 36px rgba(40,55,30,.14);
    position: relative;
    max-width: 100%;
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
  .thought-reply { display: flex; gap: 8px; align-items: flex-end; width: 100%; }
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
</style>