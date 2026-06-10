<script>
  //@ts-nocheck
  import { onMount, createEventDispatcher } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';

  const dispatch = createEventDispatcher();
  let canvasRef;

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
      id: 'fly',
      icon: '🦋',
      label: 'Fly',
      title: 'Make it fly around',
      description: 'Double-tap the bee\'s wings to send it looping through the garden.',
      hint: 'Click the wing area twice quickly.',
      angle: -30
    },
    {
      id: 'anger',
      icon: '🍯',
      label: 'Honeycomb',
      title: 'Touch the honeycomb',
      description: 'Click the golden honeycomb hanging in the scene to trigger a buzzing rage.',
      hint: 'The honeycomb is visible in the upper right of the scene.',
      angle: 30
    },
    {
      id: 'pollen',
      icon: '🌸',
      label: 'Pollen',
      title: 'Gather pollen',
      description: 'Click any of the flowers scattered around the garden to send the bee collecting.',
      hint: 'Tap the flowers in the foreground.',
      angle: 90
    },
    {
      id: 'chat',
      icon: '💬',
      label: 'Chat',
      title: 'Speak to the bee',
      description: 'The bee has thoughts and wisdom about effort, rest, and how others see you.',
      hint: 'Reply to the bee\'s thought bubbles below.',
      angle: 150
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
  let flyAroundFn;
  let triggerAngerFn;
  let gatherPollenFn;
  let talkingFn;
  let wingTapCount = 0;
  let wingTapTimer = null;

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
    if (activeCard.id === 'fly') flyAroundFn?.();
    if (activeCard.id === 'anger') triggerAngerFn?.();
    if (activeCard.id === 'pollen') gatherPollenFn?.();
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

  onMount(() => {
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

    const envGroup = new Zdog.Anchor({ addTo: scene });

    // Sky gradient panels
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: -1000 }, { x: 2000, y: -1000 }, { x: 2000, y: 0 }, { x: -2000, y: 0 }],
      stroke: 0, fill: true, color: '#FFB7D5', translate: { z: -800 }
    });
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: 0 }, { x: 2000, y: 0 }, { x: 2000, y: 600 }, { x: -2000, y: 600 }],
      stroke: 0, fill: true, color: '#FFE1F0', translate: { z: -800 }
    });

    function addHill(x, w, col, z, amp = 160) {
      const pts = [{ x: x - w / 2, y: 300 }];
      for (let i = 0; i <= 24; i++) {
        const t = i / 24;
        pts.push({ x: x - w / 2 + t * w, y: 300 - Math.sin(t * Math.PI) * amp });
      }
      pts.push({ x: x + w / 2, y: 300 });
      new Zdog.Shape({ addTo: envGroup, path: pts, stroke: 0, fill: true, color: col, translate: { z } });
    }

    addHill(-600, 900, color.darkGreen, -700, 160);
    addHill(500,  800, '#4a7020',       -700, 160);
    addHill(-200, 700, '#5a8a28',       -680, 150);
    addHill(900,  700, '#3d6a18',       -720, 170);

    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: 220 }, { x: 2000, y: 220 }, { x: 2000, y: 800 }, { x: -2000, y: 800 }],
      stroke: 0, fill: true, color: color.land, translate: { z: -500 }
    });
    new Zdog.Shape({
      addTo: envGroup,
      path: [{ x: -2000, y: 220 }, { x: 2000, y: 220 }, { x: 2000, y: 260 }, { x: -2000, y: 260 }],
      stroke: 0, fill: true, color: color.landLight, translate: { z: -490 }
    });

    // Clouds
    [
      { x: -420, y: -280, z: -420, s: 1.2 },
      { x: -160, y: -310, z: -510, s: 0.9 },
      { x:  360, y: -220, z: -380, s: 1.4 },
      { x:   80, y: -290, z: -460, s: 1.0 },
      { x:  700, y: -260, z: -500, s: 1.1 }
    ].forEach(p => {
      const c = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: c, stroke: 95, color: color.cloudPink });
      new Zdog.Shape({ addTo: c, stroke: 72, color: color.cloudWhite, translate: { x: -55, y: 10 } });
      new Zdog.Shape({ addTo: c, stroke: 78, color: color.cloudPink,  translate: { x:  55, y:  5 } });
      new Zdog.Shape({ addTo: c, stroke: 62, color: color.cloudWhite, translate: { x: -95, y: 15 } });
      new Zdog.Shape({ addTo: c, stroke: 58, color: color.cloudPink,  translate: { x: 100, y: 12 } });
    });

    // Trees
    [
      { x: -560, y: 140, z: -320, s: 1.0 },
      { x: -380, y: 160, z: -260, s: 0.8 },
      { x:  460, y: 150, z: -370, s: 1.2 },
      { x:  580, y: 170, z: -220, s: 0.85 },
      { x:  740, y: 145, z: -300, s: 0.9 },
      { x: -680, y: 155, z: -340, s: 1.0 }
    ].forEach(p => {
      const t = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: t, path: [{ y: 0 }, { y: -90 }], stroke: 18, color: color.trunkBrown });
      new Zdog.Shape({ addTo: t, stroke: 130, color: color.darkGreen, translate: { y: -130 } });
      new Zdog.Shape({ addTo: t, stroke: 100, color: color.leafGreen, translate: { y: -175, x: -20 } });
      new Zdog.Shape({ addTo: t, stroke: 90, color: '#7fcf38', translate: { y: -185, x: 20 } });
    });

    // ─── Flower helpers ────────────────────────────────────────────────
    const fgGroup = new Zdog.Anchor({ addTo: scene, translate: { x: 0, y: 50, z: 120 } });

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

    // ─── Honeycomb – placed prominently in the scene ───────────────────
    const honeycombGroup = new Zdog.Anchor({ addTo: fgGroup, translate: { x: -310, y: -180, z: -60 } });

    // Hanging string
    new Zdog.Shape({
      addTo: honeycombGroup,
      path: [{ x: 0, y: -80 }, { x: 0, y: -30 }],
      stroke: 5, color: color.trunkBrown
    });

    // Outer drip for honey feel
    new Zdog.Shape({
      addTo: honeycombGroup,
      stroke: 14, color: '#E8A800',
      translate: { x: 12, y: 46 }
    });

    const hexCoords = [
      { x: 0,    y: 0   },
      { x: 0,    y: -38 },
      { x: -33,  y: -19 },
      { x:  33,  y: -19 },
      { x: -33,  y:  19 },
      { x:  33,  y:  19 },
      { x: 0,    y:  38 }
    ];
    hexCoords.forEach((pos, i) => {
      new Zdog.Polygon({
        addTo: honeycombGroup,
        sides: 6,
        radius: 17,
        fill: true,
        stroke: 9,
        color: i % 2 === 0 ? color.honeyGold : '#EDAB20',
        translate: { x: pos.x, y: pos.y, z: i % 2 === 0 ? 5 : 0 },
        rotate: { z: TAU / 12 }
      });
      // Cell inner shading
      new Zdog.Polygon({
        addTo: honeycombGroup,
        sides: 6,
        radius: 10,
        fill: true,
        stroke: 0,
        color: '#D4920A',
        translate: { x: pos.x, y: pos.y, z: i % 2 === 0 ? 8 : 3 },
        rotate: { z: TAU / 12 }
      });
    });

    // ─── Scatter flowers ───────────────────────────────────────────────
    const flowers = [
      // Far left edge
      { f: 'd', x: -750, y: 140, z:  20, s: 4.8, len: 120, rx: -TAU/4,   ry:  0.3,  c: color.daisyWhite },
      { f: 't', x: -680, y: 155, z:  60, s: 5.0, len: 115, rx: -TAU/4.2, ry: -0.2,  c: color.tulipPink },
      { f: 'r', x: -620, y: 130, z: -40, s: 4.6, len: 110, rx: -TAU/3.8, ry:  0.1  },
      { f: 's', x: -580, y: 165, z:  80, s: 4.5, len: 105, rx: -TAU/3.6, ry: -0.3  },
      { f: 'l', x: -530, y: 145, z: 100, s: 5.2, len: 170, rx: -TAU/5,   ry: -0.5  },
      { f: 'd', x: -480, y: 150, z:  30, s: 4.7, len: 125, rx: -TAU/4,   ry:  0.2,  c: '#FFE4F0' },
      { f: 't', x: -420, y: 135, z: -80, s: 5.3, len: 120, rx: -TAU/4.3, ry:  0.4,  c: color.tulipOrange },
      
      // Left side
      { f: 'r', x: -360, y: 140, z:  50, s: 4.8, len: 115, rx: -TAU/4.4, ry: -0.2  },
      { f: 'd', x: -310, y: 125, z: -100, s: 4.9, len: 130, rx: -TAU/3.8, ry:  0.15, c: '#fff0fb' },
      { f: 's', x: -260, y: 155, z:  70, s: 4.4, len: 110, rx: -TAU/3.5, ry:  0.25 },
      { f: 'l', x: -200, y: 140, z:  45, s: 5.1, len: 165, rx: -TAU/4.8, ry: -0.35 },
      { f: 't', x: -140, y: 160, z: 120, s: 5.0, len: 100, rx: -TAU/4.5, ry:  0.3,  c: '#ff85a2' },
      
      // Center-left
      { f: 'd', x:  -80, y: 145, z:  80, s: 4.6, len: 115, rx: -TAU/4,   ry:  0.1,  c: color.daisyWhite },
      { f: 'r', x:  -20, y: 130, z: -50, s: 4.7, len: 120, rx: -TAU/3.9, ry: -0.1  },
      
      // Center (sparse)
      { f: 's', x:   60, y: 155, z:  90, s: 4.5, len: 105, rx: -TAU/3.7, ry:  0.2  },
      { f: 'd', x:  140, y: 140, z:  40, s: 5.0, len: 125, rx: -TAU/4,   ry: -0.25, c: '#FFE4F0' },
      
      // Center-right
      { f: 't', x:  210, y: 150, z: 110, s: 5.2, len: 115, rx: -TAU/4.2, ry:  0.35, c: color.tulipPink },
      { f: 'l', x:  280, y: 135, z:  60, s: 4.8, len: 160, rx: -TAU/5,   ry: -0.4  },
      
      // Right side
      { f: 'd', x:  350, y: 160, z:  75, s: 4.9, len: 120, rx: -TAU/4,   ry:  0.2,  c: '#fff0fb' },
      { f: 'r', x:  410, y: 140, z: -70, s: 4.6, len: 125, rx: -TAU/3.8, ry:  0.15 },
      { f: 's', x:  470, y: 155, z:  85, s: 4.7, len: 110, rx: -TAU/3.5, ry: -0.2  },
      { f: 't', x:  540, y: 130, z:  30, s: 5.1, len: 100, rx: -TAU/4.5, ry:  0.4,  c: color.tulipOrange },
      { f: 'l', x:  600, y: 145, z: 120, s: 5.0, len: 175, rx: -TAU/4.8, ry: -0.3  },
      
      // Far right edge
      { f: 'd', x:  660, y: 150, z:  50, s: 4.8, len: 130, rx: -TAU/4,   ry:  0.25, c: '#FFE4F0' },
      { f: 'r', x:  720, y: 140, z: -90, s: 4.9, len: 115, rx: -TAU/3.9, ry: -0.15 },
      { f: 's', x:  780, y: 165, z:  95, s: 4.5, len: 105, rx: -TAU/3.6, ry:  0.1  },
      { f: 't', x:  840, y: 135, z:  40, s: 5.3, len: 120, rx: -TAU/4.3, ry:  0.3,  c: color.tulipPink },
      { f: 'd', x:  900, y: 155, z: 110, s: 4.7, len: 125, rx: -TAU/4.1, ry: -0.2,  c: '#fff0fb' },
      { f: 'l', x:  950, y: 140, z:  60, s: 5.2, len: 170, rx: -TAU/5,   ry: -0.4  }
    ];

    flowers.forEach(d => {
      if      (d.f === 'd') createDaisy(fgGroup,    d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
      else if (d.f === 'r') createRose(fgGroup,     d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 'l') createLavender(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 's') createSunflower(fgGroup,d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 't') createTulip(fgGroup,    d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
    });

    // ─── Bee ───────────────────────────────────────────────────────────
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

    // ─── Expression system ────────────────────────────────────────────
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

    // ─── Interaction animations ────────────────────────────────────────
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

    function flyAround() {
      if (isGathering) return;
      isGathering = true;
      setExpression('excited');
      const tl = gsap.timeline({
        onComplete: () => { setExpression('neutral'); isGathering = false; }
      });
      // Flying in full 3D with rotations to show side and back views
      tl.to(BEE_CONT.translate, { duration: 0.4, x: -200, y: -150, z: 80, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.4, x: 0.3, y: -0.6, z: 0.5, ease: 'power2.inOut' }, 0);
      
      tl.to(BEE_CONT.translate, { duration: 0.35, x: 0, y: -220, z: 120, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.35, x: 0.25, y: -TAU/8, z: 0.3, ease: 'power2.inOut' }, '<');
      
      tl.to(BEE_CONT.translate, { duration: 0.35, x: 200, y: -150, z: 80, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.35, x: 0.3, y: 0.6, z: -0.5, ease: 'power2.inOut' }, '<');
      
      tl.to(BEE_CONT.translate, { duration: 0.35, x: 0, y: -80, z: 20, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.35, x: 0.35, y: TAU/6, z: 0.8, ease: 'power2.inOut' }, '<');
      
      tl.to(BEE_CONT.translate, { duration: 0.35, x: 0, y: -100, z: 0, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.5, x: 0, y: 0, z: 0, ease: 'power2.inOut' }, '<');
    }

    function gatherPollen() {
      if (isGathering) return;
      isGathering = true;
      setExpression('happy');
      const tl = gsap.timeline({ onComplete: () => { setExpression('neutral'); isGathering = false; } });
      tl.to(BEE_CONT.translate, { duration: 0.5, x: -60, y: 70, z: 5, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.2, x: 0.5, y: 0.4,      ease: 'power2.out'   }, '-=0.2');
      tl.to(leftArm.rotate,     { duration: 0.2, z: 0.2, x: 0.1,      ease: 'back.out(1.5)'}, '-=0.1');
      tl.to(rightArm.rotate,    { duration: 0.2, z: -0.2, x: 0.1,     ease: 'back.out(1.5)'}, '<');
      tl.to(BEE_CONT.translate, { duration: 0.12, y: 80,              ease: 'power1.inOut' });
      tl.to(BEE_CONT.translate, { duration: 0.12, y: 65,              ease: 'power1.inOut' });
      tl.to(leftArm.rotate,     { duration: 0.2, z: 0, x: 0,          ease: 'power2.in' });
      tl.to(rightArm.rotate,    { duration: 0.2, z: 0, x: 0,          ease: 'power2.in' }, '<');
      tl.to(BEE_CONT.translate, { duration: 0.55, x: 0, y: -100, z: 0, ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.4, x: 0, y: 0,          ease: 'power2.inOut' }, '-=0.4');
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
    flyAroundFn        = flyAround;
    gatherPollenFn     = gatherPollen;
    talkingFn          = talkingAnimation;
    triggerAngerFn     = () => {
      showThought('You touched my honeycomb! Bzzzz! 🍯');
      triggerAnnoyance();
    };

    // ─── Input detection zones ────────────────────────────────────────
    const honeycombZone  = { x: -310, y: -130, r: 80 };
    const flowerZones = [
      { x: -280, y: 120, r: 70 }, { x: -140, y: 130, r: 70 },
      { x:   60, y: 140, r: 70 }, { x:  220, y: 125, r: 70 },
      { x: -360, y: 150, r: 70 }, { x: -220, y: 145, r: 70 }
    ];

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

      // Honeycomb click
      if (Math.hypot(cx - honeycombZone.x, cy - honeycombZone.y) < honeycombZone.r) {
        triggerAngerFn(); return;
      }

      // Flower clicks
      for (const fz of flowerZones) {
        if (Math.hypot(cx - fz.x, cy - fz.y) < fz.r) {
          gatherPollen(); return;
        }
      }

      // Wing double-tap
      const wingLeft  = Math.hypot(cx - (-40), cy - (-130)) < 90;
      const wingRight = Math.hypot(cx - ( 40), cy - (-130)) < 90;
      if (wingLeft || wingRight) {
        wingTapCount++;
        if (wingTapTimer) clearTimeout(wingTapTimer);
        if (wingTapCount >= 2) { wingTapCount = 0; flyAround(); }
        else wingTapTimer = setTimeout(() => { wingTapCount = 0; }, 700);
        return;
      }

      // Bee body click → radial menu
      if (Math.hypot(cx, cy + 100) < 130) {
        radialMenuOpen = !radialMenuOpen;
        activeCard = null;
      }
    }

    canvasRef.addEventListener('click',       onCanvasClick);
    canvasRef.addEventListener('pointerdown', onPointerDown);
    canvasRef.addEventListener('pointermove', onPointerMove);
    window.addEventListener('pointerup',      onPointerUp);

    // First thought after 3 s
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

<!-- ─── Top bar ──────────────────────────────────────────────────────── -->
<div class="top-bar">
  <button class="bar-btn" on:click={goHome}>← Home</button>
  <button class="bar-btn" on:click={toggleHistory}>
    {historyOpen ? 'Close history' : '💬 History'}
  </button>
</div>

<!-- ─── Chat history panel ─────────────────────────────────────────── -->
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

<!-- ─── Canvas ─────────────────────────────────────────────────────── -->
<canvas bind:this={canvasRef} class="scene"></canvas>

<!-- ─── Radial menu ────────────────────────────────────────────────── -->
{#if radialMenuOpen}
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
    <!-- Centre close -->
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

<!-- ─── Hint ───────────────────────────────────────────────────────── -->
<p class="hint">Tap the bee to open interactions ✿</p>

<div class="credit">Made by MakMin</div>

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
    display: block; z-index: 0;
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
    font-weight: 600;
  }

  .thought-reply {
    display: flex; gap: 8px; align-items: flex-end;
  }
  .thought-reply textarea {
    flex: 1;
    background: rgba(255,255,255,0.96);
    border: 1.5px solid rgba(240,74,111,.35);
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
  .thought-reply textarea::placeholder { color: #c090a0; }
  .thought-reply textarea:focus {
    border-color: #F04A6F;
    box-shadow: 0 0 0 3px rgba(240,74,111,.15);
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

  .credit {
    position: fixed; bottom: 8px; left: 14px;
    font-size: 0.7rem; color: rgba(90,26,48,.4);
    z-index: 10; pointer-events: none;
  }
</style>