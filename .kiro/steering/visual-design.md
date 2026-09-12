# Ghosty Visual Design Steering File

This steering file defines Ghosty character animations, wall textures, and background parallax effects for the Ghosty game.

## Table of Contents

- [Ghosty Character Animations](#ghosty-character-animations)
- [Wall Textures](#wall-textures)
- [Background Parallax Effects](#background-parallax-effects)
- [Visual Systems Integration](#visual-systems-integration)

---

## Ghosty Character Animations

### 1. Ghosty Animation States

```javascript
const GhostyAnimationState = {
  IDLE: 'idle',
  FLYING: 'flying',
  GLIDING: 'gliding',
  FLAPPING: 'flapping',
  SINKING: 'sinking',
  HIT: 'hit',
  POOF: 'poof'
};

const GhostyAnimations = {
  idle: {
    frames: [0],
    duration: 1000,
    loop: true,
    description: 'Floating gently in place'
  },
  flying: {
    frames: [0, 1, 2, 1],
    duration: 500,
    loop: true,
    description: 'Normal flight motion'
  },
  gliding: {
    frames: [2],
    duration: 500,
    loop: true,
    description: 'Slow descending glide'
  },
  flapping: {
    frames: [1, 3, 4, 3],
    duration: 300,
    loop: true,
    description: 'Aggressive upward motion'
  },
  sinking: {
    frames: [2, 0],
    duration: 200,
    loop: true,
    description: 'Falling with reduced upward force'
  },
  hit: {
    frames: [5, 6, 7],
    duration: 150,
    loop: false,
    description: 'Collision reaction'
  },
  poof: {
    frames: [8, 9, 10],
    duration: 100,
    loop: false,
    description: 'Disappearing effect'
  }
};
```

### 2. Ghosty Animation Controller

```javascript
class GhostyAnimationController {
  constructor() {
    this.animationManager = new AnimationManager();
    this.state = GhostyAnimationState.IDLE;
    this.facingRight = true;
    this.rotation = 0;
    this.wobble = 0;
    this.wobbleSpeed = 2;
    this.wobbleAmount = 0.05;
  }

  addAnimations(spriteSheet) {
    GhostyAnimations.idle.frames.forEach((frame, index) => {
      this.animationManager.addAnimation(
        GhostyAnimationState.IDLE + '_' + index,
        new SpriteAnimation([frame], 1000, true)
      );
    });

    this.animationManager.addAnimation(
      GhostyAnimationState.FLYING,
      new SpriteAnimation(GhostyAnimations.flying.frames, 125, true)
    );

    this.animationManager.addAnimation(
      GhostyAnimationState.FLAPPING,
      new SpriteAnimation(GhostyAnimations.flapping.frames, 75, true)
    );

    this.animationManager.addAnimation(
      GhostyAnimationState.GLIDING,
      new SpriteAnimation(GhostyAnimations.gliding.frames, 500, true)
    );

    this.animationManager.addAnimation(
      GhostyAnimationState.SINKING,
      new SpriteAnimation(GhostyAnimations.sinking.frames, 100, true)
    );

    this.animationManager.addAnimation(
      GhostyAnimationState.HIT,
      new SpriteAnimation(GhostyAnimations.hit.frames, 150, false)
    );

    this.animationManager.addAnimation(
      GhostyAnimationState.POOF,
      new SpriteAnimation(GhostyAnimations.poof.frames, 100, false)
    );
  }

  updateState(velocityY, isHovering, deltaTime) {
    const prevState = this.state;

    if (this.state === GhostyAnimationState.HIT || 
        this.state === GhostyAnimationState.POOF) {
      return;
    }

    // Determine new state based on physics
    if (isHovering && velocityY < -50) {
      this.state = GhostyAnimationState.FLAPPING;
    } else if (velocityY < -10) {
      this.state = GhostyAnimationState.FLYING;
    } else if (velocityY > 100) {
      this.state = GhostyAnimationState.SINKING;
    } else if (velocityY > 50) {
      this.state = GhostyAnimationState.GLIDING;
    } else {
      this.state = GhostyAnimationState.IDLE;
    }

    // Update animation if state changed
    if (this.state !== prevState) {
      this.animationManager.play(this.state);
    }

    // Update animation
    this.animationManager.update(this.state, deltaTime);
  }

  updateRotation(deltaTime, velocityY) {
    // Calculate target rotation based on velocity
    let targetRotation = 0;

    if (velocityY < -100) {
      targetRotation = -30 * Math.PI / 180;
    } else if (velocityY > 100) {
      targetRotation = 45 * Math.PI / 180;
    }

    // Smooth rotation
    this.rotation += (targetRotation - this.rotation) * 0.1;
  }

  updateWobble(deltaTime) {
    this.wobble += this.wobbleSpeed * deltaTime;
  }

  getAnimationFrame() {
    return this.animationManager.getCurrentFrameKey(this.state);
  }

  isAnimationFinished() {
    return this.animationManager.getAnimation(this.state)?.isFinished() || false;
  }

  playPoof() {
    this.state = GhostyAnimationState.POOF;
    this.animationManager.play(GhostyAnimationState.POOF);
  }

  playHit() {
    this.state = GhostyAnimationState.HIT;
    this.animationManager.play(GhostyAnimationState.HIT);
  }
}
```

### 3. Ghosty Character Renderer

```javascript
class GhostyRenderer {
  constructor(spriteManager) {
    this.spriteManager = spriteManager;
    this.particleSystem = new ParticleSystem(30);
  }

  draw(ctx, ghosty, animationController) {
    const frameKey = animationController.getAnimationFrame();
    const sprite = this.spriteManager.get(frameKey);

    if (!sprite) return;

    // Calculate wobble effect
    const wobbleX = Math.sin(animationController.wobble) * 
      animationController.wobbleAmount * ghosty.width;
    const wobbleY = Math.cos(animationController.wobble) * 
      animationController.wobbleAmount * ghosty.height;

    ctx.save();
    ctx.translate(
      ghosty.x + ghosty.width / 2 + wobbleX,
      ghosty.y + ghosty.height / 2 + wobbleY
    );

    // Rotate based on velocity
    ctx.rotate(animationController.rotation);

    // Flip based on facing direction
    if (!animationController.facingRight) {
      ctx.scale(-1, 1);
    }

    // Draw ghost
    const scale = 1 + Math.sin(animationController.wobble) * 0.05;
    ctx.scale(scale, scale);

    ctx.drawImage(
      sprite,
      -ghosty.width / 2,
      -ghosty.height / 2,
      ghosty.width,
      ghosty.height
    );

    // Draw eyes
    this.drawEyes(ctx, ghosty);

    // Draw tail
    this.drawTail(ctx, ghosty, animationController);

    ctx.restore();

    // Draw particles
    this.drawParticles(ctx, ghosty, animationController);
  }

  drawEyes(ctx, ghosty) {
    ctx.fillStyle = '#000';
    
    // Left eye
    ctx.beginPath();
    ctx.ellipse(-8, -5, 3, 5, 0, 0, Math.PI * 2);
    ctx.fill();

    // Right eye
    ctx.beginPath();
    ctx.ellipse(8, -5, 3, 5, 0, 0, Math.PI * 2);
    ctx.fill();

    // Pupils (look in direction of movement)
    const lookX = Math.cos(ghosty.velocityY) * 1;
    const lookY = Math.sin(ghosty.velocityY) * 1;

    ctx.fillStyle = '#FFF';
    ctx.beginPath();
    ctx.arc(-8 + lookX, -5 + lookY, 1, 0, Math.PI * 2);
    ctx.fill();
    ctx.beginPath();
    ctx.arc(8 + lookX, -5 + lookY, 1, 0, Math.PI * 2);
    ctx.fill();
  }

  drawTail(ctx, ghosty, animationController) {
    const tailLength = 15;
    const tailSegments = 3;
    const time = animationController.wobble;

    ctx.strokeStyle = '#FFF';
    ctx.lineWidth = 3;
    ctx.lineCap = 'round';

    ctx.beginPath();
    ctx.moveTo(0, ghosty.height / 2);

    for (let i = 1; i <= tailSegments; i++) {
      const offset = i * tailLength;
      const sway = Math.sin(time + i * 0.5) * 5;
      const length = tailLength * (1 - i / tailSegments);
      
      ctx.lineTo(
        sway,
        ghosty.height / 2 + length + offset
      );
    }

    ctx.stroke();
  }

  drawParticles(ctx, ghosty, animationController) {
    // Emit particles from ghost
    if (animationController.state !== GhostyAnimationState.POOF) {
      if (Math.random() < 0.05) {
        this.particleSystem.emit(
          ghosty.x + ghosty.width / 2,
          ghosty.y + ghosty.height,
          {
            speed: 20,
            life: 0.5,
            color: '#FFF',
            scale: 2 + Math.random() * 2,
            shape: 'circle'
          }
        );
      }
    }

    // Draw particles
    this.particleSystem.update(1 / 60);
    this.particleSystem.draw(ctx);
  }

  clearParticles() {
    this.particleSystem.clear();
  }
}
```

---

## Wall Textures

### 1. Wall Texture Manager

```javascript
class WallTextureManager {
  constructor() {
    this.textures = new Map();
    this.cachedCanvases = new Map();
  }

  createWallTexture(type, width, height, options = {}) {
    const key = `${type}_${width}_${height}_${JSON.stringify(options)}`;
    
    if (this.cachedCanvases.has(key)) {
      return this.cachedCanvases.get(key);
    }

    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');

    // Draw base
    ctx.fillStyle = this.getBaseColor(type, options);
    ctx.fillRect(0, 0, width, height);

    // Add texture details
    this.addTextureDetails(ctx, type, width, height, options);

    // Add borders
    this.addBorders(ctx, type, width, height, options);

    this.cachedCanvases.set(key, canvas);
    return canvas;
  }

  getBaseColor(type, options) {
    const colors = {
      default: options.color || '#228B22',
      glowing: options.glowColor || '#4169E1',
      spike: options.spikeColor || '#DC143C',
      bonus: options.bonusColor || '#FFD700'
    };
    return colors[type] || colors.default;
  }

  addTextureDetails(ctx, type, width, height, options) {
    const textureType = options.texture || 'stripes';

    switch (textureType) {
      case 'stripes':
        this.drawStripes(ctx, width, height, options);
        break;
      case 'dots':
        this.drawDots(ctx, width, height, options);
        break;
      case 'grunge':
        this.drawGrunge(ctx, width, height, options);
        break;
      case 'bricks':
        this.drawBricks(ctx, width, height, options);
        break;
    }
  }

  drawStripes(ctx, width, height, options) {
    ctx.fillStyle = this.getStripeColor(options);
    const stripeWidth = options.stripeWidth || 4;

    for (let x = 0; x < width; x += stripeWidth * 4) {
      for (let y = 0; y < height; y += stripeWidth) {
        ctx.fillRect(x + y % (stripeWidth * 4), y, stripeWidth, stripeWidth);
      }
    }
  }

  drawDots(ctx, width, height, options) {
    ctx.fillStyle = this.getDotColor(options);
    const dotSize = options.dotSize || 3;
    const spacing = options.dotSpacing || 10;

    for (let x = dotSize; x < width; x += spacing) {
      for (let y = dotSize; y < height; y += spacing) {
        if (Math.random() > 0.2) {
          ctx.beginPath();
          ctx.arc(x, y, dotSize / 2, 0, Math.PI * 2);
          ctx.fill();
        }
      }
    }
  }

  drawGrunge(ctx, width, height, options) {
    ctx.fillStyle = 'rgba(0, 0, 0, 0.1)';
    const count = (width * height) / 100;

    for (let i = 0; i < count; i++) {
      const x = Math.random() * width;
      const y = Math.random() * height;
      const size = Math.random() * 2;

      ctx.fillRect(x, y, size, size);
    }
  }

  drawBricks(ctx, width, height, options) {
    const brickWidth = options.brickWidth || 30;
    const brickHeight = options.brickHeight || 15;
    const mortarWidth = options.mortarWidth || 2;

    ctx.fillStyle = '#000';
    const rows = Math.ceil(height / (brickHeight + mortarWidth));

    for (let row = 0; row < rows; row++) {
      const y = row * (brickHeight + mortarWidth);
      const offset = (row % 2) * (brickWidth / 2);

      for (let x = -brickWidth; x < width; x += brickWidth + mortarWidth) {
        ctx.fillRect(x + offset, y, brickWidth, brickHeight);
      }
    }
  }

  getStripeColor(options) {
    return `rgba(255, 255, 255, ${options.stripeOpacity || 0.1})`;
  }

  getDotColor(options) {
    return `rgba(255, 255, 255, ${options.dotOpacity || 0.2})`;
  }

  addBorders(ctx, type, width, height, options) {
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.3)';
    ctx.lineWidth = 2;
    ctx.strokeRect(0, 0, width, height);

    // Add highlight
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)';
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(0, 0);
    ctx.lineTo(width, 0);
    ctx.lineTo(0, height);
    ctx.stroke();
  }

  getTexture(type, width, height, options) {
    const key = `${type}_${width}_${height}_${JSON.stringify(options)}`;
    return this.cachedCanvases.get(key);
  }

  createAllTextures() {
    return {
      default: {
        small: this.createWallTexture('default', 60, 100),
        large: this.createWallTexture('default', 60, 200)
      },
      glowing: {
        small: this.createWallTexture('glowing', 60, 100),
        large: this.createWallTexture('glowing', 60, 200)
      },
      spike: {
        small: this.createWallTexture('spike', 60, 100),
        large: this.createWallTexture('spike', 60, 200)
      },
      bonus: {
        small: this.createWallTexture('bonus', 60, 100),
        large: this.createWallTexture('bonus', 60, 200)
      }
    };
  }
}
```

### 2. Wall Renderer

```javascript
class WallRenderer {
  constructor(textureManager) {
    this.textureManager = textureManager;
    this.textures = textureManager.createAllTextures();
  }

  drawWall(ctx, wall, options = {}) {
    const texture = this.textures[wall.type]?.small;
    
    if (texture) {
      ctx.drawImage(texture, wall.x, wall.y, wall.width, wall.height);
    } else {
      ctx.fillStyle = this.textureManager.getBaseColor(wall.type, options);
      ctx.fillRect(wall.x, wall.y, wall.width, wall.height);
    }

    // Add glow effect for special walls
    if (wall.type === 'glowing' || wall.type === 'bonus') {
      this.drawGlow(ctx, wall, options);
    }

    // Add spikes for spike walls
    if (wall.type === 'spike') {
      this.drawSpikes(ctx, wall, options);
    }

    // Add bonus marker
    if (wall.type === 'bonus') {
      this.drawBonusMarker(ctx, wall, options);
    }
  }

  drawGlow(ctx, wall, options) {
    ctx.shadowBlur = options.glowBlur || 20;
    ctx.shadowColor = options.glowColor || 
      (wall.type === 'bonus' ? '#FFD700' : '#4169E1');

    ctx.fillStyle = 'rgba(255, 255, 255, 0.3)';
    ctx.fillRect(wall.x, wall.y, wall.width, wall.height);

    ctx.shadowBlur = 0;
  }

  drawSpikes(ctx, wall, options) {
    ctx.fillStyle = '#FFF';
    const spikeCount = Math.floor(wall.height / 20);

    for (let i = 0; i < spikeCount; i++) {
      const y = wall.y + i * 20;
      ctx.beginPath();
      ctx.moveTo(wall.x, y);
      ctx.lineTo(wall.x + wall.width / 2, y + 10);
      ctx.lineTo(wall.x + wall.width, y);
      ctx.closePath();
      ctx.fill();
    }
  }

  drawBonusMarker(ctx, wall, options) {
    ctx.fillStyle = '#FFF';
    ctx.font = 'bold 14px Arial';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';

    ctx.fillText('?', wall.x + wall.width / 2, wall.y + wall.height / 2);
  }

  drawPipe(ctx, pipe) {
    const texture = this.textures.default?.small;

    if (texture) {
      ctx.drawImage(texture, pipe.x, pipe.y, pipe.width, pipe.height);
    } else {
      ctx.fillStyle = '#228B22';
      ctx.fillRect(pipe.x, pipe.y, pipe.width, pipe.height);
    }

    // Add pipe cap
    ctx.fillStyle = '#32CD32';
    const capHeight = 10;
    const capY = pipe.type === 'top' 
      ? pipe.height - capHeight 
      : pipe.y;

    ctx.fillRect(pipe.x - 2, capY, pipe.width + 4, capHeight);
  }
}
```

---

## Background Parallax Effects

### 1. Parallax Layer Manager

```javascript
class ParallaxLayerManager {
  constructor(width, height) {
    this.width = width;
    this.height = height;
    this.layers = [];
    this.cameraX = 0;
    this.cameraY = 0;
  }

  addLayer(image, speed, yOffset = 0, repeat = true, type = 'background') {
    this.layers.push({
      image,
      speed,
      yOffset,
      repeat,
      x: 0,
      type
    });
  }

  addGradientLayer(colors, speed, yOffset = 0, type = 'sky') {
    const canvas = document.createElement('canvas');
    canvas.width = this.width;
    canvas.height = this.height;
    const ctx = canvas.getContext('2d');

    // Create gradient
    const gradient = ctx.createLinearGradient(0, 0, 0, this.height);
    colors.forEach((color, index) => {
      gradient.addColorStop(index / (colors.length - 1), color);
    });

    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, this.width, this.height);

    this.addLayer(canvas, speed, yOffset, false, type);
  }

  addCloudLayer(speed = 0.2) {
    const canvas = document.createElement('canvas');
    canvas.width = this.width;
    canvas.height = this.height;
    const ctx = canvas.getContext('2d');

    // Draw random clouds
    ctx.fillStyle = 'rgba(255, 255, 255, 0.1)';
    const cloudCount = 10;
    const cloudPositions = [];

    for (let i = 0; i < cloudCount; i++) {
      cloudPositions.push({
        x: Math.random() * this.width,
        y: Math.random() * (this.height * 0.6),
        size: 30 + Math.random() * 50
      });
    }

    cloudPositions.forEach(cloud => {
      ctx.beginPath();
      ctx.arc(cloud.x, cloud.y, cloud.size, 0, Math.PI * 2);
      ctx.arc(cloud.x + cloud.size * 0.5, cloud.y - cloud.size * 0.3, cloud.size * 0.8, 0, Math.PI * 2);
      ctx.arc(cloud.x - cloud.size * 0.5, cloud.y - cloud.size * 0.3, cloud.size * 0.8, 0, Math.PI * 2);
      ctx.fill();
    });

    this.addLayer(canvas, speed, 0, true, 'clouds');
  }

  addStarLayer(speed = 0.05) {
    const canvas = document.createElement('canvas');
    canvas.width = this.width;
    canvas.height = this.height;
    const ctx = canvas.getContext('2d');

    // Draw stars
    ctx.fillStyle = '#FFF';
    const starCount = 50;

    for (let i = 0; i < starCount; i++) {
      const x = Math.random() * this.width;
      const y = Math.random() * this.height;
      const size = Math.random() * 2;

      ctx.fillRect(x, y, size, size);
    }

    this.addLayer(canvas, speed, 0, true, 'stars');
  }

  addGroundLayer(speed = 1) {
    const canvas = document.createElement('canvas');
    canvas.width = this.width * 2;
    canvas.height = 50;
    const ctx = canvas.getContext('2d');

    // Draw ground pattern
    ctx.fillStyle = '#2a2a3a';
    ctx.fillRect(0, 0, this.width * 2, 50);

    // Add ground details
    ctx.fillStyle = '#1a1a2a';
    for (let x = 0; x < this.width * 2; x += 40) {
      ctx.fillRect(x, 0, 20, 10);
      ctx.fillRect(x + 20, 25, 20, 10);
    }

    this.addLayer(canvas, speed, this.height - 50, true, 'ground');
  }

  update(deltaTime, cameraX, cameraY) {
    this.cameraX = cameraX;
    this.cameraY = cameraY;

    this.layers.forEach(layer => {
      const moveAmount = layer.speed * deltaTime * 100;
      layer.x -= moveAmount;

      if (layer.repeat) {
        if (layer.x <= -this.width) {
          layer.x = 0;
        }
      }
    });
  }

  draw(ctx) {
    this.layers.forEach(layer => {
      if (!layer.image) return;

      let x = layer.x - this.cameraX * (layer.speed || 0);
      
      if (layer.repeat) {
        while (x < this.width) {
          ctx.drawImage(layer.image, x, layer.yOffset + this.cameraY);
          x += this.width;
        }
      } else {
        ctx.drawImage(layer.image, x, layer.yOffset + this.cameraY);
      }
    });
  }
}
```

### 2. Dynamic Background Manager

```javascript
class DynamicBackgroundManager {
  constructor(width, height) {
    this.width = width;
    this.height = height;
    this.parallax = new ParallaxLayerManager(width, height);
    this.time = 0;
    this.dayCycle = 0;
  }

  createDayCycleBackground() {
    // Sky gradients for different times of day
    this.skyColors = {
      morning: ['#87CEEB', '#FFE4B5', '#FFFACD'],
      noon: ['#87CEEB', '#E0F7FA', '#FFFFFF'],
      evening: ['#FF6347', '#FF8C00', '#FFD700'],
      night: ['#000033', '#000066', '#000099']
    };

    this.parallax.addGradientLayer(this.skyColors.noon, 0, 0, 'sky');
    this.parallax.addCloudLayer(0.2);
    this.parallax.addStarLayer(0.05);
  }

  createSpaceBackground() {
    // Deep space with distant stars and nebulas
    this.parallax.addGradientLayer(['#000000', '#1a0033', '#330066'], 0, 0, 'space');
    
    // Add distant stars (slower parallax)
    const starCanvas = document.createElement('canvas');
    starCanvas.width = this.width;
    starCanvas.height = this.height;
    const ctx = starCanvas.getContext('2d');

    ctx.fillStyle = '#FFF';
    for (let i = 0; i < 100; i++) {
      ctx.fillRect(
        Math.random() * this.width,
        Math.random() * this.height,
        Math.random() * 2,
        Math.random() * 2
      );
    }

    this.parallax.addLayer(starCanvas, 0.01, 0, true, 'stars');
  }

  createForestBackground() {
    // Layered forest with trees at different depths
    this.parallax.addGradientLayer(['#87CEEB', '#E0F7FA'], 0, 0, 'sky');

    // Distant mountains (slowest)
    const mountainCanvas = document.createElement('canvas');
    mountainCanvas.width = this.width;
    mountainCanvas.height = this.height;
    const mCtx = mountainCanvas.getContext('2d');

    mCtx.fillStyle = '#4A69BD';
    mCtx.beginPath();
    for (let x = 0; x <= this.width; x += 50) {
      mCtx.lineTo(x, this.height - 100 - Math.random() * 50);
    }
    mCtx.lineTo(this.width, this.height);
    mCtx.lineTo(0, this.height);
    mCtx.closePath();
    mCtx.fill();

    this.parallax.addLayer(mountainCanvas, 0.1, 0, true, 'mountains');

    // Trees (medium speed)
    const treeCanvas = document.createElement('canvas');
    treeCanvas.width = this.width;
    treeCanvas.height = this.height;
    const tCtx = treeCanvas.getContext('2d');

    tCtx.fillStyle = '#228B22';
    for (let x = 0; x <= this.width; x += 80) {
      const treeHeight = 100 + Math.random() * 50;
      tCtx.fillRect(x + 20, this.height - treeHeight - 50, 20, treeHeight);
      tCtx.beginPath();
      tCtx.arc(x + 30, this.height - treeHeight - 50, 30, 0, Math.PI * 2);
      tCtx.fill();
    }

    this.parallax.addLayer(treeCanvas, 0.5, 0, true, 'trees');

    // Ground
    this.parallax.addGroundLayer(1);
  }

  createNightModeBackground() {
    this.parallax.addGradientLayer(this.skyColors.night, 0, 0, 'sky');
    
    // Moon
    const moonCanvas = document.createElement('canvas');
    moonCanvas.width = 100;
    moonCanvas.height = 100;
    const mCtx = moonCanvas.getContext('2d');

    mCtx.fillStyle = '#FFFACD';
    mCtx.beginPath();
    mCtx.arc(50, 50, 40, 0, Math.PI * 2);
    mCtx.fill();

    mCtx.fillStyle = '#1a1a2a';
    mCtx.beginPath();
    mCtx.arc(70, 40, 30, 0, Math.PI * 2);
    mCtx.fill();

    this.parallax.addLayer(moonCanvas, 0.1, 50, false, 'moon');

    this.parallax.addCloudLayer(0.15);
  }

  setTimeOfDay(time) {
    this.dayCycle = time;
  }

  update(deltaTime) {
    this.time += deltaTime;
    this.parallax.update(deltaTime, 0, 0);
  }

  draw(ctx) {
    this.parallax.draw(ctx);
  }

  setParallaxFactor(factor) {
    this.parallax.setParallaxFactor(factor);
  }
}
```

### 3. Parallax Controller

```javascript
class ParallaxController {
  constructor(width, height) {
    this.width = width;
    this.height = height;
    this.manager = new DynamicBackgroundManager(width, height);
    this.speedMultiplier = 1;
    this.enabled = true;
  }

  createBackground(type) {
    switch (type) {
      case 'day':
        this.manager.createDayCycleBackground();
        break;
      case 'space':
        this.manager.createSpaceBackground();
        break;
      case 'forest':
        this.manager.createForestBackground();
        break;
      case 'night':
        this.manager.createNightModeBackground();
        break;
      default:
        this.manager.createDayCycleBackground();
    }
  }

  update(deltaTime) {
    if (!this.enabled) return;

    this.manager.update(deltaTime * this.speedMultiplier);
  }

  draw(ctx) {
    this.manager.draw(ctx);
  }

  // Smooth parallax following
  followCamera(camera, deltaTime) {
    this.manager.parallax.update(deltaTime, camera.x, camera.y);
  }

  // Speed control
  setSpeedMultiplier(multiplier) {
    this.speedMultiplier = multiplier;
  }

  // Time of day control
  setTimeOfDay(time) {
    this.manager.setTimeOfDay(time);
  }
}
```

### 4. Wall Scrolling Background

```javascript
class ScrollingBackground {
  constructor(width, height) {
    this.width = width;
    this.height = height;
    this.backgroundX = 0;
    this.scrollSpeed = 100;
    this.layers = [];
  }

  addLayer(image, speed) {
    this.layers.push({
      image,
      speed,
      x: 0
    });
  }

  addGradientLayer(colors, speed) {
    const canvas = document.createElement('canvas');
    canvas.width = this.width;
    canvas.height = this.height;
    const ctx = canvas.getContext('2d');

    const gradient = ctx.createLinearGradient(0, 0, this.width, 0);
    colors.forEach((color, index) => {
      gradient.addColorStop(index / (colors.length - 1), color);
    });

    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, this.width, this.height);

    this.addLayer(canvas, speed);
  }

  update(deltaTime) {
    this.backgroundX -= this.scrollSpeed * deltaTime;

    this.layers.forEach(layer => {
      const moveAmount = layer.speed * this.scrollSpeed * deltaTime;
      layer.x -= moveAmount;

      if (layer.x <= -this.width) {
        layer.x = 0;
      }
    });
  }

  draw(ctx) {
    this.layers.forEach(layer => {
      let x = layer.x;
      
      while (x < this.width) {
        ctx.drawImage(layer.image, x, 0);
        x += this.width;
      }
    });
  }

  setSpeed(speed) {
    this.scrollSpeed = speed;
  }
}
```
