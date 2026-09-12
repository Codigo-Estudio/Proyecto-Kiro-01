# Audio-Visual Steering File

This steering file defines sound effect integration, screen shake mechanics, and UI animation patterns for the Ghosty game.

## Table of Contents

- [Sound Effect Integration](#sound-effect-integration)
- [Screen Shake Mechanics](#screen-shake-mechanics)
- [UI Animation Patterns](#ui-animation-patterns)
- [Audio-Visual Systems Integration](#audio-visual-systems-integration)

---

## Sound Effect Integration

### 1. Audio Manager

```javascript
class AudioManager {
  constructor() {
    this.sounds = new Map();
    this.music = new Map();
    this.masterVolume = 1.0;
    this.sfxVolume = 1.0;
    this.musicVolume = 0.7;
    this.enabled = true;
    this.muted = false;
  }

  async loadSound(key, path, type = 'effect') {
    const audio = new Audio(path);
    
    // Preload audio
    await new Promise((resolve, reject) => {
      audio.addEventListener('canplaythrough', resolve);
      audio.addEventListener('error', reject);
      audio.load();
    });

    if (type === 'effect') {
      this.sounds.set(key, audio);
    } else {
      this.music.set(key, audio);
    }

    return audio;
  }

  async loadSounds(sounds) {
    return Promise.all(
      sounds.map(item => this.loadSound(item.key, item.path, item.type))
    );
  }

  playSound(key, options = {}) {
    if (!this.enabled || this.muted) return;

    const sound = this.sounds.get(key);
    if (!sound) return;

    const audio = sound.cloneNode();
    audio.volume = this.sfxVolume * this.masterVolume;
    
    if (options.pitch) {
      audio.playbackRate = options.pitch;
    }

    if (options.loop) {
      audio.loop = true;
    }

    if (options.startTime) {
      audio.currentTime = options.startTime;
    }

    // Cancel previous same sound
    if (options.cancelPrevious) {
      this.cancelSound(key);
    }

    // Add to active sounds
    this.activeSounds.set(key, audio);

    audio.play().catch(e => console.warn('Audio play failed:', e));

    if (!options.loop) {
      audio.onended = () => {
        this.activeSounds.delete(key);
      };
    }

    return audio;
  }

  playMusic(key, options = {}) {
    if (!this.enabled || this.muted) return;

    // Stop current music
    this.music.forEach(audio => {
      audio.pause();
      audio.currentTime = 0;
    });

    const music = this.music.get(key);
    if (!music) return;

    music.volume = this.musicVolume * this.masterVolume;
    
    if (options.loop) {
      music.loop = true;
    }

    music.play().catch(e => console.warn('Music play failed:', e));
    this.currentMusic = key;

    return music;
  }

  stopMusic() {
    if (this.currentMusic) {
      const music = this.music.get(this.currentMusic);
      if (music) {
        music.pause();
        music.currentTime = 0;
      }
    }
  }

  cancelSound(key) {
    const sound = this.activeSounds.get(key);
    if (sound) {
      sound.pause();
      this.activeSounds.delete(key);
    }
  }

  // Volume controls
  setMasterVolume(volume) {
    this.masterVolume = Math.max(0, Math.min(1, volume));
    this.updateAllVolumes();
  }

  setSFXVolume(volume) {
    this.sfxVolume = Math.max(0, Math.min(1, volume));
    this.updateAllVolumes();
  }

  setMusicVolume(volume) {
    this.musicVolume = Math.max(0, Math.min(1, volume));
    this.updateAllVolumes();
  }

  updateAllVolumes() {
    const masterVol = this.masterVolume * (this.muted ? 0 : 1);
    
    this.sounds.forEach(sound => {
      sound.volume = this.sfxVolume * masterVol;
    });

    this.music.forEach(music => {
      music.volume = this.musicVolume * masterVol;
    });
  }

  mute() {
    this.muted = true;
    this.updateAllVolumes();
  }

  unmute() {
    this.muted = false;
    this.updateAllVolumes();
  }

  toggleMute() {
    this.muted = !this.muted;
    this.updateAllVolumes();
  }

  isMuted() {
    return this.muted;
  }

  // Audio effects
  playSoundWithPitch(key, pitch, volume = 1) {
    this.playSound(key, {
      pitch,
      volume: volume * this.sfxVolume * this.masterVolume
    });
  }

  playSoundAtPosition(key, x, y, width, height) {
    // Stereo panning based on position
    const pan = (x - width / 2) / (width / 2);
    this.playSound(key, {
      pan: Math.max(-1, Math.min(1, pan))
    });
  }

  // Sound effects collection
  playJump() {
    this.playSound('jump', { pitch: 1.0 });
  }

  playCollect() {
    this.playSound('collect', { pitch: 1.5 });
  }

  playScore() {
    this.playSound('score', { pitch: 1.2 });
  }

  playHit() {
    this.playSound('hit', { pitch: 0.8 });
  }

  playGameOver() {
    this.playMusic('gameover');
    this.playSound('gameover', { pitch: 0.6 });
  }

  playWin() {
    this.playMusic('win');
    this.playSound('win', { pitch: 1.3 });
  }
}

// Usage example
const audioManager = new AudioManager();

// Load sounds
await audioManager.loadSounds([
  { key: 'jump', path: 'assets/audio/jump.wav', type: 'effect' },
  { key: 'gameover', path: 'assets/audio/game_over.wav', type: 'effect' },
  { key: 'bgm', path: 'assets/audio/music.mp3', type: 'music' }
]);

// Play sounds
audioManager.playJump();
audioManager.playScore();
```

### 2. Sound Effect Player

```javascript
class SoundEffectPlayer {
  constructor(audioManager) {
    this.audioManager = audioManager;
    this.sfxMap = new Map();
    this.soundCount = 0;
  }

  registerSound(key, config) {
    this.sfxMap.set(key, config);
    this.soundCount++;
  }

  play(key, overrides = {}) {
    const config = this.sfxMap.get(key);
    if (!config) return null;

    const options = { ...config, ...overrides };
    
    // Calculate effective volume
    const volume = (options.volume || 1) * 
      this.audioManager.sfxVolume * 
      this.audioManager.masterVolume;

    return this.audioManager.playSound(key, {
      ...options,
      volume,
      pitch: options.pitch || 1.0,
      loop: options.loop || false
    });
  }

  // Game event sound triggers
  playPlayerJump() {
    this.play('jump', {
      pitch: 0.8 + Math.random() * 0.4,
      volume: 0.8
    });
  }

  playPlayerCollect() {
    this.play('collect', {
      pitch: 1.2 + Math.random() * 0.4,
      volume: 0.6
    });
  }

  playPlayerScore() {
    this.play('score', {
      pitch: 1.0 + Math.random() * 0.2,
      volume: 0.5
    });
  }

  playPlayerHit() {
    this.play('hit', {
      pitch: 0.5 + Math.random() * 0.3,
      volume: 0.9,
      duration: 0.3
    });
  }

  playPowerupActivate() {
    this.play('powerup', {
      pitch: 1.5,
      volume: 0.8
    });
  }

  playUIClick() {
    this.play('click', {
      pitch: 1.2,
      volume: 0.4
    });
  }

  playLevelComplete() {
    this.play('complete', {
      pitch: 1.3,
      volume: 0.7
    });
  }

  // Sound variations
  playVariation(key, variations) {
    const variationIndex = Math.floor(Math.random() * variations.length);
    const variation = variations[variationIndex];
    return this.play(key, variation);
  }
}
```

### 3. Audio Visualization

```javascript
class AudioVisualizer {
  constructor(audioContext) {
    this.audioContext = audioContext || new (window.AudioContext || window.webkitAudioContext)();
    this.analyser = this.audioContext.createAnalyser();
    this.analyser.fftSize = 256;
    this.dataArray = new Uint8Array(this.analyser.frequencyBinCount);
    this.visualizations = new Map();
  }

  createWaveformVisualization(element) {
    const canvas = document.createElement('canvas');
    canvas.width = element.clientWidth;
    canvas.height = element.clientHeight;
    element.appendChild(canvas);
    
    this.visualizations.set('waveform', {
      canvas,
      ctx: canvas.getContext('2d')
    });

    return canvas;
  }

  createFrequencyVisualization(element) {
    const canvas = document.createElement('canvas');
    canvas.width = element.clientWidth;
    canvas.height = element.clientHeight;
    element.appendChild(canvas);

    this.visualizations.set('frequency', {
      canvas,
      ctx: canvas.getContext('2d')
    });

    return canvas;
  }

  update() {
    this.analyser.getByteFrequencyData(this.dataArray);

    this.visualizations.forEach((vis, type) => {
      if (type === 'waveform') {
        this.drawWaveform(vis.ctx, vis.canvas.width, vis.canvas.height);
      } else if (type === 'frequency') {
        this.drawFrequency(vis.ctx, vis.canvas.width, vis.canvas.height);
      }
    });
  }

  drawWaveform(ctx, width, height) {
    ctx.clearRect(0, 0, width, height);

    const bufferLength = this.analyser.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);
    this.analyser.getByteTimeDomainData(dataArray);

    ctx.lineWidth = 2;
    ctx.strokeStyle = '#00FFFF';
    ctx.beginPath();

    const sliceWidth = width / bufferLength;
    let x = 0;

    for (let i = 0; i < bufferLength; i++) {
      const v = dataArray[i] / 128.0;
      const y = (v * height) / 2;

      if (i === 0) {
        ctx.moveTo(x, y);
      } else {
        ctx.lineTo(x, y);
      }

      x += sliceWidth;
    }

    ctx.lineTo(width, height / 2);
    ctx.stroke();
  }

  drawFrequency(ctx, width, height) {
    ctx.clearRect(0, 0, width, height);

    ctx.fillStyle = '#FF6347';
    ctx.strokeStyle = '#00FFFF';
    ctx.lineWidth = 2;

    const barWidth = width / this.dataArray.length;
    let x = 0;

    for (let i = 0; i < this.dataArray.length; i++) {
      const barHeight = (this.dataArray[i] / 255) * height;

      ctx.beginPath();
      ctx.rect(x, height - barHeight, barWidth - 2, barHeight);
      ctx.fill();

      // Draw outline
      ctx.beginPath();
      ctx.rect(x, height - barHeight, barWidth - 2, barHeight);
      ctx.stroke();

      x += barWidth;
    }
  }

  drawCircleVisualizer(ctx, centerX, centerY, baseRadius) {
    const data = this.dataArray;
    const sliceAngle = (Math.PI * 2) / data.length;

    for (let i = 0; i < data.length; i++) {
      const value = data[i] / 255;
      const radius = baseRadius + value * 50;
      const angle = i * sliceAngle;

      const x = centerX + Math.cos(angle) * radius;
      const y = centerY + Math.sin(angle) * radius;

      ctx.fillStyle = `hsl(${i}, 100%, 50%)`;
      ctx.beginPath();
      ctx.arc(x, y, 3, 0, Math.PI * 2);
      ctx.fill();
    }
  }
}
```

---

## Screen Shake Mechanics

### 1. Screen Shake Manager

```javascript
class ScreenShakeManager {
  constructor() {
    this.shakeIntensity = 0;
    this.shakeDuration = 0;
    this.shakeStartTime = 0;
    this.isShaking = false;
    this.offset = { x: 0, y: 0 };
    this.shakeCurve = 'easeOut';
  }

  shake(intensity = 10, duration = 0.5, curve = 'easeOut') {
    this.shakeIntensity = intensity;
    this.shakeDuration = duration;
    this.shakeStartTime = performance.now();
    this.shakeCurve = curve;
    this.isShaking = true;
  }

  update(deltaTime) {
    if (!this.isShaking) return;

    const elapsed = (performance.now() - this.shakeStartTime) / 1000;
    
    if (elapsed >= this.shakeDuration) {
      this.isShaking = false;
      this.offset.x = 0;
      this.offset.y = 0;
      return;
    }

    const progress = elapsed / this.shakeDuration;
    const remainingIntensity = this.shakeIntensity * this.getCurveValue(progress);
    const randomX = (Math.random() - 0.5) * 2 * remainingIntensity;
    const randomY = (Math.random() - 0.5) * 2 * remainingIntensity;

    this.offset.x = randomX;
    this.offset.y = randomY;
  }

  getCurveValue(t) {
    switch (this.shakeCurve) {
      case 'easeOut':
        return 1 - t;
      case 'easeInOut':
        return t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2;
      case 'linear':
        return 1 - t;
      case 'exponential':
        return Math.pow(1 - t, 3);
      case 'custom':
        // Custom shake curve
        return (1 - t) * (1 + Math.sin(t * Math.PI * 4) * 0.2);
      default:
        return 1 - t;
    }
  }

  getOffset() {
    return { ...this.offset };
  }

  isActive() {
    return this.isShaking;
  }

  reset() {
    this.isShaking = false;
    this.offset.x = 0;
    this.offset.y = 0;
  }
}
```

### 2. Screen Shake Effects

```javascript
class ShakeEffects {
  constructor(shakeManager) {
    this.shakeManager = shakeManager;
  }

  // Game events that trigger screen shake
  triggerExplosion() {
    this.shakeManager.shake(20, 0.8, 'exponential');
  }

  triggerHeavyImpact() {
    this.shakeManager.shake(15, 0.6, 'custom');
  }

  triggerLightImpact() {
    this.shakeManager.shake(5, 0.3, 'easeOut');
  }

  triggerPowerup() {
    this.shakeManager.shake(10, 0.5, 'linear');
  }

  triggerGameOver() {
    this.shakeManager.shake(10, 1.0, 'exponential');
  }

  triggerWin() {
    this.shakeManager.shake(5, 0.8, 'easeInOut');
  }

  triggerScore(score) {
    // Scale shake intensity based on score
    const intensity = 3 + score * 0.2;
    this.shakeManager.shake(intensity, 0.3, 'easeOut');
  }

  triggerCombo(comboCount) {
    // Scale shake based on combo
    const intensity = Math.min(15, comboCount * 0.5);
    this.shakeManager.shake(intensity, 0.5, 'custom');
  }

  // Camera shake
  shakeCamera(camera, intensity = 5, duration = 0.3) {
    camera.shake(intensity, duration);
  }
}
```

### 3. Combined Visual Feedback

```javascript
class ScreenShakeWithVisualFeedback {
  constructor() {
    this.shake = new ScreenShakeManager();
    this.flash = new HitFlash();
    this.damageNumbers = new DamageNumberSystem();
    this.combo = new ComboVisualFeedback();
    this.particles = new FeedbackParticleSystem();
  }

  // Combined effects
  triggerExplosion(x, y) {
    this.shake.shake(20, 0.8);
    this.flash.flash('#FFF', 0.2);
    this.particles.hitImpact(x, y, '#FFA500');
    this.damageNumbers.spawn(x, y, 100, true);
  }

  triggerPowerup(x, y) {
    this.shake.shake(10, 0.5);
    this.flash.flash('#00FFFF', 0.1);
    this.particles.powerupCollect(x, y);
  }

  triggerScore(x, y, amount) {
    this.shake.shake(3, 0.2);
    this.damageNumbers.spawn(x, y, amount);
  }

  triggerCombo(comboCount) {
    this.shake.shake(Math.min(15, comboCount * 0.5), 0.5);
    this.combo.addHit();
  }

  triggerGameOver() {
    this.shake.shake(10, 1.0);
    this.flash.flash('#000', 0.5);
    this.particles.clear();
  }

  triggerWin(x, y) {
    this.shake.shake(5, 0.8);
    this.flash.flash('#FFD700', 0.3);
    this.particles.levelUp(x, y);
  }

  // Update all effects
  update(deltaTime) {
    this.shake.update(deltaTime);
    this.flash.update(deltaTime);
    this.damageNumbers.update(deltaTime);
    this.combo.update(deltaTime);
    this.particles.update(deltaTime);
  }

  // Draw all effects
  draw(ctx, width, height) {
    this.shake.draw(ctx, width, height);
    this.flash.draw(ctx, width, height);
    this.damageNumbers.draw(ctx);
    this.combo.draw(ctx);
    this.particles.draw(ctx);
  }

  // Get screen offset
  getScreenOffset() {
    return this.shake.getOffset();
  }
}
```

---

## UI Animation Patterns

### 1. UI Animation Manager

```javascript
class UIAnimationManager {
  constructor() {
    this.animations = new Map();
    this.activeAnimations = new Set();
    this.defaultDuration = 0.3;
    this.defaultEasing = 'easeOut';
  }

  // Element animations
  animate(element, keyframes, options = {}) {
    const animation = element.animate(keyframes, {
      duration: options.duration || this.defaultDuration * 1000,
      easing: options.easing || this.defaultEasing,
      fill: 'forwards'
    });

    this.activeAnimations.add(animation);
    
    animation.onfinish = () => {
      this.activeAnimations.delete(animation);
    };

    return animation;
  }

  // Common UI animations
  fadeIn(element, duration = 0.3, delay = 0) {
    element.style.opacity = '0';
    return this.animate(element, [
      { opacity: 0, offset: 0 },
      { opacity: 1, offset: 1 }
    ], { duration, delay });
  }

  fadeOut(element, duration = 0.3, delay = 0) {
    return this.animate(element, [
      { opacity: 1, offset: 0 },
      { opacity: 0, offset: 1 }
    ], { duration, delay });
  }

  slideIn(element, direction = 'bottom', duration = 0.3, delay = 0) {
    const transform = {
      top: 'translateY(-100%)',
      right: 'translateX(100%)',
      bottom: 'translateY(100%)',
      left: 'translateX(-100%)'
    };

    element.style.transform = transform[direction];
    return this.animate(element, [
      { transform: transform[direction], opacity: 0, offset: 0 },
      { transform: 'translate(0, 0)', opacity: 1, offset: 1 }
    ], { duration, delay });
  }

  slideOut(element, direction = 'bottom', duration = 0.3, delay = 0) {
    const transform = {
      top: 'translateY(-100%)',
      right: 'translateX(100%)',
      bottom: 'translateY(100%)',
      left: 'translateX(-100%)'
    };

    return this.animate(element, [
      { transform: 'translate(0, 0)', opacity: 1, offset: 0 },
      { transform: transform[direction], opacity: 0, offset: 1 }
    ], { duration, delay });
  }

  scale(element, scale = 1.1, duration = 0.2, delay = 0) {
    return this.animate(element, [
      { transform: 'scale(1)', offset: 0 },
      { transform: `scale(${scale})`, offset: 0.5 },
      { transform: 'scale(1)', offset: 1 }
    ], { duration, delay });
  }

  bounce(element, duration = 0.5, delay = 0) {
    return this.animate(element, [
      { transform: 'translateY(0)', offset: 0 },
      { transform: 'translateY(-20px)', offset: 0.5 },
      { transform: 'translateY(0)', offset: 1 }
    ], { duration, delay });
  }

  pulse(element, duration = 0.3, delay = 0) {
    return this.animate(element, [
      { transform: 'scale(1)', offset: 0 },
      { transform: 'scale(1.05)', offset: 0.5 },
      { transform: 'scale(1)', offset: 1 }
    ], { duration, delay });
  }

  // Stagger animations
  stagger(elements, animation, options = {}) {
    elements.forEach((element, index) => {
      const delay = index * (options.interval || 0.1);
      animation(element, { ...options, delay });
    });
  }

  // Button hover effects
  registerButtonHover(button) {
    button.addEventListener('mouseenter', () => {
      this.scale(button, 1.05, 0.2);
    });

    button.addEventListener('mouseleave', () => {
      this.scale(button, 1, 0.2);
    });

    button.addEventListener('click', () => {
      this.scale(button, 0.95, 0.1);
    });
  }

  // Menu entry animations
  animateMenuEntry(entry, index, total) {
    const delay = index * 0.1;
    return this.slideIn(entry, 'right', 0.4, delay);
  }

  // Clear animations
  clear() {
    this.activeAnimations.forEach(animation => animation.cancel());
    this.activeAnimations.clear();
  }

  // Get active animation count
  getActiveCount() {
    return this.activeAnimations.size;
  }
}
```

### 2. UI Transition System

```javascript
class UITransitionSystem {
  constructor() {
    this.transitions = new Map();
    this.activeTransitions = [];
  }

  addTransition(name, fromState, toState, options = {}) {
    this.transitions.set(name, {
      fromState,
      toState,
      ...options
    });
  }

  transition(name) {
    const transition = this.transitions.get(name);
    if (!transition) return null;

    this.activeTransitions.push(transition);

    // Play transition animations
    if (transition.onEnter) {
      transition.onEnter();
    }

    // Return promise for async transition
    return new Promise(resolve => {
      setTimeout(() => {
        if (transition.onExit) {
          transition.onExit();
        }
        this.activeTransitions = this.activeTransitions.filter(t => t !== transition);
        resolve();
      }, transition.duration || 500);
    });
  }

  // Screen transitions
  fadeTransition(duration = 0.5) {
    const overlay = document.createElement('div');
    overlay.style.position = 'fixed';
    overlay.style.top = '0';
    overlay.style.left = '0';
    overlay.style.width = '100%';
    overlay.style.height = '100%';
    overlay.style.backgroundColor = '#000';
    overlay.style.opacity = '0';
    overlay.style.transition = `opacity ${duration}s`;
    overlay.style.zIndex = '9999';
    document.body.appendChild(overlay);

    return new Promise(resolve => {
      setTimeout(() => {
        overlay.style.opacity = '1';
      }, 50);

      setTimeout(() => {
        if (overlay.parentNode) {
          overlay.parentNode.removeChild(overlay);
        }
        resolve();
      }, duration * 1000);
    });
  }

  slideTransition(direction = 'left', duration = 0.5) {
    return new Promise(resolve => {
      const screens = document.querySelectorAll('.screen');
      screens.forEach(screen => {
        screen.style.transition = `transform ${duration}s`;
        screen.style.transform = `translate${direction === 'left' ? 'X' : 'Y'}(-100%)`;
      });

      setTimeout(() => {
        resolve();
      }, duration * 1000);
    });
  }

  // Menu transitions
  animateMenuTransition(menu, show) {
    if (show) {
      this.fadeIn(menu);
    } else {
      this.fadeOut(menu);
    }
  }

  // Modal transitions
  animateModalTransition(modal, show) {
    if (show) {
      this.scale(modal, 1.1, 0.3);
    } else {
      this.scale(modal, 0.9, 0.2);
    }
  }

  // Get active transition count
  getActiveCount() {
    return this.activeTransitions.length;
  }
}
```

### 3. Progress Bar Animation

```javascript
class ProgressBarAnimation {
  constructor(element, options = {}) {
    this.element = element;
    this.progress = 0;
    this.targetProgress = 0;
    this.duration = options.duration || 1.0;
    this.startTime = 0;
    this.isPlaying = false;
    this.onComplete = null;
    this.onUpdate = null;
  }

  play(targetProgress, duration = null) {
    this.targetProgress = targetProgress;
    this.startTime = performance.now() / 1000;
    this.duration = duration || this.duration;
    this.isPlaying = true;

    if (!this.element.dataset.initialProgress) {
      this.element.dataset.initialProgress = this.element.dataset.progress || '0';
    }
  }

  update(deltaTime) {
    if (!this.isPlaying) return;

    const elapsed = (performance.now() / 1000) - this.startTime;
    const t = Math.min(elapsed / this.duration, 1);

    // Smooth progress update
    this.progress = this.lerp(
      parseFloat(this.element.dataset.initialProgress || '0'),
      this.targetProgress,
      this.smoothStep(0, 1, t)
    );

    this.updateBar();
    this.element.dataset.progress = this.progress;

    if (this.onUpdate) {
      this.onUpdate(this.progress);
    }

    if (t >= 1 && this.isPlaying) {
      this.isPlaying = false;
      if (this.onComplete) {
        this.onComplete(this.progress);
      }
    }
  }

  updateBar() {
    this.element.style.width = `${this.progress}%`;
    
    // Color changes based on progress
    if (this.progress < 30) {
      this.element.style.backgroundColor = '#FF4500';
    } else if (this.progress < 70) {
      this.element.style.backgroundColor = '#FFD700';
    } else {
      this.element.style.backgroundColor = '#32CD32';
    }
  }

  lerp(start, end, t) {
    return start + (end - start) * t;
  }

  smoothStep(min, max, t) {
    const t2 = t * t;
    const t3 = t2 * t;
    return min + (max - min) * (3 * t2 - 2 * t3);
  }

  setProgress(progress) {
    this.progress = progress;
    this.targetProgress = progress;
    this.updateBar();
  }

  reset() {
    this.progress = 0;
    this.targetProgress = 0;
    this.isPlaying = false;
    this.element.dataset.progress = '0';
    this.element.style.width = '0%';
  }
}
```

### 4. Floating Text Animation

```javascript
class FloatingTextAnimation {
  constructor() {
    this.texts = [];
    this.maxTexts = 20;
  }

  spawn(x, y, text, options = {}) {
    if (this.texts.length >= this.maxTexts) {
      this.texts.shift();
    }

    this.texts.push({
      x,
      y,
      text,
      life: options.life || 1.0,
      maxLife: options.life || 1.0,
      velocityY: options.velocityY || -50,
      velocityX: options.velocityX || (Math.random() - 0.5) * 30,
      scale: options.scale || 1,
      scaleChange: options.scaleChange || -0.5,
      color: options.color || '#FFF',
      fontSize: options.fontSize || 24,
      align: options.align || 'center'
    });
  }

  update(deltaTime) {
    for (let i = this.texts.length - 1; i >= 0; i--) {
      const t = this.texts[i];
      
      t.x += t.velocityX * deltaTime;
      t.y += t.velocityY * deltaTime;
      t.life -= deltaTime;
      t.scale += t.scaleChange * deltaTime;

      if (t.life <= 0 || t.scale <= 0) {
        this.texts.splice(i, 1);
      }
    }
  }

  draw(ctx) {
    this.texts.forEach(t => {
      if (t.life <= 0) return;

      ctx.save();
      ctx.translate(t.x, t.y);
      ctx.scale(t.scale, t.scale);
      ctx.globalAlpha = t.life;

      ctx.font = `bold ${t.fontSize}px Arial`;
      ctx.fillStyle = t.color;
      ctx.textAlign = t.align;
      ctx.textBaseline = 'middle';
      ctx.strokeStyle = '#000';
      ctx.lineWidth = 2;

      ctx.strokeText(t.text, 0, 0);
      ctx.fillText(t.text, 0, 0);

      ctx.restore();
    });
  }

  clear() {
    this.texts = [];
  }
}
```

### 5. UI Feedback Manager

```javascript
class UIFeedbackManager {
  constructor() {
    this.shake = new ScreenShakeManager();
    this.flash = new HitFlash();
    this.floatingText = new FloatingTextAnimation();
    this.uiAnimations = new UIAnimationManager();
    this.audioManager = null;
  }

  setAudioManager(audioManager) {
    this.audioManager = audioManager;
  }

  // Game events
  triggerScore(x, y, amount, isCritical = false) {
    this.shake.shake(3, 0.2);
    this.floatingText.spawn(x, y, amount, {
      color: isCritical ? '#FFD700' : '#FFF',
      fontSize: isCritical ? 36 : 24,
      velocityY: -80
    });
    
    if (this.audioManager) {
      this.audioManager.playSound('score', { pitch: 1.2 });
    }
  }

  triggerPowerup(x, y, type) {
    this.shake.shake(10, 0.5);
    this.flash.flash('#00FFFF', 0.2);
    this.floatingText.spawn(x, y, type, {
      color: '#00FFFF',
      fontSize: 48,
      velocityY: -100
    });
    
    if (this.audioManager) {
      this.audioManager.playSound('powerup', { pitch: 1.5 });
    }
  }

  triggerCombo(comboCount) {
    this.shake.shake(Math.min(15, comboCount * 0.5), 0.5);
    this.floatingText.spawn(100, 50, `${comboCount} COMBO`, {
      color: this.getComboColor(comboCount),
      fontSize: 48,
      velocityY: -60
    });
    
    if (this.audioManager) {
      this.audioManager.playSound('combo', { pitch: 1.3 });
    }
  }

  triggerGameOver() {
    this.shake.shake(10, 1.0);
    this.flash.flash('#000', 0.5);
    
    if (this.audioManager) {
      this.audioManager.playMusic('gameover');
    }
  }

  triggerWin(x, y) {
    this.shake.shake(5, 0.8);
    this.flash.flash('#FFD700', 0.3);
    this.floatingText.spawn(x, y, 'YOU WIN!', {
      color: '#FFD700',
      fontSize: 64,
      velocityY: -120
    });
    
    if (this.audioManager) {
      this.audioManager.playMusic('win');
    }
  }

  triggerUIClick(x, y) {
    this.shake.shake(2, 0.1);
    this.floatingText.spawn(x, y, 'CLICK', {
      color: '#FFF',
      fontSize: 20,
      velocityY: -30,
      life: 0.5
    });
    
    if (this.audioManager) {
      this.audioManager.playSound('click', { pitch: 1.2 });
    }
  }

  triggerDamage(x, y, amount) {
    this.shake.shake(5, 0.3);
    this.flash.flash('#FF0000', 0.1);
    this.floatingText.spawn(x, y, amount, {
      color: '#FF4500',
      fontSize: 36,
      velocityY: -60,
      life: 0.8
    });
    
    if (this.audioManager) {
      this.audioManager.playSound('hit', { pitch: 0.8 });
    }
  }

  // UI animations
  animateMenuShow(menu) {
    this.uiAnimations.fadeIn(menu);
    this.uiAnimations.stagger(menu.querySelectorAll('.menu-item'), 
      (el, opts) => this.uiAnimations.slideIn(el, 'right', 0.4, 0), 
      { interval: 0.1 }
    );
  }

  animateMenuHide(menu) {
    this.uiAnimations.fadeOut(menu);
    this.uiAnimations.stagger(menu.querySelectorAll('.menu-item'), 
      (el, opts) => this.uiAnimations.slideOut(el, 'right', 0.4, 0), 
      { interval: 0.1 }
    );
  }

  animateModalShow(modal) {
    this.uiAnimations.scale(modal, 1.1, 0.3);
  }

  animateModalHide(modal) {
    this.uiAnimations.scale(modal, 0.9, 0.2);
  }

  // Update all animations
  update(deltaTime) {
    this.shake.update(deltaTime);
    this.flash.update(deltaTime);
    this.floatingText.update(deltaTime);
    this.uiAnimations.update(deltaTime);
  }

  // Draw all animations
  draw(ctx, width, height) {
    this.shake.draw(ctx, width, height);
    this.flash.draw(ctx, width, height);
    this.floatingText.draw(ctx);
  }

  // Get screen offset
  getScreenOffset() {
    return this.shake.getOffset();
  }

  getComboColor(comboCount) {
    if (comboCount < 5) return '#FFF';
    if (comboCount < 10) return '#4169E1';
    if (comboCount < 20) return '#FFD700';
    if (comboCount < 50) return '#FF4500';
    return '#DC143C';
  }
}
```
