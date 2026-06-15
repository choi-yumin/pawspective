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
      id: 'startle',
      icon: '😯',
      label: 'Startle',
      title: 'Startle the dove',
      description: 'A sudden poke sends Olive into a flurry of wingbeats — then calm returns.',
      hint: 'Or give the dove a quick tug.',
      angle: -90
    },
    {
      id: 'fly',
      icon: '🕊️',
      label: 'Fly',
      title: 'Take flight',
      description: 'Olive lifts off and circles on a slow, easy loop.',
      hint: 'Watch the wings open wide.',
      angle: -30
    },
    {
      id: 'coo',
      icon: '🎵',
      label: 'Coo',
      title: 'Make it coo',
      description: 'A soft chest-puff and a gentle, bobbing coo.',
      hint: 'Doves are happiest when they sing.',
      angle: 30
    },
    {
      id: 'peace',
      icon: '🌿',
      label: 'Peace',
      title: 'Offer peace',
      description: 'Olive bows her head and holds out an olive branch.',
      hint: 'The dove\'s oldest gift.',
      angle: 90
    },
    {
      id: 'chat',
      icon: '💬',
      label: 'Chat',
      title: 'Talk with Olive',
      description: 'The dove has soft wisdom about calm, hope, and letting go.',
      hint: 'Reply to Olive\'s thought bubbles below.',
      angle: 150
    }
  ];

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
  let cooFn;
  let peaceFn;
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
    if (activeCard.id === 'startle') startleFn?.();
    if (activeCard.id === 'fly')     flyFn?.();
    if (activeCard.id === 'coo')     cooFn?.();
    if (activeCard.id === 'peace')   peaceFn?.();
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
        showThought("Coo… the wind carried my thought away. Ask me again?", true);
      }
    } catch (e) {
      console.error('OpenAI Catch block triggered:', e);
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
      white:    '#F4F2EE',
      mid:      '#E8E5DE',
      shade:    '#DAD6CE',
      breast:   '#FBF9F5',
      beak:     '#D89A6E',
      beakDk:   '#C07E50',
      cheek:    '#F0A8A0',
      eye:      '#1A1410',
      brow:     '#CFC9BE',
      crest:    '#E2DED6',
      foot:     '#D89A6E',
      oliveStem:'#6E8B3D',
      oliveLeaf:'#8AA653',
      olive:    '#46662C',
      skyTop:   '#BFD9EF',
      skyLow:   '#E7F1FA',
      sun:      '#FFF7E6',
      cloud:    '#FFFFFF',
      cloudDim: '#EAF2FA'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.1, y: 0.18, z: 0 },
      zoom: 1.0
    });
    // background is rendered in DoveBackground.svelte

    const fg = new Zdog.Anchor({ addTo: scene });
    const DOVE_CONT = new Zdog.Anchor({ addTo: fg, translate: { x: 0, y: 0, z: 0 }, scale: 1.35 });
    const PIVOT = new Zdog.Anchor({ addTo: DOVE_CONT });
    const DOVE  = new Zdog.Anchor({ addTo: PIVOT });

    // Body — the dove's own form: a sleek flat ellipse + a rounded breast.
    // (Deliberately NOT the bee's stacked-sphere torso.)
    new Zdog.Ellipse({ addTo: DOVE, width: 138, height: 42, stroke: 18, color: C.white,  fill: true, translate: { y: 8 } });
    new Zdog.Ellipse({ addTo: DOVE, width: 120, height: 30, stroke: 10, color: C.shade,  fill: true, translate: { y: 18, z: -4 } });
    new Zdog.Ellipse({ addTo: DOVE, width: 48,  height: 46, stroke: 14, color: C.breast, fill: true, translate: { x: -58, y: 2, z: 6 } });
    new Zdog.Ellipse({ addTo: DOVE, width: 26,  height: 30, stroke: 12, color: C.white,  fill: true, translate: { x: -74, y: -6, z: 2 } });

    // Tucked feet
    new Zdog.Shape({ addTo: DOVE, path: [{ x: -14, y: 26 }, { x: -14, y: 34 }], stroke: 3, color: C.foot });
    new Zdog.Shape({ addTo: DOVE, path: [{ x: 2, y: 26 },  { x: 2, y: 34 }],  stroke: 3, color: C.foot });
    [-14, 2].forEach(x => { for (let i = -1; i <= 1; i++) new Zdog.Shape({ addTo: DOVE, path: [{ x, y: 34 }, { x: x + i * 4, y: 38 }], stroke: 2, color: C.foot }); });

    // Fanned tail
    const tail = new Zdog.Anchor({ addTo: DOVE, translate: { x: 66, y: 12 } });
    [-24, -14, -5, 5, 14, 24].forEach((a, i) => {
      new Zdog.Shape({
        addTo: tail,
        path: [{ x: 0, y: 0 }, { bezier: [{ x: 25, y: a * 0.3 }, { x: 55, y: a * 0.65 }, { x: 78, y: a * 0.85 }] }],
        stroke: (i === 0 || i === 5) ? 4 : 6, color: (i === 0 || i === 5) ? C.shade : C.mid
      });
    });

    // Wings — layered bezier shapes on anchors so they can flap
    const wingPath = [
      { x: 0, y: 0 },
      { bezier: [{ x: -30, y: -20 }, { x: -70, y: -70 }, { x: -50, y: -150 }] },
      { bezier: [{ x: -20, y: -130 }, { x: 20, y: -110 }, { x: 60, y: -80 }] },
      { bezier: [{ x: 90, y: -55 }, { x: 90, y: -20 }, { x: 60, y: -5 }] },
      { bezier: [{ x: 40, y: 4 }, { x: 20, y: 4 }, { x: 0, y: 0 }] }
    ];
    const backWing = new Zdog.Anchor({ addTo: DOVE, translate: { x: 40, y: 8, z: -18 } });
    new Zdog.Shape({ addTo: backWing, path: wingPath, stroke: 3, color: C.shade, fill: true });
    const frontWing = new Zdog.Anchor({ addTo: DOVE, translate: { x: 40, y: 8, z: 18 } });
    new Zdog.Shape({ addTo: frontWing, path: wingPath, stroke: 3, color: C.mid, fill: true });

    // Head — turned three-quarters toward the camera for an expressive face
    const head = new Zdog.Anchor({ addTo: DOVE, translate: { x: -82, y: -10 }, rotate: { y: -0.5 } });
    new Zdog.Shape({ addTo: head, stroke: 34, color: C.white });
    new Zdog.Ellipse({ addTo: head, diameter: 30, stroke: 8, color: C.shade, fill: true, translate: { z: -8 } });
    [-0.3, 0, 0.3].forEach((r, i) => new Zdog.Shape({ addTo: head, path: [{ x: 0, y: -16 }, { x: i * 4, y: -26, z: 2 }], stroke: 3, color: C.crest, rotate: { z: r } }));
    new Zdog.Shape({ addTo: head, stroke: 9, color: C.cheek, translate: { x: -3, y: 6, z: 13 } });
    new Zdog.Shape({ addTo: head, stroke: 7, color: C.eye, translate: { x: -3, y: -3, z: 16 } });
    new Zdog.Shape({ addTo: head, stroke: 2.5, color: '#fff', translate: { x: -4, y: -5, z: 18 } });
    new Zdog.Shape({ addTo: head, stroke: 5, color: C.eye, translate: { x: 13, y: -3, z: 8 } });
    const brow = new Zdog.Shape({ addTo: head, path: [{ x: -9, y: -10 }, { x: 3, y: -12 }], stroke: 3, color: C.brow, translate: { z: 15 } });
    const lid = new Zdog.Shape({ addTo: head, stroke: 8, color: C.white, translate: { x: -3, y: -11, z: 19 } });

    // Openable beak: fixed upper + lower on a hinge
    const beak = new Zdog.Anchor({ addTo: head, translate: { x: -14, y: 3, z: 11 } });
    new Zdog.Shape({ addTo: beak, path: [{ x: 0, y: -3 }, { x: -22, y: 1 }, { x: 0, y: 3 }], stroke: 2, color: C.beak, fill: true });
    const lowerBeak = new Zdog.Anchor({ addTo: beak, translate: { x: 0, y: 3 } });
    new Zdog.Shape({ addTo: lowerBeak, path: [{ x: 0, y: 0 }, { x: -20, y: 0 }, { x: 0, y: 4 }], stroke: 2, color: C.beakDk, fill: true });

    // Olive sprig — hidden until the Peace interaction
    const sprig = new Zdog.Anchor({ addTo: beak, translate: { x: -24, y: 2, z: 0 } });
    const sprigParts = [];
    sprigParts.push(new Zdog.Shape({ addTo: sprig, path: [{ x: 0, y: 0 }, { x: -22, y: -6 }], stroke: 2.5, color: C.oliveStem, visible: false }));
    [[-6, -2, 0.6], [-13, -5, 0.3], [-20, -8, 0.5]].forEach(([x, y, r]) =>
      sprigParts.push(new Zdog.Ellipse({ addTo: sprig, width: 10, height: 5, stroke: 2, color: C.oliveLeaf, fill: true, translate: { x, y }, rotate: { z: r }, visible: false })));
    sprigParts.push(new Zdog.Shape({ addTo: sprig, stroke: 5, color: C.olive, translate: { x: -9, y: -1 }, visible: false }));

    // ─── Expression system (transforms only) ───────────────────────────
    let lidBase = -11;
    function setExpression(name) {
      let brz = 0, lidy = -11, beakOpen = 0;
      if (name === 'calm')         { brz = 0;     lidy = -10; beakOpen = 0; }
      else if (name === 'content') { brz = -0.05; lidy = -11; beakOpen = 0.12; }
      else if (name === 'happy')   { brz = 0.12;  lidy = -13; beakOpen = 0.28; }
      else if (name === 'alarmed') { brz = -0.22; lidy = -14; beakOpen = 0.38; }
      else if (name === 'curious') { brz = 0.18;  lidy = -12; beakOpen = 0.06; }
      brow.rotate.z = brz;
      lidBase = lidy; lid.translate.y = lidy;
      lowerBeak.rotate.z = beakOpen;
    }
    setExpression('calm');

    // ─── Idle + flutter + blink loop ───────────────────────────────────
    let frame = 0, isRunning = true, isBusy = false, blinkTimer = 0, wingBeat = 0;
    const flap = { amp: 0.09, speed: 1 };

    function render() {
      if (!isRunning) return;
      frame++;

      // wings always flutter; flap state swells during startle / flight
      wingBeat += 0.13 * flap.speed;
      frontWing.rotate.z = Math.sin(wingBeat) * flap.amp;
      backWing.rotate.z  = Math.sin(wingBeat - 0.5) * flap.amp;

      if (!isBusy) {
        DOVE_CONT.translate.y = Math.sin(frame / 40) * 4;   // gentle hover
        head.rotate.z = Math.sin(frame / 46) * 0.05;        // soft head bob
        tail.rotate.z = Math.sin(frame / 60) * 0.04;
      }

      blinkTimer++;
      let blink = 0;
      if (blinkTimer > 200) {
        const t = blinkTimer - 200;
        if (t < 12) blink = Math.sin((t / 12) * Math.PI) * 8;
        else blinkTimer = 0;
      }
      lid.translate.y = lidBase + blink;

      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }
    render();

    // ─── Interactions ──────────────────────────────────────────────────
    function startle() {
      if (isBusy) return;
      isBusy = true;
      setExpression('alarmed');
      gsap.to(flap, { duration: 0.1, amp: 0.5, speed: 3 });
      const tl = gsap.timeline({
        onComplete: () => {
          setExpression('calm');
          gsap.to(flap, { duration: 0.7, amp: 0.09, speed: 1 });
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
      gsap.to(flap, { duration: 0.2, amp: 0.55, speed: 3 });
      const tl = gsap.timeline({
        onComplete: () => {
          setExpression('calm');
          gsap.to(flap, { duration: 0.7, amp: 0.09, speed: 1 });
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

    function coo() {
      if (isBusy) return;
      isBusy = true;
      setExpression('content');
      const tl = gsap.timeline({ onComplete: () => { setExpression('calm'); isBusy = false; head.rotate.z = 0; } });
      tl.to(DOVE.scale, { duration: 0.3, x: 1.05, y: 1.05, ease: 'sine.inOut' });   // chest puff
      for (let i = 0; i < 3; i++) {
        tl.add(() => setExpression('happy'));
        tl.to(head.rotate, { duration: 0.22, z: 0.12, ease: 'sine.inOut' });
        tl.add(() => setExpression('content'));
        tl.to(head.rotate, { duration: 0.22, z: -0.02, ease: 'sine.inOut' });
      }
      tl.to(DOVE.scale, { duration: 0.3, x: 1, y: 1, ease: 'sine.inOut' });
      tl.to(head.rotate, { duration: 0.3, z: 0, ease: 'sine.inOut' });
    }

    function peace() {
      if (isBusy) return;
      isBusy = true;
      setExpression('content');
      sprigParts.forEach(p => p.visible = true);
      const tl = gsap.timeline({
        onComplete: () => { sprigParts.forEach(p => p.visible = false); setExpression('calm'); isBusy = false; head.rotate.x = 0; PIVOT.rotate.z = 0; }
      });
      tl.to(head.rotate,  { duration: 0.7, x: 0.32, ease: 'power2.out' });   // bow
      tl.to(PIVOT.rotate, { duration: 0.7, z: 0.06, ease: 'power2.out' }, '<');
      tl.to({}, { duration: 1.3 });
      tl.to(head.rotate,  { duration: 0.8, x: 0, ease: 'power1.inOut' });
      tl.to(PIVOT.rotate, { duration: 0.8, z: 0, ease: 'power1.inOut' }, '<');
      askOlive("I am bowing and holding out an olive branch to offer peace. Share a short, hopeful thought about making peace or letting go of a grudge.");
    }

    function talkingAnimation() {
      if (isBusy) return;
      isBusy = true;
      setExpression('content');
      const tl = gsap.timeline({ onComplete: () => { setExpression('calm'); isBusy = false; head.rotate.z = 0; } });
      for (let i = 0; i < 3; i++) {
        tl.add(() => setExpression('happy'));
        tl.to(head.rotate, { duration: 0.25, z: 0.08, ease: 'sine.inOut' });
        tl.add(() => setExpression('content'));
        tl.to(head.rotate, { duration: 0.25, z: 0, ease: 'sine.inOut' });
      }
    }

    startleFn  = startle;
    flyFn      = fly;
    cooFn      = coo;
    peaceFn    = peace;
    talkingFn  = talkingAnimation;

    // ─── Pointer: drag to ruffle / tug to startle ──────────────────────
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
      if (dragDist > 80 && !isBusy) startle();
    }

    function onCanvasClick(e) {
      if (wasDragging) return;
      const rect = canvasRef.getBoundingClientRect();
      const cx = (e.clientX - rect.left - rect.width  / 2);
      const cy = (e.clientY - rect.top  - rect.height / 2);
      // the dove is wide and flat — a generous box opens the menu
      if (Math.abs(cx) < 190 && Math.abs(cy) < 150) {
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

    if (!embedded) {
      setTimeout(() => showThought("Coo… the sky is calm today. 🕊️ What is on your mind?"), 3500);
      scheduleThought();
    }

    return () => {
      isRunning = false;
      if (thoughtTimer) clearTimeout(thoughtTimer);
      if (bubbleAutoHideTimer) clearTimeout(bubbleAutoHideTimer);
      if (!embedded) {
        canvasRef?.removeEventListener('click',       onCanvasClick);
        canvasRef?.removeEventListener('pointerdown', onPointerDown);
        canvasRef?.removeEventListener('pointermove', onPointerMove);
        window.removeEventListener('pointerup',       onPointerUp);
      }
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
          <span class="history-label">{msg.role === 'assistant' ? '🕊️ Olive' : 'You'}</span>
          <p>{msg.content}</p>
        </div>
      {/each}
    </div>
  </div>
{/if}

<!-- ─── Canvas ─────────────────────────────────────────────────────── -->
{#if !embedded}
  <DoveBackground />
{/if}
<canvas bind:this={canvasRef} class="scene" class:embedded></canvas>

<!-- ─── Radial menu ────────────────────────────────────────────────── -->
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

<!-- ─── Hint ───────────────────────────────────────────────────────── -->
<p class="hint">Tap the dove to open interactions 🕊️</p>

<div class="credit">Made by MakMin</div>

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

  /* ── Radial menu ─────────────────────────────────────────────────── */
  .radial-overlay { position: fixed; inset: 0; z-index: 20; pointer-events: none; }
  .radial-btn {
    position: fixed;
    width: 72px; height: 72px;
    border: none; border-radius: 50%;
    background: rgba(255,255,255,0.95);
    box-shadow: 0 6px 28px rgba(46,58,71,.18);
    cursor: pointer;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    gap: 2px;
    pointer-events: all;
    animation: radialPop .25s cubic-bezier(.34,1.56,.64,1) both;
    transition: transform .12s, background .12s;
  }
  .radial-btn:hover { background: #9CC4E6; transform: scale(1.12); }
  .radial-btn:hover .radial-label { color: #fff; }
  @keyframes radialPop {
    from { transform: scale(0); opacity: 0; }
    to   { transform: scale(1); opacity: 1; }
  }
  .radial-icon  { font-size: 1.5rem; line-height: 1; }
  .radial-label { font-size: 0.65rem; font-weight: 800; color: #3A6699; letter-spacing: .02em; }
  .radial-center {
    position: fixed;
    left:  calc(50% - 24px);
    top:   calc(50% - 24px);
    width: 48px; height: 48px;
    border: none; border-radius: 50%;
    background: rgba(111,168,220,.92);
    color: #fff;
    font-size: 1rem; font-weight: 800;
    cursor: pointer;
    box-shadow: 0 4px 20px rgba(46,58,71,.18);
    pointer-events: all;
    animation: radialPop .2s cubic-bezier(.34,1.56,.64,1) both;
    transition: background .12s, transform .1s;
    z-index: 21;
  }
  .radial-center:hover { background: #4F8BC9; transform: scale(1.08); }

  /* ── Interaction card ────────────────────────────────────────────── */
  .card-overlay {
    position: fixed; inset: 0;
    background: rgba(30, 45, 60, 0.30);
    backdrop-filter: blur(4px);
    z-index: 30;
    display: flex; align-items: center; justify-content: center;
  }
  .card {
    background: #fff;
    border-radius: 28px;
    padding: 32px 28px;
    max-width: 340px; width: calc(100vw - 48px);
    box-shadow: 0 28px 80px rgba(46,58,71,.22);
    text-align: center;
    animation: cardIn .22s cubic-bezier(.34,1.56,.64,1) both;
  }
  @keyframes cardIn {
    from { transform: scale(.88); opacity: 0; }
    to   { transform: scale(1);   opacity: 1; }
  }
  .card-icon { font-size: 2.8rem; margin-bottom: 8px; }
  .card h2   { margin: 0 0 10px; font-size: 1.1rem; color: #2E3A47; }
  .card-desc { margin: 0 0 8px; color: #4a6276; line-height: 1.6; font-size: 0.95rem; }
  .card-hint { margin: 0 0 24px; color: #8aa6bf; font-size: 0.85rem; font-style: italic; }
  .card-actions { display: flex; gap: 10px; }
  .card-btn {
    flex: 1; border: none; border-radius: 16px;
    padding: 13px 0; font-family: 'Nunito', sans-serif;
    font-weight: 800; font-size: 0.9rem; cursor: pointer;
    transition: background .12s, transform .1s;
  }
  .card-btn:active { transform: scale(.96); }
  .card-btn.primary { background: #6FA8DC; color: #fff; }
  .card-btn.primary:hover { background: #4F8BC9; }
  .card-btn.ghost   { background: #e4eef8; color: #3A6699; }
  .card-btn.ghost:hover { background: #d3e4f4; }

  /* ── Thought bubble ──────────────────────────────────────────────── */
  .thought-wrap {
    position: fixed;
    top: 30%;
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
    border: 2px solid rgba(111,168,220,.30);
    border-radius: 22px 22px 22px 6px;
    padding: 14px 18px;
    box-shadow: 0 8px 36px rgba(46,58,71,.14);
    position: relative;
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
  .thought-reply { display: flex; gap: 8px; align-items: flex-end; }
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