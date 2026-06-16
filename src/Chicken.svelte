<script>
  //@ts-nocheck
  import { onMount, createEventDispatcher } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';
  import ChickenBackground from './ChickenBackground.svelte';

  const dispatch = createEventDispatcher();
  let canvasRef;

  export let embedded = false;

  const OPENAI_API_KEY = import.meta.env.VITE_OPENAI_API_KEY;

  let thoughtBubbleVisible = true;
  let thoughtBubbleText = 'Bawk bawk… got a question for this thoughtful chicken?';
  let replyInputValue = '';
  let isApiLoading = false;

  let chatHistory = [
    { role: 'assistant', content: 'Bawk bawk… got a question for this thoughtful chicken?' }
  ];

  const SYSTEM_TEXT = `You are Professor Cluck, a clever, warm, slightly silly chicken. You answer in short, helpful messages, usually 2-4 sentences max. You can use light chicken humor, but keep answers useful and clear.`;

  let chatFn, peckFn, strutFn;

  function goHome() { dispatch('back'); }

  function showThought(text) {
    thoughtBubbleText = text;
    thoughtBubbleVisible = true;
  }

  async function sendReply() {
    const msg = replyInputValue.trim();
    if (!msg || isApiLoading) return;
    replyInputValue = '';
    await askChicken(msg);
  }

  function handleReplyKey(e) {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      sendReply();
    }
  }

  async function askChicken(userMessage) {
    if (isApiLoading) return;
    isApiLoading = true;
    showThought('…');
    chatFn?.();

    chatHistory = [...chatHistory, { role: 'user', content: userMessage }];

    try {
      const res = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${OPENAI_API_KEY}`
        },
        body: JSON.stringify({
          model: 'gpt-4o-mini',
          messages: [
            { role: 'system', content: SYSTEM_TEXT },
            ...chatHistory
          ],
          max_tokens: 256,
          temperature: 0.85
        })
      });

      if (!res.ok) throw new Error(`API returned status ${res.status}`);

      const data = await res.json();
      const replyText = data?.choices?.[0]?.message?.content?.trim();

      if (replyText) {
        chatHistory = [...chatHistory, { role: 'assistant', content: replyText }];
        showThought(replyText);
      } else {
        showThought('Bawk… my thought got scrambled. Try asking again?');
      }
    } catch (e) {
      chatHistory = chatHistory.slice(0, -1);
      showThought('Bawk… my thought got scrambled. Try asking again?');
    } finally {
      isApiLoading = false;
    }
  }

  onMount(() => {
    if (!canvasRef) return;
    const TAU = Zdog.TAU;

    const color = {
      body: '#FFF1E6', headBase: '#FFF5E8', eye: '#FFFFFF', pupil: '#111111',
      comb: '#FF3B5C', tail: '#F8D7C7', leg: '#FAA353',
      sky: '#FFB7D5', skyLight: '#FFE1F0',
      land: '#7ab535', landLight: '#9fd95c', darkGreen: '#3A5C18',
      trunkBrown: '#8B6340', cloudPink: '#FFE4F0', cloudWhite: '#FFF7FB',
      blossomPink: '#FFACD2', appleRed: '#FF4B4B',
      leafGreen: '#5DA020', daisyWhite: '#FFFFFF', daisyYellow: '#F9D342',
      roseRed: '#F04A6F', rosePink: '#FF8FAE',
      lavPurple: '#C07EE0', lavLight: '#E2B8F5',
      sunflowerY: '#FFB800', sunflowerC: '#7A3B10',
      tulipPink: '#FF85A2', tulipOrange: '#FF6B35'
    };

    const scene = new Zdog.Illustration({ element: canvasRef, dragRotate: false, resize: 'window' });

    // ─── Flower clusters ──────────────────────────────────────────────
    function createDaisy(parent, x, y, z, scale, stemLen, rotX, rotY, petalCol) {
      petalCol = petalCol || color.daisyWhite;
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (120 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen, z: 0 }, { x: 0, y: 0, z: 0 }], stroke: 7, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 13; i++) {
        const ph = new Zdog.Anchor({ addTo: hd, rotate: { z: (TAU / 13) * i } });
        new Zdog.Ellipse({ addTo: ph, width: 11, height: 30, fill: true, color: petalCol, stroke: 0, translate: { y: -18 } });
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 15, fill: true, stroke: 4, color: color.daisyYellow, translate: { z: 1.5 } });
    }

    function createRose(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (140 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen }, { x: 0, y: 0 }], stroke: 7, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let l = 0; l < 3; l++) {
        const pc = 6 + l * 2, r = 10 + l * 9;
        for (let i = 0; i < pc; i++) {
          const a = (TAU / pc) * i + l * 0.3;
          const ph = new Zdog.Anchor({ addTo: hd, translate: { x: Math.cos(a) * r, y: Math.sin(a) * r } });
          new Zdog.Ellipse({ addTo: ph, width: l === 0 ? 8 : 10 + l * 2, height: l === 0 ? 12 : 14 + l * 3, fill: true, color: l === 0 ? color.roseRed : color.rosePink, stroke: 0, rotate: { z: a } });
        }
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 8, fill: true, stroke: 0, color: '#c0243a', translate: { z: 1 } });
    }

    function createLavender(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (150 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen }, { x: 0, y: 0 }], stroke: 6, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 8; i++) {
        const ph = new Zdog.Anchor({ addTo: hd, translate: { x: 0, y: -i * 10 }, rotate: { z: i % 2 === 0 ? 0.3 : -0.3 } });
        new Zdog.Ellipse({ addTo: ph, width: 9, height: 14, fill: true, color: i < 3 ? color.lavPurple : color.lavLight, stroke: 0, translate: { x: 6 } });
        new Zdog.Ellipse({ addTo: ph, width: 9, height: 14, fill: true, color: i < 3 ? color.lavPurple : color.lavLight, stroke: 0, translate: { x: -6 } });
      }
    }

    function createSunflower(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (160 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen }, { x: 0, y: 0 }], stroke: 9, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 18; i++) {
        const ph = new Zdog.Anchor({ addTo: hd, rotate: { z: (TAU / 18) * i } });
        new Zdog.Ellipse({ addTo: ph, width: 13, height: 36, fill: true, color: color.sunflowerY, stroke: 0, translate: { y: -24 } });
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 22, fill: true, stroke: 5, color: color.sunflowerC, translate: { z: 2 } });
    }

    function createTulip(parent, x, y, z, scale, stemLen, rotX, rotY, col) {
      col = col || color.tulipPink;
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (130 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{ x: 0, y: stemLen }, { x: 0, y: 0 }], stroke: 8, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 6; i++) {
        const a = (TAU / 6) * i;
        new Zdog.Ellipse({ addTo: hd, width: 18, height: 36, fill: true, color: col, stroke: 0, translate: { x: Math.cos(a) * 10, z: Math.sin(a) * 10 }, rotate: { y: a, x: -0.3 } });
      }
    }

    // ─── Chicken ──────────────────────────────────────────────────────
    let chickenLeftEyeGroup, chickenRightEyeGroup, cEyeLeftPupil, cEyeRightPupil, glassesAnchor, rightHand, leftHand;
    const chicken = new Zdog.Anchor({ addTo: scene, translate: { x: 0, y: 70, z: 0 }, rotate: { x: -0.05, y: 0, z: 0 } });
    
    // NEW: Isolate the upper body so the legs don't move during the peck animation
    const chickenBody = new Zdog.Anchor({ addTo: chicken });

    let bodyLower = new Zdog.Shape({ addTo: chickenBody, stroke: 270, color: color.body, translate: { y: 75 } });
    new Zdog.Cylinder({ addTo: chickenBody, diameter: 72, length: 150, stroke: 36, color: '#F6D8C8', rotate: { x: TAU/4 }, translate: { y: -45, z: 18 } });

    const head = new Zdog.Anchor({ addTo: chickenBody, translate: { y: -150, z: 42 } });
    new Zdog.Shape({ addTo: head, stroke: 135, color: color.headBase });
    new Zdog.Cone({ addTo: head, diameter: 42, length: 66, stroke: 12, color: '#F4A63A', translate: { y: 30, z: 72 } });

    const wattleLeft = new Zdog.Shape({ addTo: head, path: [{ y: 0 }, { y: 48 }], stroke: 30, color: color.comb, translate: { x: -18, y: 42, z: 48 } });
    wattleLeft.copy({ translate: { x: 18, y: 42, z: 48 } });
    new Zdog.Shape({
      addTo: head, stroke: 36, color: color.comb,
      path: [{ x: -18, y: -66, z: 6 }, { x: -30, y: -108, z: 18 }, { x: -6, y: -84, z: 2 }, { x: 6, y: -126, z: -6 }, { x: 24, y: -78, z: -12 }]
    });

    chickenLeftEyeGroup = new Zdog.Anchor({ addTo: head, translate: { x: -60, y: -12, z: 42 } });
    new Zdog.Shape({ addTo: chickenLeftEyeGroup, stroke: 84, color: color.eye, fill: true });
    cEyeLeftPupil = new Zdog.Shape({ addTo: chickenLeftEyeGroup, stroke: 30, color: color.pupil, fill: true, translate: { x: -12, y: -12, z: 36 } });

    chickenRightEyeGroup = new Zdog.Anchor({ addTo: head, translate: { x: 60, y: -6, z: 36 } });
    new Zdog.Shape({ addTo: chickenRightEyeGroup, stroke: 72, color: color.eye, fill: true });
    cEyeRightPupil = new Zdog.Shape({ addTo: chickenRightEyeGroup, stroke: 27, color: color.pupil, fill: true, translate: { x: 9, y: 6, z: 30 } });

    let glassesLeft, glassesRight, glassesBridge;
    glassesAnchor = new Zdog.Anchor({ addTo: head, translate: { y: -6, z: 84 }, scale: 0.001 });
    glassesLeft = new Zdog.Ellipse({ addTo: glassesAnchor, diameter: 102, stroke: 15, color: '#1A1A1A', translate: { x: -54 }, visible: false });
    glassesRight = glassesLeft.copy({ translate: { x: 54 }, visible: false });
    glassesBridge = new Zdog.Shape({ addTo: glassesAnchor, path: [{ x: -20, y: 0 }, { x: 20, y: 0 }], stroke: 13, color: '#1A1A1A', visible: false });

    // Wings attached to chickenBody
    leftHand = new Zdog.Anchor({ addTo: chickenBody, translate: { x: -105, y: 30, z: 15 }, rotate: { y: 0.2, z: 0.3 } });
    new Zdog.Shape({
      addTo: leftHand, stroke: 32, color: color.body, closed: false,
      path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: -35, y: -30, z: -10 }, { x: -50, y: 20, z: -20 }, { x: -20, y: 55, z: -10 }] }]
    });
    new Zdog.Shape({
      addTo: leftHand, stroke: 24, color: color.tail, closed: false,
      path: [{ x: -5, y: 8, z: -3 }, { bezier: [{ x: -38, y: -20, z: -15 }, { x: -48, y: 25, z: -25 }, { x: -18, y: 48, z: -12 }] }]
    });

    rightHand = new Zdog.Anchor({ addTo: chickenBody, translate: { x: 105, y: 30, z: 15 }, rotate: { y: -0.2, z: -0.3 } });
    new Zdog.Shape({
      addTo: rightHand, stroke: 32, color: color.body, closed: false,
      path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: 35, y: -30, z: -10 }, { x: 50, y: 20, z: -20 }, { x: 20, y: 55, z: -10 }] }]
    });
    new Zdog.Shape({
      addTo: rightHand, stroke: 24, color: color.tail, closed: false,
      path: [{ x: 5, y: 8, z: -3 }, { bezier: [{ x: 38, y: -20, z: -15 }, { x: 48, y: 25, z: -25 }, { x: 18, y: 48, z: -12 }] }]
    });

    // Tail feathers attached to chickenBody
    const tailFeathers = [];
    const tailAnchor = new Zdog.Anchor({ addTo: chickenBody, translate: { x: -54, y: 36, z: -75 } });
    new Zdog.Shape({ addTo: tailAnchor, stroke: 36, color: color.tail, closed: false, path: [{ x: 0, y: 0, z: 0 }, { bezier: [{ x: -66, y: -66, z: -18 }, { x: -96, y: -144, z: -72 }, { x: -48, y: -192, z: -96 }] }] });
    tailFeathers.push(tailAnchor);
    for (let i = 1; i < 8; i++) {
      const copy = tailAnchor.copyGraph({ translate: { x: -54 - i * 9, y: 36 + i * 6, z: -75 - i * 6 }, rotate: { y: i * 0.06, z: i * 0.04 } });
      tailFeathers.push(copy);
    }

    // Legs explicitly stay attached to the base chicken
    const legLeg = new Zdog.Shape({ path: [{ y: 60 }, { y: 120 }], stroke: 27, color: color.leg });
    const leftLegAnchor = new Zdog.Anchor({ addTo: chicken, translate: { x: -42, y: 150, z: 0 } });
    legLeg.copy({ addTo: leftLegAnchor });
    const leftFootAnchor = new Zdog.Anchor({ addTo: leftLegAnchor, translate: { y: 120 }, rotate: { y: 0.3 } });
    new Zdog.Shape({ addTo: leftFootAnchor, path: [{ x: 0, z: 0 }, { x: 0, z: 45 }],  stroke: 27, color: color.leg });
    new Zdog.Shape({ addTo: leftFootAnchor, path: [{ x: 0, z: 0 }, { x: -36, z: 30 }], stroke: 27, color: color.leg });
    new Zdog.Shape({ addTo: leftFootAnchor, path: [{ x: 0, z: 0 }, { x: 36, z: 30 }],  stroke: 27, color: color.leg });

    const rightLegAnchor = new Zdog.Anchor({ addTo: chicken, translate: { x: 42, y: 150, z: 0 } });
    legLeg.copy({ addTo: rightLegAnchor });
    const rightFootAnchor = new Zdog.Anchor({ addTo: rightLegAnchor, translate: { y: 120 }, rotate: { y: -0.3 } });
    new Zdog.Shape({ addTo: rightFootAnchor, path: [{ x: 0, z: 0 }, { x: 0, z: 45 }],  stroke: 27, color: color.leg });
    new Zdog.Shape({ addTo: rightFootAnchor, path: [{ x: 0, z: 0 }, { x: -36, z: 30 }], stroke: 27, color: color.leg });
    new Zdog.Shape({ addTo: rightFootAnchor, path: [{ x: 0, z: 0 }, { x: 36, z: 30 }],  stroke: 27, color: color.leg });

    // ─── Animation loop ────────────────────────────────────────────
    let frame = 0, isRunning = true, isSmartMode = false;
    let chickenTargetLookX = 0, chickenLookX = 0;

    function render() {
      if (!isRunning) return;
      frame++;
      const blinkCycle = frame % 340;
      if (blinkCycle > 320) {
        const sY = 0.5 + 0.5 * Math.cos(((blinkCycle - 320) / 20) * Math.PI * 2);
        chickenLeftEyeGroup.scale.y = sY; chickenRightEyeGroup.scale.y = sY;
      } else {
        chickenLeftEyeGroup.scale.y = 1; chickenRightEyeGroup.scale.y = 1;
      }
      if (frame % 240 === 0) chickenTargetLookX = [-14, 0, 12][Math.floor(Math.random() * 3)];
      chickenLookX += (chickenTargetLookX - chickenLookX) * 0.08;
      if (!isSmartMode) {
        cEyeLeftPupil.translate.x  = -12 + chickenLookX;
        cEyeRightPupil.translate.x =   9 + chickenLookX;
      }
      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }
    render();

    // ─── Chat interaction (wears glasses and eyes fixed) ───────────────
    let isChatting = false;
    function chat() {
      if (isChatting) return;
      isChatting = true;
      isSmartMode = true;

      glassesLeft.visible = true; glassesRight.visible = true; glassesBridge.visible = true;

      const tl = gsap.timeline({
        onComplete: () => {
          gsap.to(glassesAnchor.scale, {
            duration: 0.4, x: 0.001, y: 0.001, z: 0.001, ease: 'power2.in',
            onComplete: () => {
              glassesLeft.visible = false; glassesRight.visible = false; glassesBridge.visible = false;
              isSmartMode = false;
              isChatting = false;
            }
          });
        }
      });

      // Rapidly centers eyes and pops up glasses
      tl.to(glassesAnchor.scale, { duration: 0.6, x: 1, y: 1, z: 1, ease: 'elastic.out(1, 0.75)' })
        .to(cEyeLeftPupil.translate,  { duration: 0.3, x: 0, y: 0, ease: 'power2.out' }, 0)
        .to(cEyeRightPupil.translate, { duration: 0.3, x: 0, y: 0, ease: 'power2.out' }, 0);

      // Conversational head bobs while answering thoughts
      tl.to(head.translate, { duration: 0.25, y: -142, yoyo: true, repeat: 7, ease: 'sine.inOut' }, 0.5)
        .to(leftHand.rotate, { duration: 0.25, z: 0.5, yoyo: true, repeat: 7, ease: 'sine.inOut' }, 0.5)
        .to(rightHand.rotate, { duration: 0.25, z: -0.5, yoyo: true, repeat: 7, ease: 'sine.inOut' }, 0.5);
    }

    // ─── Peck the ground (Deep Body Pivot) ─────────────────────────
    let isPecking = false;
    function peck() {
      if (isPecking || !chickenBody) return;
      isPecking = true;
      
      const tl = gsap.timeline({ onComplete: () => { isPecking = false; } });

      for (let i = 0; i < 3; i++) {
        // 1. Pivot the upper body drastically downward and bend the neck
        tl.to(chickenBody.rotate, { 
          duration: 0.15, 
          x: -1.2, // Deep forward tilt towards the ground
          ease: 'power2.in' 
        }, i * 0.35);

        tl.to(head.rotate, {
          duration: 0.15,
          x: -0.6, // Bend the neck further down
          ease: 'power2.in'
        }, i * 0.35);
        
        // 2. Snap back to standing pose
        tl.to(chickenBody.rotate, { 
          duration: 0.2, 
          x: 0, // Back to rest
          ease: 'power2.out' 
        }, i * 0.35 + 0.15);

        tl.to(head.rotate, {
          duration: 0.2,
          x: 0, 
          ease: 'power2.out'
        }, i * 0.35 + 0.15);
      }
    }

    // ─── Strut dance ────────────────────────────────────────────────
    let isStrutting = false;
    function strut() {
      if (isStrutting) return;
      isStrutting = true;
      const tl = gsap.timeline({
        onComplete: () => {
          isStrutting = false;
          chicken.rotate.z = 0; chicken.translate.y = 70;
          leftLegAnchor.rotate.z = 0; rightLegAnchor.rotate.z = 0;
        }
      });
      for (let i = 0; i < 4; i++) {
        const dir = i % 2 === 0 ? 1 : -1;
        tl.to(chicken.rotate,        { duration: 0.18, z: dir * 0.07, ease: 'sine.inOut' });
        tl.to(chicken.translate,     { duration: 0.18, y: 58,         ease: 'sine.out'   }, '<');
        tl.to(leftLegAnchor.rotate,  { duration: 0.18, z:  dir * 0.18, ease: 'sine.inOut' }, '<');
        tl.to(rightLegAnchor.rotate, { duration: 0.18, z: -dir * 0.18, ease: 'sine.inOut' }, '<');
        tl.to(chicken.translate,     { duration: 0.18, y: 70,         ease: 'sine.in'    });
      }
    }

    chatFn = chat;
    peckFn = peck;
    strutFn = strut;

    // ─── Pointer interactions ─────────────────────────────────────
    let isDragging = false, lastX = 0, lastY = 0, wasDragging = false;
    let clickTimeout = null;
    const sensitivity = 0.005;

    function onPointerDown(e) {
      isDragging = true; wasDragging = false;
      lastX = e.clientX; lastY = e.clientY;
      try { canvasRef.setPointerCapture(e.pointerId); } catch (_) {}
    }
    function onPointerMove(e) {
      if (!isDragging) return;
      const dx = e.clientX - lastX, dy = e.clientY - lastY;
      lastX = e.clientX; lastY = e.clientY;
      if (Math.hypot(dx, dy) > 8) wasDragging = true;
      chicken.rotate.y += dx * sensitivity;
      scene.updateRenderGraph();
    }
    function onPointerUp(e) {
      isDragging = false;
      try { canvasRef.releasePointerCapture(e.pointerId); } catch (_) {}
    }

    function handleClick() {
      if (wasDragging) return;

      if (clickTimeout !== null) {
        clearTimeout(clickTimeout);
        clickTimeout = null;
        strutFn?.();
      } else {
        clickTimeout = setTimeout(() => {
          clickTimeout = null;
          peckFn?.();
        }, 250);
      }
    }

    if (!embedded) {
      canvasRef.addEventListener('pointerdown', onPointerDown);
      canvasRef.addEventListener('pointermove', onPointerMove);
      window.addEventListener('pointerup', onPointerUp);
      canvasRef.addEventListener('click', handleClick);
    }

    return () => {
      isRunning = false;
      if (clickTimeout) clearTimeout(clickTimeout);
      if (!embedded) {
        canvasRef?.removeEventListener('pointerdown', onPointerDown);
        canvasRef?.removeEventListener('pointermove', onPointerMove);
        window.removeEventListener('pointerup', onPointerUp);
        canvasRef?.removeEventListener('click', handleClick);
      }
    };
  });
</script>

{#if !embedded}
  <div class="top-bar">
    <button class="bar-btn" on:click={goHome}>←</button>
  </div>
{/if}

{#if !embedded}
  <ChickenBackground />
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
      placeholder="Ask Professor Cluck…"
      rows="1"
      disabled={isApiLoading}
      on:keydown={handleReplyKey}
    ></textarea>
    <button class="reply-send" on:click={sendReply} disabled={isApiLoading || !replyInputValue.trim()}>
      {isApiLoading ? '…' : '➤'}
    </button>
  </div>
</div>

<p class="hint">Drag to rotate. Click to peck. Double-click to strut.</p>

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

  /* ── Chatbot ─────────────────────────────────────────────────── */
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
    border: 2px solid rgba(255,143,174,.32);
    border-radius: 22px 22px 6px 22px;
    padding: 14px 18px;
    box-shadow: 0 8px 36px rgba(90,26,48,.14);
    position: relative;
    max-width: 100%;
  }
  .thought-dots { display: flex; gap: 4px; margin-bottom: 6px; }
  .thought-dots span { width: 5px; height: 5px; background: #FF8FAE; border-radius: 50%; }
  .thought-text {
    margin: 0;
    font-size: 0.92rem;
    color: #5A1A30;
    line-height: 1.6;
    font-weight: 600;
  }
  .thought-reply { display: flex; gap: 8px; align-items: flex-end; width: 100%; }
  .thought-reply textarea {
    flex: 1;
    background: rgba(255,255,255,0.96);
    border: 1.5px solid rgba(255,143,174,.42);
    border-radius: 14px;
    padding: 10px 14px;
    font-family: 'Nunito', sans-serif;
    font-size: 0.9rem;
    color: #5A1A30;
    outline: none; resize: none;
    height: 44px; min-height: 44px; max-height: 88px;
    line-height: 1.5; box-sizing: border-box;
    transition: border-color .15s, box-shadow .15s;
  }
  .thought-reply textarea::placeholder { color: rgba(90,26,48,.42); }
  .thought-reply textarea:focus {
    border-color: #F04A6F;
    box-shadow: 0 0 0 3px rgba(240,74,111,.16);
  }
  .reply-send {
    width: 44px; height: 44px;
    border-radius: 50%; border: none;
    background: #FF8FAE; color: #fff;
    font-size: 1rem; cursor: pointer;
    flex-shrink: 0;
    transition: background .12s, transform .1s;
    display: flex; align-items: center; justify-content: center;
  }
  .reply-send:hover:not(:disabled) { background: #F04A6F; }
  .reply-send:active:not(:disabled) { transform: scale(.94); }
  .reply-send:disabled { opacity: .5; cursor: default; }

  /* ── Hint ────────────────────────────────────────────────────────── */
  .hint {
    position: fixed;
    bottom: 22px; left: 50%;
    transform: translateX(-50%);
    font-size: 0.8rem; font-weight: 600;
    color: rgba(90, 26, 48, 0.55);
    pointer-events: none;
    letter-spacing: .04em;
    z-index: 10;
  }
</style>