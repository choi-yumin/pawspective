<script>
  //@ts-nocheck
  import { onMount, createEventDispatcher } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';
  import ChickenBackground from './ChickenBackground.svelte';

  const dispatch = createEventDispatcher();
  let canvasRef;

  export let embedded = false;

  let radialMenuOpen = false;
  let activeCard = null;

  const interactionCards = [
    {
      id: 'chat',
      icon: '💬',
      label: 'Chat',
      title: 'Answer a thought',
      description: 'The chicken puts on its glasses, fixes its gaze, and thoughtfully responds to you.',
      hint: 'A brilliant mind at work.'
    },
    {
      id: 'peck',
      icon: '🌾',
      label: 'Peck',
      title: 'Peck the ground',
      description: 'The chicken bobs down and pecks at the grass a few times, looking for seeds.',
      hint: 'A quick little snack break.'
    },
    {
      id: 'strut',
      icon: '💃',
      label: 'Strut',
      title: 'Do a little strut',
      description: 'The chicken struts side to side and kicks up its feet, showing off a bit.',
      hint: 'Pure chicken swagger.'
    }
  ];

  let chatFn, peckFn, strutFn;

  function goHome() { dispatch('back'); }

  function openCard(id) {
    activeCard = interactionCards.find(c => c.id === id);
    radialMenuOpen = false;
  }

  function closeCard() { activeCard = null; }

  function executeCard() {
    if (!activeCard) return;
    if (activeCard.id === 'chat')   chatFn?.();
    if (activeCard.id === 'peck')   peckFn?.();
    if (activeCard.id === 'strut')  strutFn?.();
    activeCard = null;
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

    function handleClick(e) {
      if (wasDragging) return;
      const rect = canvasRef.getBoundingClientRect();
      const x = e.clientX - rect.left - rect.width / 2;
      const y = e.clientY - rect.top - rect.height / 2;
      const dist = Math.hypot(x - chicken.translate.x, y - chicken.translate.y);
      if (dist < 450) {
        radialMenuOpen = !radialMenuOpen;
        activeCard = null;
      } else {
        radialMenuOpen = false;
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

{#if !embedded && radialMenuOpen}
  <div class="radial-overlay">
    {#each interactionCards as card, i}
      {@const angle = (i / interactionCards.length) * 360 - 90}
      {@const rad   = angle * (Math.PI / 180)}
      {@const r     = 120}
      <button
        class="radial-btn"
        style="
          left: calc(50% + {Math.cos(rad) * r}px - 38px);
          top:  calc(58% + {Math.sin(rad) * r}px - 38px);
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

<p class="hint">Tap the chicken to open interactions ✿</p>

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

  /* ── Radial menu ─────────────────────────────────────────────────── */
  .radial-overlay {
    position: fixed; inset: 0;
    z-index: 20;
    pointer-events: none;
  }
  .radial-btn {
    position: fixed;
    width: 76px; height: 76px;
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
  .radial-btn:hover { background: #FF8FAE; transform: scale(1.12); }
  .radial-btn:hover .radial-label { color: #fff; }
  @keyframes radialPop {
    from { transform: scale(0); opacity: 0; }
    to   { transform: scale(1);   opacity: 1; }
  }
  .radial-icon  { font-size: 1.6rem; line-height: 1; }
  .radial-label { font-size: 0.66rem; font-weight: 800; color: #7A2D44; letter-spacing: .02em; }

  .radial-center {
    position: fixed;
    left: calc(50% - 24px);
    top:  calc(58% - 24px);
    width: 48px; height: 48px;
    border: none; border-radius: 50%;
    background: rgba(255, 143, 174, 0.902);
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