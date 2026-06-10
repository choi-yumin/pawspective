<script>
  //@ts-nocheck
  import { onMount } from 'svelte';
  import Zdog from 'zdog';
  import { gsap } from 'gsap';

  let canvasRef;

  // Pulls your key safely from Svelte's import.meta.env system
  const OPENAI_API_KEY = import.meta.env.VITE_OPENAI_API_KEY; 

  onMount(() => {
    if (!canvasRef) return;

    const TAU = Zdog.TAU;

    const color = {
      beeYellow:'#FCD116', beeBlack:'#3D2620', beeWhite:'#FFFDF0', beeCheek:'#F26B50',
      leafGreen:'#5DA020', darkGreen:'#3A5C18', land:'#7ab535', landLight:'#9fd95c',
      trunkBrown:'#8B6340', cloudPink:'#FFE4F0', cloudWhite:'#FFF7FB',
      daisyWhite:'#FFFFFF', daisyYellow:'#F9D342',
      roseRed:'#F04A6F', rosePink:'#FF8FAE',
      lavPurple:'#C07EE0', lavLight:'#E2B8F5',
      sunflowerYellow:'#FFB800', sunflowerCenter:'#7A3B10',
      tulipPink:'#FF85A2', tulipOrange:'#FF6B35',
      beeAnnoyedRed: '#FF3B30'
    };

    const scene = new Zdog.Illustration({
      element: canvasRef,
      dragRotate: false,
      resize: 'window',
      rotate: { x: -0.32, y: -0.18, z: 0 },
    });

    const SCENE_OFFSET_X = -180;
    const envGroup = new Zdog.Anchor({ addTo: scene });

    new Zdog.Shape({
      addTo: envGroup,
      path: [{x:-2000,y:-1000},{x:2000,y:-1000},{x:2000,y:0},{x:-2000,y:0}],
      stroke: 0, fill: true, color: '#FFB7D5', translate: { z: -800 }
    });
    new Zdog.Shape({
      addTo: envGroup,
      path: [{x:-2000,y:0},{x:2000,y:0},{x:2000,y:600},{x:-2000,y:600}],
      stroke: 0, fill: true, color: '#FFE1F0', translate: { z: -800 }
    });

    function addHill(x, w, col, z, amp = 160) {
      const pts = [{x: x - w / 2, y: 300}];
      for (let i = 0; i <= 24; i++) {
        const t = i / 24;
        pts.push({ x: x - w/2 + t*w, y: 300 - Math.sin(t * Math.PI) * amp });
      }
      pts.push({x: x + w / 2, y: 300});
      new Zdog.Shape({ addTo: envGroup, path: pts, stroke: 0, fill: true, color: col, translate: { z } });
    }

    addHill(-600, 900, color.darkGreen, -700, 160);
    addHill(500,  800, '#4a7020',       -700, 160);
    addHill(-200, 700, '#5a8a28',       -680, 150);
    addHill(900,  700, '#3d6a18',       -720, 170);

    new Zdog.Shape({
      addTo: envGroup,
      path: [{x:-2000,y:220},{x:2000,y:220},{x:2000,y:800},{x:-2000,y:800}],
      stroke: 0, fill: true, color: color.land, translate: { z: -500 }
    });
    new Zdog.Shape({
      addTo: envGroup,
      path: [{x:-2000,y:220},{x:2000,y:220},{x:2000,y:260},{x:-2000,y:260}],
      stroke: 0, fill: true, color: color.landLight, translate: { z: -490 }
    });

    // Clouds
    [
      {x:-420,y:-280,z:-420,s:1.2},
      {x:-160,y:-310,z:-510,s:0.9},
      {x: 360,y:-220,z:-380,s:1.4},
      {x:  80,y:-290,z:-460,s:1.0},
      {x: 700,y:-260,z:-500,s:1.1},
    ].forEach(p => {
      let c = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: c, stroke: 95, color: color.cloudPink });
      new Zdog.Shape({ addTo: c, stroke: 72, color: color.cloudWhite, translate: { x: -55, y: 10 } });
      new Zdog.Shape({ addTo: c, stroke: 78, color: color.cloudPink, translate: { x: 55, y: 5 } });
      new Zdog.Shape({ addTo: c, stroke: 62, color: color.cloudWhite, translate: { x: -95, y: 15 } });
      new Zdog.Shape({ addTo: c, stroke: 58, color: color.cloudPink, translate: { x: 100, y: 12 } });
    });

    // Trees
    [
      {x:-460,y:140,z:-320,s:1.0},
      {x:-310,y:160,z:-260,s:0.8},
      {x: 390,y:150,z:-370,s:1.2},
      {x: 520,y:170,z:-220,s:0.85},
      {x: 720,y:145,z:-300,s:0.9},
      {x: 860,y:165,z:-340,s:1.0},
    ].forEach(p => {
      let t = new Zdog.Anchor({ addTo: envGroup, translate: { x: p.x, y: p.y, z: p.z }, scale: p.s });
      new Zdog.Shape({ addTo: t, path: [{y:0},{y:-90}], stroke: 18, color: color.trunkBrown });
      new Zdog.Shape({ addTo: t, stroke: 130, color: color.darkGreen, translate: { y: -130 } });
      new Zdog.Shape({ addTo: t, stroke: 100, color: color.leafGreen, translate: { y: -175, x: -20 } });
      new Zdog.Shape({ addTo: t, stroke: 90, color: '#7fcf38', translate: { y: -185, x: 20 } });
    });

    const fgGroup = new Zdog.Anchor({ addTo: scene, translate: { x: SCENE_OFFSET_X, y: 50, z: 120 } });
    const rgGroup = new Zdog.Anchor({ addTo: scene, translate: { x: 260, y: 50, z: 120 } });

    function createDaisy(parent, x, y, z, scale, stemLen, rotX, rotY, petalCol) {
      petalCol = petalCol || color.daisyWhite;
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (120 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{x:0,y:stemLen,z:0},{x:0,y:0,z:0}], stroke: 7, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 13; i++) {
        let ph = new Zdog.Anchor({ addTo: hd, rotate: { z: (TAU / 13) * i } });
        new Zdog.Ellipse({ addTo: ph, width: 11, height: 30, fill: true, color: petalCol, stroke: 0, translate: { y: -18 } });
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 15, fill: true, stroke: 4, color: color.daisyYellow, translate: { z: 1.5 } });
    }

    function createRose(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (140 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{x:0,y:stemLen},{x:0,y:0}], stroke: 7, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let l = 0; l < 3; l++) {
        const pc = 6 + l * 2, r = 10 + l * 9;
        for (let i = 0; i < pc; i++) {
          const a = (TAU / pc) * i + l * 0.3;
          const ph = new Zdog.Anchor({ addTo: hd, translate: { x: Math.cos(a) * r, y: Math.sin(a) * r } });
          new Zdog.Ellipse({ addTo: ph, width: l===0?8:10+l*2, height: l===0?12:14+l*3, fill: true, color: l===0?color.roseRed:color.rosePink, stroke: 0, rotate: { z: a } });
        }
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 8, fill: true, stroke: 0, color: '#c0243a', translate: { z: 1 } });
    }

    function createLavender(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (150 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{x:0,y:stemLen},{x:0,y:0}], stroke: 6, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 8; i++) {
        const ph = new Zdog.Anchor({ addTo: hd, translate: { x: 0, y: -i * 10 }, rotate: { z: i % 2 === 0 ? 0.3 : -0.3 } });
        new Zdog.Ellipse({ addTo: ph, width: 9, height: 14, fill: true, color: i<3?color.lavPurple:color.lavLight, stroke: 0, translate: { x: 6 } });
        new Zdog.Ellipse({ addTo: ph, width: 9, height: 14, fill: true, color: i<3?color.lavPurple:color.lavLight, stroke: 0, translate: { x: -6 } });
      }
    }

    function createSunflower(parent, x, y, z, scale, stemLen, rotX, rotY) {
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (160 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{x:0,y:stemLen},{x:0,y:0}], stroke: 9, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 18; i++) {
        let ph = new Zdog.Anchor({ addTo: hd, rotate: { z: (TAU / 18) * i } });
        new Zdog.Ellipse({ addTo: ph, width: 13, height: 36, fill: true, color: color.sunflowerYellow, stroke: 0, translate: { y: -24 } });
      }
      new Zdog.Ellipse({ addTo: hd, diameter: 22, fill: true, stroke: 5, color: color.sunflowerCenter, translate: { z: 2 } });
    }

    function createTulip(parent, x, y, z, scale, stemLen, rotX, rotY, col) {
      col = col || color.tulipPink;
      const fg = new Zdog.Anchor({ addTo: parent, translate: { x, y: y + (130 - stemLen), z }, scale });
      new Zdog.Shape({ addTo: fg, path: [{x:0,y:stemLen},{x:0,y:0}], stroke: 8, color: color.leafGreen });
      const hd = new Zdog.Anchor({ addTo: fg, rotate: { x: rotX, y: rotY } });
      for (let i = 0; i < 6; i++) {
        const a = (TAU / 6) * i;
        new Zdog.Ellipse({ addTo: hd, width: 18, height: 36, fill: true, color: col, stroke: 0, translate: { x: Math.cos(a)*10, z: Math.sin(a)*10 }, rotate: { y: a, x: -0.3 } });
      }
    }

    createDaisy(fgGroup, 0, 100, 10, 5, 120, -TAU / 4, 0, color.daisyWhite);

    const fd = [
      {f:'r',x:-55, y:135,z:-80, s:5.5, len:160, rx:-TAU/5,   ry:0.3},
      {f:'s',x:115, y:210,z:120, s:4.2, len:110, rx:-TAU/3.5, ry:-0.4},
      {f:'d',x:-195,y:115,z:-180,s:5,   len:150, rx:-TAU/4.5, ry:0.1,  c:'#ffe4ef'},
      {f:'t',x:215, y:165,z:-100,s:7,   len:100, rx:-TAU/4,   ry:-0.2, c:color.tulipPink},
      {f:'l',x:-110,y:125,z:60,  s:4.8, len:170, rx:-TAU/5,   ry:-0.5},
      {f:'d',x:90,  y:155,z:-140,s:4.5, len:130, rx:-TAU/3.8, ry:0.2,  c:'#ffe9f5'},
      {f:'t',x:-280,y:160,z:90,  s:5.5, len:145, rx:-TAU/4.2, ry:0.5,  c:color.tulipOrange},
      {f:'d',x:280, y:145,z:50,  s:5,   len:155, rx:-TAU/4,   ry:-0.3, c:color.daisyWhite},
      {f:'r',x:350, y:185,z:-110,s:4.8, len:115, rx:-TAU/3.2, ry:0.1},
      {f:'s',x:-340,y:135,z:-30, s:4.5, len:180, rx:-TAU/4.8, ry:-0.2},
      {f:'l',x:160, y:195,z:100, s:5,   len:125, rx:-TAU/4,   ry:0.4},
      {f:'t',x:-160,y:150,z:140, s:5.5, len:165, rx:-TAU/5.2, ry:-0.1, c:'#ff85a2'},
      {f:'d',x:45,  y:205,z:170, s:4.5, len:95,  rx:-TAU/3.4, ry:0.3,  c:'#fff0fb'},
      {f:'r',x:-260,y:140,z:-130,s:4.2, len:175, rx:-TAU/4.5, ry:-0.4},
      {f:'d',x:420, y:170,z:0,   s:4,   len:140, rx:-TAU/4,   ry:0.2,  c:'#fff'},
      {f:'s',x:-450,y:210,z:30,  s:4,   len:105, rx:-TAU/3.6, ry:-0.3},
    ];

    fd.forEach(d => {
      if      (d.f === 'd') createDaisy(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
      else if (d.f === 'r') createRose(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 'l') createLavender(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 's') createSunflower(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 't') createTulip(fgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
    });

    const rd = [
      {f:'d', x:-80,  y:210, z:160,  s:4.5, len:100, rx:-TAU/3.5, ry:0.2,  c:color.daisyWhite},
      {f:'t', x:60,   y:190, z:130,  s:5,   len:115, rx:-TAU/4,   ry:-0.3, c:color.tulipPink},
      {f:'r', x:-160, y:200, z:80,   s:4.8, len:110, rx:-TAU/3.8, ry:0.4},
      {f:'l', x:160,  y:185, z:90,   s:4.5, len:125, rx:-TAU/4.2, ry:-0.2},
      {f:'s', x:260,  y:215, z:50,   s:4.2, len:100, rx:-TAU/3.6, ry:0.1},
      {f:'d', x:340,  y:195, z:110,  s:4,   len:130, rx:-TAU/4,   ry:0.3,  c:'#ffe9f5'},
      {f:'t', x:-240, y:155, z:40,   s:5.2, len:155, rx:-TAU/5,   ry:0.5,  c:color.tulipOrange},
      {f:'r', x:300,  y:140, z:-60,  s:4.6, len:160, rx:-TAU/4.5, ry:-0.4},
      {f:'d', x:400,  y:165, z:20,   s:4.3, len:140, rx:-TAU/4,   ry:0.1,  c:'#fff0fb'},
      {f:'l', x:-300, y:130, z:-80,  s:4.8, len:170, rx:-TAU/5.2, ry:-0.3},
      {f:'s', x:180,  y:145, z:-120, s:4,   len:145, rx:-TAU/3.8, ry:0.2},
      {f:'d', x:-200, y:110, z:-150, s:4.2, len:150, rx:-TAU/4.8, ry:0.4,  c:color.daisyWhite},
      {f:'r', x:240,  y:120, z:-180, s:4.5, len:165, rx:-TAU/4.2, ry:-0.1},
      {f:'t', x:360,  y:135, z:-100, s:5,   len:135, rx:-TAU/4,   ry:0.3,  c:'#ff85a2'},
      {f:'l', x:-100, y:145, z:200,  s:5.2, len:120, rx:-TAU/3.6, ry:-0.5},
      {f:'d', x:100,  y:160, z:220,  s:4.5, len:105, rx:-TAU/3.4, ry:0.2,  c:'#fff'},
    ];

    rd.forEach(d => {
      if      (d.f === 'd') createDaisy(rgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
      else if (d.f === 'r') createRose(rgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 'l') createLavender(rgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 's') createSunflower(rgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry);
      else if (d.f === 't') createTulip(rgGroup, d.x, d.y, d.z, d.s, d.len, d.rx, d.ry, d.c);
    });

    let BEE_CONT = new Zdog.Anchor({ addTo: fgGroup, translate: { x: 0, y: -130, z: 0 }, scale: 1.45 });
    let BEE_PIVOT = new Zdog.Anchor({ addTo: BEE_CONT, translate: { x: 0, y: 32, z: -35 }, rotate: { y: 0 } });
    let BEE = new Zdog.Anchor({ addTo: BEE_PIVOT, translate: { x: 0, y: -32, z: 35 } });

    let beeHead = new Zdog.Shape({ addTo: BEE, stroke: 135, color: color.beeYellow });
    new Zdog.Shape({ addTo: beeHead, stroke: 10, color: color.beeBlack, translate: { x: -22, y: 4, z: 36 } });
    new Zdog.Shape({ addTo: beeHead, stroke: 10, color: color.beeBlack, translate: { x: 14,  y: 4, z: 40 } });
    
    // Store references to the mouth path vector to manipulate it directly
    let beeMouth = new Zdog.Shape({
      addTo: beeHead, stroke: 3.5, color: color.beeBlack, closed: false,
      path: [{x:-4,y:10,z:42},{bezier:[{x:-2,y:13,z:42},{x:2,y:13,z:42},{x:4,y:10,z:42}]}]
    });

    let rc = new Zdog.Shape({ addTo: beeHead, stroke: 14, color: color.beeCheek, translate: { x: -32, y: 14, z: 24 } });
    rc.copy({ translate: { x: 24, y: 14, z: 30 } });

    let antlerAnchor = new Zdog.Anchor({ addTo: beeHead, translate: { y: -44, z: 10 } });
    new Zdog.Shape({ addTo: antlerAnchor, path: [{y:0,x:-10},{y:-22,x:-24,z:8}], stroke: 4.5, color: color.beeBlack });
    new Zdog.Shape({ addTo: antlerAnchor, path: [{y:0,x:10},{y:-22,x:-4,z:12}], stroke: 4.5, color: color.beeBlack });

    let leftArm = new Zdog.Anchor({ addTo: BEE, translate: { x: -18, y: 45, z: 32 } });
    new Zdog.Shape({ addTo: leftArm, path: [{y:0},{y:22}], stroke: 7, color: color.beeBlack });

    let rightArm = new Zdog.Anchor({ addTo: BEE, translate: { x: 18, y: 45, z: 32 } });
    new Zdog.Shape({ addTo: rightArm, path: [{y:0},{y:22}], stroke: 7, color: color.beeBlack });

    let bodyAnchor = new Zdog.Anchor({ addTo: BEE, translate: { y: 32, z: -35 } });
    let p1 = new Zdog.Shape({ addTo: bodyAnchor, stroke: 140, color: color.beeYellow });
    let p2 = p1.copy({ addTo: bodyAnchor, stroke: 162, color: color.beeBlack,  translate: { z: -32  } });
    let p3 = p1.copy({ addTo: bodyAnchor, stroke: 168, color: color.beeYellow, translate: { z: -64  } });
    let p4 = p1.copy({ addTo: bodyAnchor, stroke: 156, color: color.beeBlack,  translate: { z: -96  } });
    new Zdog.Shape({ addTo: bodyAnchor, stroke: 108, color: color.beeBlack, translate: { z: -123 } });

    let rightWing = new Zdog.Anchor({ addTo: bodyAnchor, translate: { z: -43, y: -65, x:  29 } });
    let leftWing  = new Zdog.Anchor({ addTo: bodyAnchor, translate: { z: -43, y: -65, x: -29 } });

    new Zdog.Ellipse({ addTo: rightWing, width: 80, height: 160, color: color.beeWhite, fill: true, rotate: { x: TAU/5, z:  TAU/5 }, translate: { x:  65 }, stroke: 0 });
    new Zdog.Ellipse({ addTo: leftWing,  width: 80, height: 160, color: color.beeWhite, fill: true, rotate: { x: TAU/5, z: -TAU/5 }, translate: { x: -65 }, stroke: 0 });

    let frame = 0, isRunning = true, isGathering = false;
    let isAnnoyed = false;

    function render() {
      if (!isRunning) return;
      frame++;
      
      // If annoyed, make wings flap wildly to show agitation
      const wingSpeedModifier = isAnnoyed ? 2.5 : 1.0;
      rightWing.rotate.z = -TAU/6 + (TAU/10) * Math.sin(frame / (0.9 / wingSpeedModifier));
      leftWing.rotate.z  =  TAU/6 - (TAU/10) * Math.sin(frame / (0.9 / wingSpeedModifier));
      
      antlerAnchor.rotate.x = (TAU/40) * Math.sin(frame / 10);
      if (!isGathering) {
        BEE_CONT.translate.y = -130 + Math.sin(frame / 14) * 4;
        BEE_CONT.translate.x = Math.cos(frame / 28) * 3;
      }
      scene.updateRenderGraph();
      requestAnimationFrame(render);
    }
    render();

    let bubbleTimeout = null;

    const bubble     = document.getElementById('chat-bubble');
    const bubbleText = document.getElementById('bubble-text');
    const typingDots = document.getElementById('typing-dots');
    const hint       = document.getElementById('hint');
    const userInput  = document.getElementById('user-input');
    const sendBtn    = document.getElementById('send-btn');

    function showBubble(text) {
      if (bubbleTimeout) clearTimeout(bubbleTimeout);
      typingDots.style.display = 'none';
      bubbleText.style.display = 'block';
      bubbleText.textContent   = text;
      bubble.classList.add('show');
      hint.style.opacity = '0';
      bubbleTimeout = setTimeout(() => {
        bubble.classList.remove('show');
        setTimeout(() => { hint.style.opacity = '0.75'; }, 600);
      }, 9000);
    }

    function showTyping() {
      if (bubbleTimeout) clearTimeout(bubbleTimeout);
      bubbleText.style.display = 'none';
      typingDots.style.display = 'flex';
      bubble.classList.add('show');
    }

    // OpenAI Chat History Structure (role names are 'assistant' and 'user')
    let chatHistory = [
      {
        role: 'assistant',
        content: "Buzz! 🌸 Ask me about effort, rest, or what others think of you!"
      }
    ];

    const SYSTEM_TEXT = `You are Buzz, an adorable wise bee who lives in a flower garden. You speak in short, warm, whimsical messages (2-4 sentences max). You use flower and bee metaphors naturally.
Your specialty is helping people think about:
- How others perceive them (social perception, first impressions, reputation)
- Diligence and hard work (like a bee — constant, purposeful effort)
- Laziness and rest (the importance of pause vs. the trap of avoidance)
You give gentle, insightful, slightly playful advice. You don't lecture. Keep it warm, short, and wise.`;

    let isApiLoading = false;

    async function askBee(userMessage) {
      if (isApiLoading) return;
      isApiLoading = true;
      showTyping();
      gatherPollen();
      
      // Append user prompt cleanly
      chatHistory = [
        ...chatHistory,
        { role: 'user', content: userMessage }
      ];

      try {
        const endpoint = 'https://api.openai.com/v1/chat/completions';

        // Build the OpenAI Chat API structure
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
          console.error("OpenAI Internal Error JSON:", errorData);
          throw new Error(`API returned status ${res.status}`);
        }

        const data = await res.json();
        const replyText = data?.choices?.[0]?.message?.content;

        if (replyText) {
          const cleanReply = replyText.trim();
          chatHistory = [
            ...chatHistory,
            { role: 'assistant', content: cleanReply }
          ];
          showBubble(cleanReply);
        } else {
          showBubble("Sorry, I can't think of anything to answer. Try again?");
        }
      } catch (e) {
        console.error("OpenAI Catch block triggered:", e);
        chatHistory = chatHistory.slice(0, -1);
        showBubble("Sorry, I can't think of anything to answer. Try again?");
      } finally {
        isApiLoading = false;
      }
    }

    function gatherPollen() {
      if (isGathering) return;
      isGathering = true;
      const tl = gsap.timeline({ onComplete: () => { isGathering = false; } });
      tl.to(BEE_CONT.translate, { duration: 0.55, x: 0, y: 80, z: 5,    ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.25, x: 0.5, y: 0.4,        ease: 'power2.out'   }, '-=0.2');
      tl.to(leftArm.rotate,     { duration: 0.2,  z: 0.2,  x: 0.1,       ease: 'back.out(1.5)' }, '-=0.1');
      tl.to(rightArm.rotate,    { duration: 0.2,  z: -0.2, x: 0.1,       ease: 'back.out(1.5)' }, '<');
      tl.to(BEE_CONT.translate, { duration: 0.12, y: 90,                  ease: 'power1.inOut' });
      tl.to(BEE_CONT.translate, { duration: 0.12, y: 75,                  ease: 'power1.inOut' });
      tl.to(leftArm.rotate,     { duration: 0.2,  z: 0, x: 0,            ease: 'power2.in' });
      tl.to(rightArm.rotate,    { duration: 0.2,  z: 0, x: 0,            ease: 'power2.in' }, '<');
      tl.to(BEE_CONT.translate, { duration: 0.6,  x: 0, y: -130, z: 0,  ease: 'power2.inOut' });
      tl.to(BEE_PIVOT.rotate,   { duration: 0.4,  x: 0.4, y: 0.4,        ease: 'power2.inOut' }, '-=0.5');
      tl.set(BEE_PIVOT.rotate, { z: 0 });
    }

    // Annoyance trigger mechanism logic
    function triggerAnnoyance() {
      if (isAnnoyed) return; // Prevent loop overlapping
      isAnnoyed = true;

      // Morph the mouth array points to point upward (frown orientation)
      beeMouth.path = [
        { x: -5, y: 15, z: 42 },
        { bezier: [ { x: -2, y: 8, z: 42 }, { x: 2, y: 8, z: 42 }, { x: 5, y: 15, z: 42 } ] }
      ];
      beeMouth.updatePath();

      // GSAP timeline transitions colors to angry red and then snaps back cleanly
      const annoyTl = gsap.timeline({
        onComplete: () => {
          // Restore original smiling morph points cleanly
          beeMouth.path = [
            { x: -4, y: 10, z: 42 },
            { bezier: [ { x: -2, y: 13, z: 42 }, { x: 2, y: 13, z: 42 }, { x: 4, y: 10, z: 42 } ] }
          ];
          beeMouth.updatePath();
          isAnnoyed = false;
        }
      });

      // Rapidly shift all yellow color fields to red
      annoyTl.to([beeHead, p1, p3], {
        duration: 0.15,
        color: color.beeAnnoyedRed,
        ease: 'power2.out'
      });

      // Keep it angry for a brief moment, then cool down back to original state
      annoyTl.to([beeHead, p1, p3], {
        duration: 0.7,
        color: color.beeYellow,
        ease: 'power1.inOut',
        delay: 0.8
      });
      
      // Inject an annoyed prompt interaction statement contextually
      askBee("Someone just pulled on my tail and body, it made me cross! Tell me off playfully or give me advice on keeping my cool.");
    }

    function handleCanvasClick(event) {
      // Ignore normal clicks if the user was just dragging or pulling the bee around
      if (wasDraggingVector) return;

      const rect   = canvasRef.getBoundingClientRect();
      const clickX = (event.clientX - rect.left  - rect.width  / 2) - SCENE_OFFSET_X;
      const clickY = (event.clientY - rect.top   - rect.height / 2);
      const dx = clickX - BEE_CONT.translate.x;
      const dy = clickY - (BEE_CONT.translate.y + 50);
      if (Math.sqrt(dx * dx + dy * dy) < 120) {
        askBee("Give me a random piece of wisdom about effort, laziness, or how others see me.");
      }
    }

    let isDragging = false, lastX = 0, lastY = 0;
    let wasDraggingVector = false; 
    const sensitivity = 0.008;

    function onPointerDown(e) {
      isDragging = true;
      wasDraggingVector = false;
      lastX = e.clientX; lastY = e.clientY;
      try { canvasRef.setPointerCapture(e.pointerId); } catch (_) {}
    }
    
    function onPointerMove(e) {
      if (!isDragging) return;
      const dx = e.clientX - lastX, dy = e.clientY - lastY;
      lastX = e.clientX; lastY = e.clientY;
      
      // Calculate aggregate drag vector magnitude to classify intentional pulls
      const dragDistance = Math.sqrt(dx * dx + dy * dy);
      if (dragDistance > 12) { 
        wasDraggingVector = true;
        // If they pull hard enough, it triggers the internal annoyance morph system
        if (dragDistance > 28) {
          triggerAnnoyance();
        }
      }

      BEE_PIVOT.rotate.y += dx * sensitivity;
      BEE_PIVOT.rotate.x += dy * sensitivity;
      BEE_PIVOT.rotate.x  = Math.max(-TAU/8, Math.min(TAU/8, BEE_PIVOT.rotate.x));
      scene.updateRenderGraph();
    }
    
    function onPointerUp() { 
      isDragging = false; 
    }

    canvasRef.addEventListener('click',       handleCanvasClick);
    canvasRef.addEventListener('pointerdown', onPointerDown);
    canvasRef.addEventListener('pointermove', onPointerMove);
    
    const handleGlobalPointerUp = (e) => onPointerUp(e);
    window.addEventListener('pointerup',      handleGlobalPointerUp);

    function sendMessage() {
      const msg = userInput.value.trim();
      if (!msg) return;
      userInput.value = '';
      userInput.style.height = '48px';
      askBee(msg);
    }

    sendBtn.addEventListener('click', sendMessage);
    userInput.addEventListener('keydown', e => {
      if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); sendMessage(); }
    });
    userInput.addEventListener('input', () => {
      userInput.style.height = '48px';
      userInput.style.height = Math.min(userInput.scrollHeight, 96) + 'px';
    });

    setTimeout(() => showBubble("Buzz! 🌸 Ask me about effort, rest, or what others think of you!"), 800);

    return () => {
      isRunning = false;
      if (canvasRef) {
        canvasRef.removeEventListener('click',       handleCanvasClick);
        canvasRef.removeEventListener('pointerdown', onPointerDown);
        canvasRef.removeEventListener('pointermove', onPointerMove);
      }
      window.removeEventListener('pointerup',      handleGlobalPointerUp);
    };
  });
</script>

<div class="credit"><p>Made by MakMin</p></div>

<main>
  <canvas bind:this={canvasRef} class="scene"></canvas>

  <div id="chat-bubble">
    <span id="bubble-text"></span>
    <div id="typing-dots"><span></span><span></span><span></span></div>
  </div>

  <p id="hint">Click the bee or type below ✿</p>

  <div id="input-area">
    <textarea id="user-input" placeholder="Ask the bee something…" rows="1"></textarea>
    <button id="send-btn" aria-label="Send">➤</button>
  </div>
</main>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap");

  :global(body) {
    margin: 0; padding: 0;
    display: flex; align-items: center; justify-content: center;
    height: 100vh;
    background: #FFB7D5;
    overflow: hidden;
    position: relative;
    font-family: "Inter", sans-serif;
  }

  main {
    width: 100vw; height: 100vh;
    display: grid; place-items: center;
    position: relative;
  }

  canvas {
    position: fixed; top: 0; left: 0;
    width: 100vw; height: 100vh;
    display: block; z-index: 0;
    touch-action: none;
  }

  :global(#chat-bubble) {
    position: fixed;
    top: 16vh;
    left: 56%;
    max-width: min(300px, 38vw);
    min-width: 120px;
    background: #FF8FAE;
    border: 0.5px solid #D4537E;
    border-radius: 20px 20px 20px 4px;
    padding: 14px 18px;
    font-size: 15px;
    line-height: 1.6;
    color: #4B1528;
    z-index: 10;
    pointer-events: none;
    opacity: 0;
    transition: opacity 0.3s ease;
    word-wrap: break-word;
  }
  :global(#chat-bubble.show) { opacity: 1; }

  :global(#chat-bubble::before) {
    content: '';
    position: absolute;
    top: 20px; left: -10px;
    width: 0; height: 0;
    border-top:    6px solid transparent;
    border-bottom: 6px solid transparent;
    border-right:  10px solid #FF8FAE;
  }

  :global(#typing-dots) {
    display: none; gap: 4px; align-items: center; padding: 2px 0;
  }
  :global(#typing-dots span) {
    width: 6px; height: 6px;
    background: #4B1528; border-radius: 50%;
    animation: dot-bounce 1.2s infinite;
  }
  :global(#typing-dots span:nth-child(2)) { animation-delay: 0.2s; }
  :global(#typing-dots span:nth-child(3)) { animation-delay: 0.4s; }
  
  @keyframes :global(dot-bounce) {
    0%, 80%, 100% { transform: translateY(0); opacity: 0.5; }
    40%           { transform: translateY(-5px); opacity: 1; }
  }

  :global(#hint) {
    position: fixed;
    bottom: 76px;
    left: 56%;
    font-size: 12px;
    color: #993556;
    opacity: 0.75;
    pointer-events: none;
    white-space: nowrap;
    z-index: 10;
    transition: opacity 0.4s ease;
  }

  :global(#input-area) {
    position: fixed;
    bottom: 18px;
    left: 56%;
    width: min(340px, 42vw);
    display: flex;
    gap: 8px;
    z-index: 10;
  }

  :global(#user-input) {
    flex: 1;
    background: #FFFDF0;
    border: 0.5px solid #D4537E;
    border-radius: 16px;
    padding: 10px 16px;
    font-size: 15px;
    color: #3D2620;
    outline: none;
    resize: none;
    height: 48px;
    min-height: 48px;
    max-height: 96px;
    font-family: "Inter", sans-serif;
    line-height: 1.5;
    box-sizing: border-box;
  }
  :global(#user-input::placeholder) { color: #b07890; }
  :global(#user-input:focus) {
    border-color: #F04A6F;
    box-shadow: 0 0 0 3px rgba(240, 74, 111, 0.18);
  }

  :global(#send-btn) {
    width: 48px; height: 48px;
    border-radius: 50%;
    background: #FF8FAE;
    border: none; cursor: pointer;
    display: flex; align-items: center; justify-content: center;
    color: #4B1528;
    font-size: 17px;
    flex-shrink: 0;
    transition: background 0.15s, transform 0.1s;
  }
  :global(#send-btn:hover)  { background: #F04A6F; color: white; }
  :global(#send-btn:active) { transform: scale(0.94); }

  .credit {
    position: fixed; bottom: 8px; left: 12px;
    font-size: 11px; color: rgba(77,20,40,0.5);
    z-index: 10; pointer-events: none;
  }
</style>