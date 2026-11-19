name: webxr-blocks-spatial-game-builder
description: Standard operating procedure, architectural blueprint, and canonical production template for generating interactive WebXR spatial quiz, trivia, and target-shooting games using XR Blocks (xrblocks) and Three.js, playable across VR headsets (Meta Quest 3, Android XR), desktop browsers, and mobile touchscreens.
triggers:
  - "Create an XR game using XR Blocks"
  - "Build a WebXR spatial quiz"
  - "Create a VR shooting gallery / trivia game"
  - "Design an Android XR / Meta Quest 3 educational game"
  - "Build a cross-platform WebXR / desktop / mobile 3D game"
version: 2.0.0

# WebXR Spatial Game Builder (XR Blocks)

## 0. Anti-Hallucination Guardrails (CRITICAL CONTRACT)

The generating model **MUST NEVER** emit any of these anti-patterns:

| ❌ FATAL ANTI-PATTERN (Bug / Crash) | ✅ MANDATORY XR BLOCKS STANDARD |
| :--- | :--- |
| `import { xb } from 'xrblocks'` | `import * as xb from 'xrblocks';` |
| `xb.registerScript(...)` or `xb.init({ container })` | `xb.add(new GameScript()); await xb.init(new xb.Options());` |
| `xb.core.scene.add(mesh)` | `this.add(mesh);` (`xb.Script` extends `THREE.Object3D`) |
| `uiText.setText('...')` or `btn.setText('...')` | `uiText.text = '...'` and `btn.label = '...'` (Property setters, NOT methods) |
| `new xb.UIText('str', { style })` | `new xb.UIText({ text: 'str', style: { ... } })` (Object config) |
| `new xb.UIButton('lbl', onClick, { style })` | `new xb.UIButton({ label: 'lbl', onClick: () => {}, style: { ... } })` |
| `card.add(panel)` or `panel.add(text)` | Declarative `children: [child1, child2]` in constructor config |
| `mesh.add(new xb.UIText(...))` | Render text on a `THREE.CanvasTexture` with `.needsUpdate = true` |
| Reassigning texture without `needsUpdate = true` | `mesh.material.map = newTex; mesh.material.map.needsUpdate = true; mesh.material.needsUpdate = true;` |
| `target.xb` missing | Every target mesh MUST have `mesh.xb = { pointerEvents: 'auto', reticleMode: 'auto' };` |
| Background / floor / particles missing ray filter | Every non-target mesh MUST have `mesh.xb = { pointerEvents: 'none' };` |
| Calling `.geometry.dispose()` inside frame loop | **Pre-allocated Particle Pool:** Reusable meshes with `.visible = true/false` |
| Synchronous target sway (`Math.sin(time)`) | Each target MUST have an independent `data.swayPhase = Math.random() * Math.PI * 2` |
| Wrong answer ending the question round | A wrong hit removes ONLY the wrong block after 400ms, letting the player hit remaining blocks |
| Falling blocks continuing to drop during hit transition | Check `if (this.isTransitioning) return;` before moving blocks or checking floor collisions |

---

## 1. Complete Canonical Reference Implementation

Every generated game **MUST** follow this exact, bug-free architectural structure:

```html
<!doctype html>
<html lang="en">
  <head>
    <title>WebXR Target Game</title>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, viewport-fit=cover" />
    <link type="text/css" rel="stylesheet" href="https://xrblocks.github.io/css/xr.css" />
    <script>window.litDisableBundleWarning = true;</script>
    <script type="importmap">
    {
      "imports": {
        "three": "https://cdn.jsdelivr.net/npm/three@0.184.0/build/three.module.js",
        "three/": "https://cdn.jsdelivr.net/npm/three@0.184.0/",
        "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.184.0/examples/jsm/",
        "@pmndrs/uikit": "https://cdn.jsdelivr.net/npm/@pmndrs/uikit@1.0.64/dist/index.min.js",
        "@pmndrs/uikit-pub-sub": "https://cdn.jsdelivr.net/npm/@pmndrs/uikit-pub-sub@1.0.64/dist/index.min.js",
        "@pmndrs/msdfonts": "https://cdn.jsdelivr.net/npm/@pmndrs/msdfonts@1.0.64/dist/index.min.js",
        "@preact/signals-core": "https://cdn.jsdelivr.net/npm/@preact/signals-core@1.14.0/dist/signals-core.mjs",
        "yoga-layout/load": "https://cdn.jsdelivr.net/npm/yoga-layout@3.2.1/dist/src/load.js",
        "lit": "https://esm.sh/lit@3.3.1",
        "lit/": "https://esm.sh/lit@3.3.1/",
        "xrblocks": "https://cdn.jsdelivr.net/npm/xrblocks@0.21.1/build/xrblocks.js",
        "xrblocks/addons/": "https://cdn.jsdelivr.net/npm/xrblocks@0.21.1/build/addons/"
      }
    }
    </script>
    <style>
      body, html {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: #070b14;
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
        touch-action: none; user-select: none; -webkit-user-select: none;
      }
    </style>
  </head>
  <body>
    <script type="module">
      import * as THREE from 'three';
      import * as xb from 'xrblocks';

      // --- 1. PROCEDURAL AUDIO ENGINE ---
      class SoundFX {
        constructor() { this.ctx = null; }
        ensureContext() {
          if (!this.ctx) {
            const AudioCtx = window.AudioContext || window.webkitAudioContext;
            this.ctx = new AudioCtx();
          }
          if (this.ctx.state === 'suspended') this.ctx.resume();
        }
        playSpray() {
          try {
            this.ensureContext();
            const b = this.ctx.createBuffer(1, this.ctx.sampleRate * 0.15, this.ctx.sampleRate);
            const d = b.getChannelData(0);
            for (let i = 0; i < d.length; i++) d[i] = Math.random() * 2 - 1;
            const src = this.ctx.createBufferSource();
            src.buffer = b;
            const f = this.ctx.createBiquadFilter();
            f.type = 'bandpass';
            f.frequency.setValueAtTime(1400, this.ctx.currentTime);
            const g = this.ctx.createGain();
            g.gain.setValueAtTime(0.28, this.ctx.currentTime);
            g.gain.exponentialRampToValueAtTime(0.01, this.ctx.currentTime + 0.14);
            src.connect(f); f.connect(g); g.connect(this.ctx.destination);
            src.start();
          } catch (_) {}
        }
        playSuccess() {
          try {
            this.ensureContext();
            const now = this.ctx.currentTime;
            [523.25, 659.25, 783.99, 1046.5].forEach((freq, idx) => {
              const osc = this.ctx.createOscillator();
              const gain = this.ctx.createGain();
              osc.type = 'sine';
              osc.frequency.setValueAtTime(freq, now + idx * 0.05);
              gain.gain.setValueAtTime(0.18, now + idx * 0.05);
              gain.gain.exponentialRampToValueAtTime(0.001, now + idx * 0.05 + 0.22);
              osc.connect(gain); gain.connect(this.ctx.destination);
              osc.start(now + idx * 0.05);
              osc.stop(now + idx * 0.05 + 0.25);
            });
          } catch (_) {}
        }
        playWrong() {
          try {
            this.ensureContext();
            const now = this.ctx.currentTime;
            const osc = this.ctx.createOscillator();
            const gain = this.ctx.createGain();
            osc.type = 'sawtooth';
            osc.frequency.setValueAtTime(140, now);
            osc.frequency.linearRampToValueAtTime(90, now + 0.25);
            gain.gain.setValueAtTime(0.22, now);
            gain.gain.exponentialRampToValueAtTime(0.01, now + 0.28);
            osc.connect(gain); gain.connect(this.ctx.destination);
            osc.start(now); osc.stop(now + 0.3);
          } catch (_) {}
        }
        playFloorThud() {
          try {
            this.ensureContext();
            const now = this.ctx.currentTime;
            const osc = this.ctx.createOscillator();
            const gain = this.ctx.createGain();
            osc.type = 'triangle';
            osc.frequency.setValueAtTime(110, now);
            osc.frequency.exponentialRampToValueAtTime(35, now + 0.3);
            gain.gain.setValueAtTime(0.35, now);
            gain.gain.exponentialRampToValueAtTime(0.005, now + 0.32);
            osc.connect(gain); gain.connect(this.ctx.destination);
            osc.start(now); osc.stop(now + 0.35);
          } catch (_) {}
        }
        playTick() {
          try {
            this.ensureContext();
            const now = this.ctx.currentTime;
            const osc = this.ctx.createOscillator();
            const gain = this.ctx.createGain();
            osc.type = 'sine';
            osc.frequency.setValueAtTime(900, now);
            gain.gain.setValueAtTime(0.06, now);
            gain.gain.exponentialRampToValueAtTime(0.001, now + 0.04);
            osc.connect(gain); gain.connect(this.ctx.destination);
            osc.start(now); osc.stop(now + 0.05);
          } catch (_) {}
        }
      }

      // --- 2. ZERO-ALLOCATION PARTICLE ENGINE ---
      class ParticleFX {
        constructor(scene) {
          this.scene = scene;
          this.particles = [];
          const geo = new THREE.SphereGeometry(0.024, 8, 8);
          this.sprayMat = new THREE.MeshBasicMaterial({ color: 0xe0f2fe, transparent: true, opacity: 0.85 });
          this.sparkMat = new THREE.MeshBasicMaterial({ color: 0x4ade80, transparent: true, opacity: 0.95 });
          this.smokeMat = new THREE.MeshBasicMaterial({ color: 0xf87171, transparent: true, opacity: 0.85 });

          for (let i = 0; i < 90; i++) {
            const mesh = new THREE.Mesh(geo, this.sprayMat);
            mesh.visible = false;
            mesh.xb = { pointerEvents: 'none' }; // CRITICAL: Never block rays
            this.scene.add(mesh);
            this.particles.push({ mesh, vel: new THREE.Vector3(), life: 0, maxLife: 1 });
          }
        }
        spawnSprayStream(origin, targetPos) {
          const dir = new THREE.Vector3().subVectors(targetPos, origin).normalize();
          for (let i = 0; i < 18; i++) {
            const p = this.particles.find(item => !item.mesh.visible);
            if (!p) break;
            p.mesh.material = this.sprayMat;
            p.mesh.scale.setScalar(0.7 + Math.random() * 0.8);
            p.mesh.position.copy(origin).add(
              new THREE.Vector3((Math.random() - 0.5) * 0.04, (Math.random() - 0.5) * 0.04, (Math.random() - 0.5) * 0.04)
            );
            p.vel.copy(dir).multiplyScalar(4.5 + Math.random() * 1.5);
            p.life = 0;
            p.maxLife = 0.35;
            p.mesh.visible = true;
          }
        }
        spawnBurst(pos, isCorrect = true) {
          const count = isCorrect ? 24 : 18;
          for (let i = 0; i < count; i++) {
            const p = this.particles.find(item => !item.mesh.visible);
            if (!p) break;
            p.mesh.material = isCorrect ? this.sparkMat : this.smokeMat;
            p.mesh.scale.setScalar(1.0 + Math.random() * 0.8);
            p.mesh.position.copy(pos);
            p.vel.set((Math.random() - 0.5) * 2.5, (Math.random() - 0.2) * 2.0, (Math.random() - 0.5) * 2.5);
            p.life = 0;
            p.maxLife = 0.5;
            p.mesh.visible = true;
          }
        }
        update(dt) {
          for (const p of this.particles) {
            if (!p.mesh.visible) continue;
            p.life += dt;
            if (p.life >= p.maxLife) {
              p.mesh.visible = false;
              continue;
            }
            p.mesh.position.addScaledVector(p.vel, dt);
            p.mesh.scale.multiplyScalar(0.97);
          }
        }
        dispose() {
          for (const p of this.particles) {
            this.scene.remove(p.mesh);
            p.mesh.geometry.dispose();
          }
          this.sprayMat.dispose();
          this.sparkMat.dispose();
          this.smokeMat.dispose();
        }
      }

      // --- 3. DYNAMIC CANVAS TEXTURE GENERATOR ---
      function createTargetTexture(label, text, state = 'normal') {
        const canvas = document.createElement('canvas');
        canvas.width = 1024;
        canvas.height = 512;
        const ctx = canvas.getContext('2d');

        let bgColor = '#0b1329';
        let borderColor = '#38bdf8';
        if (state === 'correct') { bgColor = '#064e3b'; borderColor = '#10b981'; }
        else if (state === 'wrong') { bgColor = '#7f1d1d'; borderColor = '#ef4444'; }

        ctx.fillStyle = bgColor;
        ctx.beginPath();
        ctx.roundRect(20, 20, 984, 472, 36);
        ctx.fill();

        ctx.lineWidth = 14;
        ctx.strokeStyle = borderColor;
        ctx.stroke();

        ctx.fillStyle = '#0284c7';
        ctx.beginPath();
        ctx.roundRect(50, 48, 160, 72, 20);
        ctx.fill();

        ctx.fillStyle = '#ffffff';
        ctx.font = 'bold 44px sans-serif';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(`OPTION ${label}`, 130, 84);

        ctx.fillStyle = '#94a3b8';
        ctx.font = 'bold 32px sans-serif';
        ctx.textAlign = 'right';
        ctx.fillText('AIM & SHOOT 🎯', 950, 84);

        ctx.fillStyle = '#ffffff';
        ctx.font = 'bold 54px sans-serif';
        ctx.textAlign = 'center';

        const words = text.split(' ');
        const lines = [];
        let currentLine = '';
        for (const w of words) {
          const testLine = currentLine ? `${currentLine} ${w}` : w;
          if (ctx.measureText(testLine).width > 860) {
            lines.push(currentLine);
            currentLine = w;
          } else {
            currentLine = testLine;
          }
        }
        if (currentLine) lines.push(currentLine);

        const startY = 280 - ((lines.length - 1) * 70) / 2;
        lines.forEach((line, index) => {
          ctx.fillText(line, 512, startY + index * 72);
        });

        const texture = new THREE.CanvasTexture(canvas);
        texture.colorSpace = THREE.SRGBColorSpace;
        return texture;
      }

      // --- 4. MAIN GAME SCRIPT CLASS ---
      class GameScript extends xb.Script {
        constructor(questions) {
          super();
          this.allQuestions = questions;
          this.score = 0;
          this.streak = 0;
          this.timeLeft = 30.0;
          this.isPlaying = false;
          this.isTransitioning = false;

          this.questionPool = [];
          this.poolIndex = 0;
          this.fallingBlocks = [];
          this.blockGeo = new THREE.BoxGeometry(0.68, 0.34, 0.07);

          this.raycaster = new THREE.Raycaster();
          this.pointer = new THREE.Vector2();
          this.tempMatrix = new THREE.Matrix4();
          this.rayOrigin = new THREE.Vector3();
          this.rayDirection = new THREE.Vector3();

          this.boundPointerDown = (e) => this.onScreenPointerDown(e);
        }

        init() {
          this.sfx = new SoundFX();
          this.fx = new ParticleFX(this);

          // Lights
          this.add(new THREE.HemisphereLight(0xffffff, 0x1e293b, 2.5));
          const dirLight = new THREE.DirectionalLight(0xfff7ed, 2.0);
          dirLight.position.set(2, 4, 1);
          this.add(dirLight);

          // Floor Grounding (Marked pointerEvents: 'none')
          const floor = new THREE.Mesh(
            new THREE.PlaneGeometry(12, 12),
            new THREE.MeshStandardMaterial({ color: 0x0f172a, roughness: 0.8 })
          );
          floor.rotation.x = -Math.PI / 2;
          floor.xb = { pointerEvents: 'none' };
          this.add(floor);

          this.buildSpatialUI();
          window.addEventListener('pointerdown', this.boundPointerDown, { passive: false });
        }

        buildSpatialUI() {
          this.timerText = new xb.UIText({
            text: '30s',
            style: { fontSize: 24, fontWeight: 'bold', color: '#38bdf8' }
          });
          this.scoreText = new xb.UIText({
            text: 'Score: 0',
            style: { fontSize: 24, fontWeight: 'bold', color: '#4ade80' }
          });
          this.streakText = new xb.UIText({
            text: 'Streak: 0x',
            style: { fontSize: 22, fontWeight: 'bold', color: '#fbbf24' }
          });

          const statsPanel = new xb.UIPanel({
            style: {
              width: '100%',
              flexDirection: 'row',
              justifyContent: 'space-between',
              alignItems: 'center',
              backgroundColor: 'rgba(15, 23, 42, 0.8)',
              padding: 10,
              borderRadius: 14
            },
            children: [this.timerText, this.scoreText, this.streakText]
          });

          this.questionText = new xb.UIText({
            text: '🎯 WEBXR TARGET DRILL\nPull trigger in VR or tap/click targets on mobile & desktop!',
            style: { fontSize: 24, fontWeight: 'bold', textAlign: 'center', lineHeight: 1.35, color: '#ffffff' }
          });

          const questionPanel = new xb.UIPanel({
            style: {
              width: '100%',
              backgroundColor: 'rgba(15, 23, 42, 0.95)',
              borderRadius: 16,
              borderWidth: 2,
              borderColor: '#f59e0b',
              padding: 8,
              justifyContent: 'center',
              alignItems: 'center'
            },
            children: [this.questionText]
          });

          this.statusBanner = new xb.UIText({
            text: 'Ready to train! 30-Second Fast Response Drill.',
            style: { fontSize: 17, fontWeight: 'bold', textAlign: 'center', color: '#f8fafc' }
          });

          this.startBtn = new xb.UIButton({
            label: 'Start 30s Challenge',
            icon: 'play_arrow',
            style: { flexGrow: 1.2 },
            onClick: () => this.toggleStartPause()
          });

          const resetBtn = new xb.UIButton({
            label: 'Reset',
            icon: 'restart_alt',
            style: { flexGrow: 0.9 },
            onClick: () => this.resetGame()
          });

          const exitBtn = new xb.UIButton({
            label: 'Exit VR',
            icon: 'logout',
            style: { flexGrow: 0.9 },
            onClick: () => this.exitVR()
          });

          const buttonsRow = new xb.UIPanel({
            style: { width: '100%', flexDirection: 'row', gap: 10, justifyContent: 'center' },
            children: [this.startBtn, resetBtn, exitBtn]
          });

          this.hudCard = new xb.UICard({
            size: { width: 0.96, height: 'auto' },
            manipulation: false,
            edge: false,
            style: {
              flexDirection: 'column',
              gap: 10,
              padding: 16,
              backgroundColor: 'rgba(11, 19, 43, 0.94)',
              borderRadius: 22,
              borderColor: '#38bdf8',
              borderWidth: 2
            },
            children: [statsPanel, questionPanel, this.statusBanner, buttonsRow]
          });

          this.hudCard.position.set(0, 1.92, -1.35);
          this.hudCard.rotation.x = 0.14;
          this.add(this.hudCard);
        }

        shuffleQuestions() {
          this.questionPool = [...this.allQuestions]
            .map(q => ({ q, r: Math.random() }))
            .sort((a, b) => a.r - b.r)
            .map(i => i.q);
          this.poolIndex = 0;
        }

        toggleStartPause() {
          this.sfx.ensureContext();
          if (this.timeLeft <= 0) {
            this.resetGame();
            return;
          }
          this.isPlaying = !this.isPlaying;
          if (this.isPlaying) {
            this.startBtn.label = 'Pause';
            this.startBtn.icon = 'pause';
            this.statusBanner.text = 'TARGETS DESCENDING: Shoot the correct option!';
            if (this.fallingBlocks.length === 0) {
              if (this.questionPool.length === 0) this.shuffleQuestions();
              this.spawnNextQuestion();
            }
          } else {
            this.startBtn.label = 'Resume';
            this.startBtn.icon = 'play_arrow';
            this.statusBanner.text = 'Challenge paused.';
          }
        }

        resetGame() {
          this.score = 0;
          this.streak = 0;
          this.timeLeft = 30.0;
          this.isPlaying = false;
          this.isTransitioning = false;
          this.shuffleQuestions();

          this.scoreText.text = 'Score: 0';
          this.streakText.text = 'Streak: 0x';
          this.timerText.text = '30s';
          this.timerText.style.color = '#38bdf8';
          this.startBtn.label = 'Start 30s Challenge';
          this.startBtn.icon = 'play_arrow';
          this.questionText.text = '🎯 WEBXR TARGET DRILL\nPull trigger in VR or tap/click targets on mobile & desktop!';
          this.statusBanner.text = 'Game reset. Ready for drill.';
          this.clearFallingBlocks();
        }

        exitVR() {
          if (xb.core?.renderer?.xr?.isPresenting) {
            const session = xb.core.renderer.xr.getSession();
            if (session) session.end();
          }
        }

        spawnNextQuestion() {
          this.clearFallingBlocks();
          this.isTransitioning = false;

          if (this.poolIndex >= this.questionPool.length) this.shuffleQuestions();
          const qData = this.questionPool[this.poolIndex++];
          this.questionText.text = `Q${this.poolIndex}: ${qData.question}`;

          const shuffledOpts = [...qData.options]
            .map(opt => ({ opt, r: Math.random() }))
            .sort((a, b) => a.r - b.r)
            .map(i => i.opt);

          const aspect = window.innerWidth / window.innerHeight;
          const laneScale = aspect < 1.0 ? Math.min(1.0, aspect * 1.1) : 1.0;
          const laneBases = [-0.64 * laneScale, 0.0, 0.64 * laneScale];

          shuffledOpts.forEach((option, i) => {
            const tex = createTargetTexture(option.label, option.text, 'normal');
            const mat = new THREE.MeshStandardMaterial({ map: tex, roughness: 0.3, metalness: 0.2 });
            const block = new THREE.Mesh(this.blockGeo, mat);

            const initialX = laneBases[i];
            const initialY = 1.70 + Math.random() * 0.12;
            block.position.set(initialX, initialY, -1.15);

            block.userData = {
              isAnswerBlock: true,
              isCorrect: option.correct,
              optionText: option.text,
              label: option.label,
              mat: mat,
              fallSpeed: 0.15 + Math.random() * 0.14 + (i * 0.03), // Independent speeds
              baseX: initialX,
              swayAmplitude: (0.10 + Math.random() * 0.15) * laneScale,
              swaySpeed: 1.6 + Math.random() * 2.2,
              swayPhase: Math.random() * Math.PI * 2, // CRITICAL: Independent phase prevents lockstep sway
              driftSpeed: (Math.random() - 0.5) * 0.15 * laneScale,
              timeAlive: 0
            };

            // CRITICAL: Must be marked pointerEvents: 'auto' for controller raycast
            block.xb = { pointerEvents: 'auto', reticleMode: 'auto' };

            this.add(block);
            this.fallingBlocks.push(block);
          });
        }

        clearFallingBlocks() {
          for (const b of this.fallingBlocks) {
            this.remove(b);
            if (b.userData.mat) {
              if (b.userData.mat.map) b.userData.mat.map.dispose();
              b.userData.mat.dispose();
            }
          }
          this.fallingBlocks = [];
        }

        // Tri-Input: Desktop Mouse & Mobile Screen Tap
        onScreenPointerDown(event) {
          this.sfx.ensureContext();
          if (xb.core?.renderer?.xr?.isPresenting) return;
          if (event.target && event.target.closest && event.target.closest('button')) return;

          const rect = xb.core.renderer.domElement.getBoundingClientRect();
          this.pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
          this.pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

          this.raycaster.setFromCamera(this.pointer, xb.core.camera);
          const hits = this.raycaster.intersectObjects(this.fallingBlocks, false);
          const cameraPos = xb.core.camera.position.clone();

          if (hits.length > 0) {
            this.handleBlockShot(hits[0].object, cameraPos);
          } else if (this.isPlaying && !this.isTransitioning) {
            this.sfx.playSpray();
            const sprayTarget = cameraPos.clone().addScaledVector(this.raycaster.ray.direction, 1.5);
            this.fx.spawnSprayStream(cameraPos, sprayTarget);
          }
        }

        // Tri-Input: Meta Quest 3 Controller Trigger Pull
        onSelectStart(event) {
          this.sfx.ensureContext();
          if (!this.isPlaying || this.isTransitioning) return;

          const controller = event.source?.controller;
          let shootOrigin = new THREE.Vector3(0, 1.4, 0);

          if (controller) {
            shootOrigin.setFromMatrixPosition(controller.matrixWorld);
            this.tempMatrix.identity().extractRotation(controller.matrixWorld);
            this.rayDirection.set(0, 0, -1).applyMatrix4(this.tempMatrix).normalize();
          } else {
            shootOrigin.setFromMatrixPosition(xb.core.camera.matrixWorld);
            xb.core.camera.getWorldDirection(this.rayDirection);
          }

          this.raycaster.set(shootOrigin, this.rayDirection);
          const hits = this.raycaster.intersectObjects(this.fallingBlocks, false);
          if (hits.length > 0) {
            this.handleBlockShot(hits[0].object, shootOrigin);
          } else {
            this.sfx.playSpray();
            const sprayTarget = shootOrigin.clone().addScaledVector(this.rayDirection, 1.5);
            this.fx.spawnSprayStream(shootOrigin, sprayTarget);
          }
        }

        // Tri-Input: XR Blocks Reticle Selection Hook
        onObjectSelectStart(event) {
          const target = event.target;
          if (this.isPlaying && !this.isTransitioning && target?.userData?.isAnswerBlock) {
            const origin = event.source?.controller
              ? event.source.controller.position.clone()
              : xb.core.camera.position.clone();
            this.handleBlockShot(target, origin);
            event.stopPropagation();
          }
        }

        handleBlockShot(block, shootOrigin) {
          if (this.isTransitioning) return;

          this.sfx.playSpray();
          this.fx.spawnSprayStream(shootOrigin, block.position);

          const isCorrect = block.userData.isCorrect;
          if (isCorrect) {
            this.isTransitioning = true;
            this.sfx.playSuccess();
            this.streak += 1;
            const points = 100 + (this.streak - 1) * 35;
            this.score += points;

            this.fx.spawnBurst(block.position, true);
            this.scoreText.text = `Score: ${this.score}`;
            this.streakText.text = `Streak: ${this.streak}x`;

            // Texture Update: CRITICAL needsUpdate flag
            block.userData.mat.map = createTargetTexture(block.userData.label, block.userData.optionText, 'correct');
            block.userData.mat.map.needsUpdate = true;
            block.userData.mat.needsUpdate = true;

            this.statusBanner.text = `🎯 TARGET HIT! +${points} PTS!`;
            this.statusBanner.style.color = '#4ade80';

            setTimeout(() => {
              if (this.isPlaying && this.timeLeft > 0) this.spawnNextQuestion();
            }, 550);
          } else {
            this.sfx.playWrong();
            this.streak = 0;
            this.score = Math.max(0, this.score - 50);

            this.fx.spawnBurst(block.position, false);
            this.scoreText.text = `Score: ${this.score}`;
            this.streakText.text = 'Streak: 0x';

            // Texture Update: CRITICAL needsUpdate flag
            block.userData.mat.map = createTargetTexture(block.userData.label, block.userData.optionText, 'wrong');
            block.userData.mat.map.needsUpdate = true;
            block.userData.mat.needsUpdate = true;

            this.statusBanner.text = '⚠️ WRONG TARGET! -50 PTS penalty!';
            this.statusBanner.style.color = '#ef4444';

            // CRITICAL: Remove ONLY the wrong block after 400ms so player can hit remaining blocks
            setTimeout(() => {
              const idx = this.fallingBlocks.indexOf(block);
              if (idx !== -1) {
                this.remove(block);
                this.fallingBlocks.splice(idx, 1);
              }
              if (this.fallingBlocks.length === 0 && this.isPlaying && this.timeLeft > 0) {
                this.spawnNextQuestion();
              }
            }, 400);
          }
        }

        update() {
          const dt = xb.getDeltaTime();
          if (this.fx) this.fx.update(dt);
          if (!this.isPlaying) return;

          this.timeLeft -= dt;
          if (this.timeLeft <= 0) {
            this.timeLeft = 0;
            this.timerText.text = '0s';
            this.finishGame();
            return;
          }

          const secDisplay = Math.ceil(this.timeLeft);
          this.timerText.text = `${secDisplay}s`;
          if (secDisplay <= 8) {
            this.timerText.style.color = '#ef4444';
            if (Math.floor(this.timeLeft * 2) !== Math.floor((this.timeLeft + dt) * 2)) {
              this.sfx.playTick();
            }
          }

          // CRITICAL: Freeze block motion during question transition to prevent false floor collisions
          if (this.isTransitioning) return;

          const aspect = window.innerWidth / window.innerHeight;
          const maxCorridorX = aspect < 1.0 ? Math.max(0.38, aspect * 0.72) : 0.85;

          let floorHitTriggered = false;
          for (let i = this.fallingBlocks.length - 1; i >= 0; i--) {
            const block = this.fallingBlocks[i];
            const data = block.userData;
            data.timeAlive += dt;

            // Independent Descent
            block.position.y -= data.fallSpeed * dt;

            // Independent Sway & Drift with individual Phase offset
            const swayOffset = Math.sin(data.timeAlive * data.swaySpeed + data.swayPhase) * data.swayAmplitude;
            const driftOffset = data.driftSpeed * data.timeAlive;
            const newX = data.baseX + swayOffset + driftOffset;

            block.position.x = THREE.MathUtils.clamp(newX, -maxCorridorX, maxCorridorX);
            block.rotation.z = Math.sin(data.timeAlive * data.swaySpeed * 0.8) * 0.08;

            // Check floor crash penalty threshold (0.42m)
            if (block.position.y <= 0.42) {
              floorHitTriggered = true;
            }
          }

          if (floorHitTriggered) {
            this.isTransitioning = true;
            this.sfx.playFloorThud();
            this.streak = 0;
            this.score = Math.max(0, this.score - 30);
            this.scoreText.text = `Score: ${this.score}`;
            this.streakText.text = 'Streak: 0x';
            this.statusBanner.text = '💥 TARGET HIT THE FLOOR! -30 PTS penalty!';
            this.statusBanner.style.color = '#f97316';

            setTimeout(() => {
              if (this.isPlaying && this.timeLeft > 0) this.spawnNextQuestion();
            }, 650);
          }
        }

        finishGame() {
          this.isPlaying = false;
          this.startBtn.label = 'Play Again';
          this.startBtn.icon = 'restart_alt';
          this.clearFallingBlocks();

          let rank = 'Safety Cadet';
          if (this.score >= 450) rank = 'Elite Responder ⚡';
          else if (this.score >= 280) rank = 'Senior Specialist 🛡️';
          else if (this.score >= 120) rank = 'Certified Technician 🎯';

          this.questionText.text = `🏆 DRILL COMPLETED!\nRank: ${rank}\nFinal Score: ${this.score} PTS`;
          this.statusBanner.text = '30s challenge finished. Tap Play Again to restart with fresh questions.';
          this.statusBanner.style.color = '#38bdf8';
        }

        dispose() {
          window.removeEventListener('pointerdown', this.boundPointerDown);
          this.clearFallingBlocks();
          if (this.fx) this.fx.dispose();
          this.blockGeo.dispose();
        }
      }

      // Initialize
      const options = new xb.Options();
      options.setAppTitle('WebXR Target Game');
      options.controllers.visualizeRays = true;
      options.reticles.enabled = true;
      options.interaction.raycastMode = 'select';

      const QUESTION_BANK = [
        {
          question: "Sample Question 1?",
          options: [
            { label: 'A', text: 'Correct Answer', correct: true },
            { label: 'B', text: 'Wrong Answer 1', correct: false },
            { label: 'C', text: 'Wrong Answer 2', correct: false }
          ]
        }
      ];

      document.addEventListener('DOMContentLoaded', async () => {
        xb.add(new GameScript(QUESTION_BANK));
        await xb.init(options);
      });
    </script>
  </body>
</html>
```

---

## 2. Universal Verification Checklist

Before emitting code, verify:
- [ ] Uses `import * as xb from 'xrblocks';` (NOT `import { xb }`).
- [ ] `xb.UIText` properties are set via `.text = '...'`, never `.setText()`.
- [ ] `xb.UIButton` properties are set via `.label = '...'`, never `.setText()`.
- [ ] Containers use declarative `children: [child1, child2]` in config, never `.add()` on UI objects.
- [ ] Target textures flag both `mat.map.needsUpdate = true;` and `mat.needsUpdate = true;`.
- [ ] Target meshes set `mesh.xb = { pointerEvents: 'auto', reticleMode: 'auto' };`.
- [ ] Backgrounds, floor, and particle pool meshes set `mesh.xb = { pointerEvents: 'none' };`.
- [ ] Particle pool is pre-allocated at startup; never calls `.geometry.dispose()` inside frame update loop.
- [ ] Each target has an independent `swayPhase` to prevent lockstep swaying.
- [ ] `update()` checks `if (this.isTransitioning) return;` so blocks freeze during hit transitions and do not trigger false floor penalties.
- [ ] Wrong hits remove ONLY the hit block after 400ms, allowing remaining blocks to still be hit.