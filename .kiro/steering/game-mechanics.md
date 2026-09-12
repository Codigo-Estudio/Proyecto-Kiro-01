# Ghosty Game Mechanics

This steering file defines physics, wall generation, and scoring patterns for the Ghosty game.

## Table of Contents

- [Ghost Physics Constants](#ghost-physics-constants)
- [Ghost Movement Algorithms](#ghost-movement-algorithms)
- [Wall Generation Algorithms](#wall-generation-algorithms)
- [Scoring System Patterns](#scoring-system-patterns)
- [Complete Game Implementation](#complete-game-implementation)

---

## Ghost Physics Constants

### Ghost-Specific Physics

```javascript
const GhostPhysics = {
  // Hover physics (ghosts float rather than fall)
  HOVER_FORCE: 150,          // Upward force to counteract gravity
  HOVER_DAMPING: 0.92,       // Air resistance for smooth hovering
  HOVER_SMOOTHING: 0.05,     // Lerp factor for smooth movement
  
  // Gravity (reduced for ghostly feel)
  GRAVITY: 300,              // Much weaker gravity
  
  // Movement physics
  MOVE_SPEED: 180,           // Horizontal movement speed
  ACCELERATION: 400,         // How quickly ghost reaches max speed
  DECELERATION: 200,         // How quickly ghost stops
  
  // Rotation physics
  ROTATION_SPEED: 150,       // Degrees per second
  ROTATION_MAX: 30,          // Maximum rotation angle
  ROTATION_SMoothing: 0.1    // Lerp factor
};

const GhostDimensions = {
  WIDTH: 48,
  HEIGHT: 40,
  RADIUS: 24,
  TAIL_LENGTH: 20
};

const GameDimensions = {
  WIDTH: 320,
  HEIGHT: 480,
  GROUND_HEIGHT: 50,
  WALL_WIDTH: 60,
  WALL_GAP: 140,
  WALL_SPACING: 220
};
```

### Difficulty Scaling

```javascript
const Difficulty = {
  INITIAL_WALL_SPEED: 100,
  WALL_SPEED_INCREMENT: 5,
  WALL_SPAWN_BASE: 2000,      // milliseconds between walls
  WALL_SPAWN_MIN: 800,
  WALL_GAP_MIN: 90,
  WALL_GAP_MAX: 200,
  
  // Scoring multipliers
  SCORE_MULTIPLIER_BASE: 1,
  SCORE_MULTIPLIER_MAX: 3,
  COMBO_WINDOW: 2.0           // seconds to chain score multipliers
};
```

---

## Ghost Movement Algorithms

### 1. Hover and Float Physics

```javascript
class HoverSystem {
  constructor() {
    this.hoverForce = GhostPhysics.HOVER_FORCE;
    this.gravity = GhostPhysics.GRAVITY;
    this.damping = GhostPhysics.HOVER_DAMPING;
    this.smoothing = GhostPhysics.HOVER_SMOOTHING;
  }

  applyHover(entity, deltaTime, input) {
    const physics = entity.getComponent('physics');
    if (!physics) return;

    // Apply gravity
    physics.velocity.y += this.gravity * deltaTime;
    
    // Apply hover force when input is active
    if (input.isHovering) {
      physics.velocity.y -= this.hoverForce * deltaTime;
    }

    // Apply air resistance
    physics.velocity.y *= this.damping;
    physics.velocity.x *= this.damping;

    // Smooth vertical velocity
    const targetVelocityY = input.isHovering ? -50 : 0;
    physics.velocity.y = this.lerp(physics.velocity.y, targetVelocityY, this.smoothing);
  }

  lerp(start, end, t) {
    return start * (1 - t) + end * t;
  }

  updatePosition(entity, deltaTime) {
    const physics = entity.getComponent('physics');
    if (!physics) return;

    entity.x += physics.velocity.x * deltaTime;
    entity.y += physics.velocity.y * deltaTime;
  }
}
```

### 2. Smooth Rotation System

```javascript
class RotationSystem {
  constructor() {
    this.maxRotation = GhostPhysics.ROTATION_MAX;
    this.rotationSpeed = GhostPhysics.ROTATION_SPEED;
  }

  updateRotation(entity, deltaTime) {
    const physics = entity.getComponent('physics');
    if (!physics || !entity.rotation) return;

    // Calculate target rotation based on vertical velocity
    let targetRotation = 0;
    
    if (physics.velocity.y < -10) {
      // Moving up - tilt up
      targetRotation = -this.maxRotation * (Math.abs(physics.velocity.y) / 200);
    } else if (physics.velocity.y > 10) {
      // Moving down - tilt down
      targetRotation = this.maxRotation * (Math.abs(physics.velocity.y) / 200);
    }

    // Smoothly interpolate rotation
    entity.rotation = this.lerp(entity.rotation, targetRotation, deltaTime);
  }

  lerp(start, end, t) {
    const diff = end - start;
    const maxStep = this.rotationSpeed * deltaTime;
    
    if (Math.abs(diff) < maxStep) {
      return end;
    }
    
    return start + Math.sign(diff) * maxStep;
  }

  toRadians(degrees) {
    return degrees * Math.PI / 180;
  }
}
```

### 3. Tail Follow Algorithm

```javascript
class TailSystem {
  constructor() {
    this.points = [];
    this.tailLength = GhostDimensions.TAIL_LENGTH;
    this.updateRate = 0.1; // seconds between point updates
    this.elapsed = 0;
  }

  updateTail(entity, deltaTime) {
    this.elapsed += deltaTime;
    
    if (this.elapsed >= this.updateRate) {
      this.elapsed = 0;
      
      // Add new tail point
      const x = entity.x + entity.width / 2;
      const y = entity.y + entity.height / 2;
      this.points.unshift({ x, y });
      
      // Limit tail length
      while (this.points.length > this.tailLength) {
        this.points.pop();
      }
    }
  }

  getTailPoint(index) {
    return this.points[Math.min(index, this.points.length - 1)];
  }

  clear() {
    this.points = [];
  }

  draw(ctx, entity) {
    if (this.points.length < 2) return;

    ctx.strokeStyle = 'rgba(200, 200, 200, 0.5)';
    ctx.lineWidth = 4;
    ctx.lineCap = 'round';
    ctx.lineJoin = 'round';

    ctx.beginPath();
    ctx.moveTo(this.points[0].x, this.points[0].y);

    for (let i = 1; i < this.points.length; i++) {
      const point = this.points[i];
      ctx.lineTo(point.x, point.y);
    }

    ctx.stroke();
  }
}
```

### 4. Input Handling

```javascript
class InputSystem {
  constructor() {
    this.isHovering = false;
    this.keys = new Map();
    this.mouse = { x: 0, y: 0, button: false };
  }

  init() {
    window.addEventListener('keydown', e => {
      this.keys.set(e.code, true);
      if (e.code === 'Space' || e.code === 'ArrowUp' || e.code === 'KeyW') {
        this.isHovering = true;
      }
    });

    window.addEventListener('keyup', e => {
      this.keys.set(e.code, false);
      if (e.code === 'Space' || e.code === 'ArrowUp' || e.code === 'KeyW') {
        this.isHovering = false;
      }
    });

    window.addEventListener('mousedown', e => {
      this.mouse.button = true;
      this.isHovering = true;
    });

    window.addEventListener('mouseup', e => {
      this.mouse.button = false;
      this.isHovering = false;
    });

    window.addEventListener('mousemove', e => {
      const rect = document.getElementById('gameCanvas').getBoundingClientRect();
      this.mouse.x = e.clientX - rect.left;
      this.mouse.y = e.clientY - rect.top;
    });

    window.addEventListener('touchstart', e => {
      e.preventDefault();
      const rect = document.getElementById('gameCanvas').getBoundingClientRect();
      this.mouse.x = e.touches[0].clientX - rect.left;
      this.mouse.y = e.touches[0].clientY - rect.top;
      this.isHovering = true;
    }, { passive: false });

    window.addEventListener('touchend', e => {
      this.isHovering = false;
    });
  }

  isPressed(key) {
    return this.keys.get(key) || false;
  }

  isClicked() {
    return this.mouse.button;
  }

  getMousePosition() {
    return { x: this.mouse.x, y: this.mouse.y };
  }
}
```

---

## Wall Generation Algorithms

### 1. Wall Spawner

```javascript
class WallSpawner {
  constructor(gameWidth, gameHeight) {
    this.gameWidth = gameWidth;
    this.gameHeight = gameHeight;
    this.walls = [];
    this.spawnTimer = 0;
    this.spawnInterval = Difficulty.WALL_SPAWN_BASE;
    this.wallSpeed = Difficulty.INITIAL_WALL_SPEED;
    this.lastWallX = 0;
  }

  update(deltaTime, score) {
    // Update spawn timer
    this.spawnTimer -= deltaTime * 1000;

    if (this.spawnTimer <= 0) {
      this.spawnWallPair();
      this.spawnTimer = this.spawnInterval;
      
      // Increase difficulty
      if (this.spawnInterval > Difficulty.WALL_SPAWN_MIN) {
        this.spawnInterval -= 20;
      }
      
      if (this.wallSpeed < Difficulty.INITIAL_WALL_SPEED + (score * Difficulty.WALL_SPEED_INCREMENT)) {
        this.wallSpeed += Difficulty.WALL_SPEED_INCREMENT;
      }
    }

    // Update wall positions
    for (let i = this.walls.length - 1; i >= 0; i--) {
      const wall = this.walls[i];
      wall.x -= this.wallSpeed * deltaTime;

      // Remove off-screen walls
      if (wall.x + GameDimensions.WALL_WIDTH < -50) {
        this.walls.splice(i, 1);
      }
    }
  }

  spawnWallPair() {
    const minGap = Math.max(Difficulty.WALL_GAP_MIN, Difficulty.WALL_GAP_MAX - this.walls.length * 5);
    const maxGapY = this.gameHeight - GameDimensions.GROUND_HEIGHT - minGap - 100;
    const gapY = Math.random() * maxGapY + 50;

    const topWall = {
      x: this.gameWidth,
      y: 0,
      width: GameDimensions.WALL_WIDTH,
      height: gapY,
      passed: false,
      type: 'top'
    };

    const bottomWall = {
      x: this.gameWidth,
      y: gapY + minGap,
      width: GameDimensions.WALL_WIDTH,
      height: this.gameHeight - GameDimensions.GROUND_HEIGHT - gapY - minGap,
      passed: false,
      type: 'bottom'
    };

    this.walls.push(topWall, bottomWall);
    this.lastWallX = this.gameWidth;
  }

  reset() {
    this.walls = [];
    this.spawnTimer = 0;
    this.spawnInterval = Difficulty.WALL_SPAWN_BASE;
    this.wallSpeed = Difficulty.INITIAL_WALL_SPEED;
    this.lastWallX = 0;
  }

  getWallCount() {
    return Math.floor(this.walls.length / 2);
  }
}
```

### 2. Wall Rendering System

```javascript
class WallRenderer {
  constructor(ctx) {
    this.ctx = ctx;
    this.glowAmount = 20;
  }

  drawWall(wall) {
    const glowColor = this.getGlowColor(wall.type);
    const fillColor = this.getFillType(wall.type);

    // Draw glow effect
    this.ctx.shadowBlur = this.glowAmount;
    this.ctx.shadowColor = glowColor;

    // Draw main wall
    this.ctx.fillStyle = fillColor;
    this.ctx.fillRect(wall.x, wall.y, wall.width, wall.height);

    // Draw wall details
    this.drawWallDetail(wall);

    // Reset shadow
    this.ctx.shadowBlur = 0;
  }

  getGlowColor(type) {
    return type === 'top' ? 'rgba(0, 255, 255, 0.5)' : 'rgba(255, 0, 255, 0.5)';
  }

  getFillType(type) {
    return type === 'top' ? 'rgba(0, 150, 150, 0.8)' : 'rgba(150, 0, 150, 0.8)';
  }

  drawWallDetail(wall) {
    // Draw horizontal stripes on wall
    this.ctx.fillStyle = 'rgba(255, 255, 255, 0.2)';
    
    for (let y = 0; y < wall.height; y += 20) {
      this.ctx.fillRect(wall.x + 5, wall.y + y, wall.width - 10, 2);
    }

    // Draw wall caps
    this.ctx.fillStyle = 'rgba(255, 255, 255, 0.3)';
    const capHeight = 10;
    this.ctx.fillRect(wall.x - 5, wall.type === 'top' ? wall.height - capHeight : 0, wall.width + 10, capHeight);
  }

  drawGhost(ghost) {
    const glowColor = 'rgba(200, 200, 255, 0.6)';
    
    this.ctx.shadowBlur = 30;
    this.ctx.shadowColor = glowColor;
    this.ctx.fillStyle = 'rgba(220, 220, 255, 0.9)';

    // Draw ghost body
    this.ctx.beginPath();
    this.ctx.arc(ghost.x + ghost.width / 2, ghost.y + ghost.height / 2, ghost.width / 2 - 4, Math.PI, 0);
    this.ctx.lineTo(ghost.x + ghost.width - 4, ghost.y + ghost.height);
    this.ctx.lineTo(ghost.x + 4, ghost.y + ghost.height);
    this.ctx.closePath();
    this.ctx.fill();

    // Draw eyes
    this.ctx.fillStyle = '#1a1a1a';
    this.ctx.shadowBlur = 0;
    
    const eyeSize = 4;
    const eyeOffset = ghost.width / 4;
    this.ctx.beginPath();
    this.ctx.arc(ghost.x + eyeOffset, ghost.y + ghost.height / 2 - 4, eyeSize, 0, Math.PI * 2);
    this.ctx.arc(ghost.x + ghost.width - eyeOffset, ghost.y + ghost.height / 2 - 4, eyeSize, 0, Math.PI * 2);
    this.ctx.fill();
  }
}
```

---

## Scoring System Patterns

### 1. Score Manager

```javascript
class ScoreManager {
  constructor() {
    this.score = 0;
    this.highScore = 0;
    this.multiplier = 1;
    this.comboTimer = 0;
    this.comboWindow = Difficulty.COMBO_WINDOW;
    this.lastScoreTime = 0;
    this.streak = 0;
  }

  addScore(points) {
    this.score += points * this.multiplier;
    this.lastScoreTime = Date.now() / 1000;
    this.streak++;
    
    // Increase multiplier for streaks
    if (this.streak > 1 && this.streak % 3 === 0) {
      this.multiplier = Math.min(this.multiplier + 0.5, Difficulty.SCORE_MULTIPLIER_MAX);
    }
  }

  update(deltaTime) {
    // Decay multiplier if combo window expires
    this.comboTimer += deltaTime;
    
    if (this.comboTimer > this.comboWindow) {
      this.multiplier = 1;
      this.streak = 0;
      this.comboTimer = 0;
    }
  }

  reset() {
    this.score = 0;
    this.multiplier = 1;
    this.streak = 0;
    this.comboTimer = 0;
    this.lastScoreTime = 0;
  }

  setHighScore() {
    if (this.score > this.highScore) {
      this.highScore = this.score;
      localStorage.setItem('ghosty_highscore', this.highScore);
    }
  }

  getHighScore() {
    return this.highScore || parseInt(localStorage.getItem('ghosty_highscore')) || 0;
  }
}
```

### 2. Combo System

```javascript
class ComboSystem {
  constructor() {
    this.combo = 0;
    this.maxCombo = 0;
    this.lastHitTime = 0;
    this.comboWindow = 2.0; // seconds
  }

  addHit(deltaTime) {
    const currentTime = Date.now() / 1000;
    const timeSinceLastHit = currentTime - this.lastHitTime;
    
    if (timeSinceLastHit <= this.comboWindow) {
      this.combo++;
      this.maxCombo = Math.max(this.combo, this.maxCombo);
    } else {
      this.combo = 1;
    }
    
    this.lastHitTime = currentTime;
    
    return this.getComboMultiplier();
  }

  getComboMultiplier() {
    // Exponential multiplier growth
    return 1 + Math.floor((this.combo - 1) / 3) * 0.5;
  }

  getComboMultiplierText() {
    const mult = this.getComboMultiplier();
    return mult > 1 ? `x${mult}` : '';
  }

  reset() {
    this.combo = 0;
    this.maxCombo = 0;
    this.lastHitTime = 0;
  }
}
```

### 3. Score Renderer

```javascript
class ScoreRenderer {
  constructor(ctx, width, height) {
    this.ctx = ctx;
    this.width = width;
    this.height = height;
  }

  drawScore(scoreManager) {
    this.ctx.fillStyle = '#FFF';
    this.ctx.textAlign = 'center';
    this.ctx.font = 'bold 48px Arial';
    
    // Main score
    this.ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
    this.ctx.shadowBlur = 4;
    this.ctx.fillText(scoreManager.score, this.width / 2, 60);
    this.ctx.shadowBlur = 0;

    // Multiplier
    if (scoreManager.multiplier > 1) {
      this.ctx.font = 'bold 24px Arial';
      this.ctx.fillStyle = '#FFD700';
      this.ctx.fillText(`x${scoreManager.multiplier}`, this.width / 2, 90);
    }

    // High score
    this.ctx.font = '16px Arial';
    this.ctx.fillStyle = '#DDD';
    this.ctx.fillText(`High Score: ${scoreManager.getHighScore()}`, this.width / 2, 30);
  }
}
```

---

## Complete Game Implementation

### Ghosty Game Class

```javascript
class GhostyGame {
  constructor(canvasId) {
    this.canvas = document.getElementById(canvasId);
    this.ctx = this.canvas.getContext('2d');
    this.canvas.width = GameDimensions.WIDTH;
    this.canvas.height = GameDimensions.HEIGHT;

    // Systems
    this.input = new InputSystem();
    this.hoverSystem = new HoverSystem();
    this.rotationSystem = new RotationSystem();
    this.tailSystem = new TailSystem();
    this.wallSpawner = new WallSpawner(this.canvas.width, this.canvas.height);
    this.scoreManager = new ScoreManager();
    this.wallRenderer = new WallRenderer(this.ctx);

    // Game state
    this.gameState = 'start'; // start, playing, gameover
    this.lastTime = 0;
    this.isRunning = false;

    // Ghost entity
    this.ghost = {
      x: 50,
      y: this.canvas.height / 2,
      width: GhostDimensions.WIDTH,
      height: GhostDimensions.HEIGHT,
      rotation: 0,
      bounds: {
        x: 50,
        y: this.canvas.height / 2,
        width: GhostDimensions.WIDTH,
        height: GhostDimensions.HEIGHT,
        right: 50 + GhostDimensions.WIDTH,
        bottom: this.canvas.height / 2 + GhostDimensions.HEIGHT
      },
      physics: {
        velocity: { x: 0, y: 0 },
        acceleration: { x: 0, y: 0 }
      }
    };

    this.input.init();
    this.bindInput();
  }

  bindInput() {
    this.input.onHover = () => this.hoverSystem.applyHover(this.ghost, this.deltaTime);
    this.input.onMove = () => this.hoverSystem.updatePosition(this.ghost, this.deltaTime);
  }

  start() {
    this.gameState = 'playing';
    this.isRunning = true;
    this.lastTime = performance.now();
    
    // Reset entities
    this.ghost.x = 50;
    this.ghost.y = this.canvas.height / 2;
    this.ghost.physics.velocity = { x: 0, y: 0 };
    this.ghost.rotation = 0;
    
    this.wallSpawner.reset();
    this.scoreManager.reset();
    this.tailSystem.clear();
    
    this.gameLoop();
  }

  gameOver() {
    this.gameState = 'gameover';
    this.isRunning = false;
    this.scoreManager.setHighScore();
  }

  update(deltaTime) {
    if (!this.isRunning) return;

    // Update systems
    this.hoverSystem.applyHover(this.ghost, deltaTime, this.input);
    this.hoverSystem.updatePosition(this.ghost, deltaTime);
    this.rotationSystem.updateRotation(this.ghost, deltaTime);
    this.tailSystem.updateTail(this.ghost, deltaTime);
    this.wallSpawner.update(deltaTime, this.scoreManager.score);
    this.scoreManager.update(deltaTime);

    // Check boundaries
    if (this.ghost.y < 0) {
      this.ghost.y = 0;
      this.ghost.physics.velocity.y = 0;
    }

    if (this.ghost.y + this.ghost.height > this.canvas.height - GameDimensions.GROUND_HEIGHT) {
      this.gameOver();
    }

    // Check collisions
    this.checkCollisions();

    // Check scoring
    this.checkScoring();
  }

  checkCollisions() {
    const ghostAABB = this.ghost.bounds;

    this.wallSpawner.walls.forEach(wall => {
      const wallAABB = {
        x: wall.x,
        y: wall.y,
        width: wall.width,
        height: wall.height,
        right: wall.x + wall.width,
        bottom: wall.y + wall.height
      };

      if (this.aabbIntersect(ghostAABB, wallAABB)) {
        this.gameOver();
      }
    });
  }

  aabbIntersect(a, b) {
    return (
      a.x < b.right &&
      a.right > b.x &&
      a.y < b.bottom &&
      a.bottom > b.y
    );
  }

  checkScoring() {
    this.wallSpawner.walls.forEach(wall => {
      if (wall.type === 'top' && !wall.passed && wall.x + wall.width < this.ghost.x) {
        wall.passed = true;
        this.scoreManager.addScore(1);
      }
    });
  }

  render() {
    // Clear canvas
    this.ctx.fillStyle = '#0a0a1a';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

    // Draw stars background
    this.drawBackground();

    // Draw walls
    this.wallSpawner.walls.forEach(wall => {
      this.wallRenderer.drawWall(wall);
    });

    // Draw ghost
    this.wallRenderer.drawGhost(this.ghost);

    // Draw tail
    this.tailSystem.draw(this.ctx, this.ghost);

    // Draw ground
    this.ctx.fillStyle = '#2a2a3a';
    this.ctx.fillRect(0, this.canvas.height - GameDimensions.GROUND_HEIGHT, this.canvas.width, GameDimensions.GROUND_HEIGHT);

    // Draw UI
    if (this.gameState === 'start') {
      this.drawStartScreen();
    } else if (this.gameState === 'gameover') {
      this.drawGameOverScreen();
    } else {
      this.wallRenderer.drawScore(this.scoreManager);
    }
  }

  drawBackground() {
    this.ctx.fillStyle = '#FFF';
    for (let i = 0; i < 50; i++) {
      const x = (i * 37 + this.scoreManager.score * 10) % this.canvas.width;
      const y = (i * 23) % (this.canvas.height - GameDimensions.GROUND_HEIGHT);
      const size = (i % 3) + 1;
      this.ctx.globalAlpha = (i % 5) / 10 + 0.3;
      this.ctx.fillRect(x, y, size, size);
    }
    this.ctx.globalAlpha = 1;
  }

  drawStartScreen() {
    this.ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

    this.ctx.fillStyle = '#FFF';
    this.ctx.textAlign = 'center';
    this.ctx.font = 'bold 48px Arial';
    this.ctx.shadowColor = '#0ff';
    this.ctx.shadowBlur = 20;
    this.ctx.fillText('GHOSTY', this.canvas.width / 2, this.canvas.height / 3);
    this.ctx.shadowBlur = 0;

    this.ctx.font = '24px Arial';
    this.ctx.fillText('Hold SPACE to Hover', this.canvas.width / 2, this.canvas.height / 2);
    this.ctx.fillText('Press SPACE to Start', this.canvas.width / 2, this.canvas.height / 2 + 50);
  }

  drawGameOverScreen() {
    this.ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

    this.ctx.fillStyle = '#FFF';
    this.ctx.textAlign = 'center';
    this.ctx.font = 'bold 48px Arial';
    this.ctx.shadowColor = '#f0f';
    this.ctx.shadowBlur = 20;
    this.ctx.fillText('GAME OVER', this.canvas.width / 2, this.canvas.height / 3);
    this.ctx.shadowBlur = 0;

    this.ctx.font = '24px Arial';
    this.ctx.fillText(`Score: ${this.scoreManager.score}`, this.canvas.width / 2, this.canvas.height / 2);
    this.ctx.fillText(`High Score: ${this.scoreManager.getHighScore()}`, this.canvas.width / 2, this.canvas.height / 2 + 40);
    this.ctx.fillText('Press SPACE to Restart', this.canvas.width / 2, this.canvas.height / 2 + 80);
  }

  gameLoop() {
    if (!this.isRunning) return;

    const timestamp = performance.now();
    this.deltaTime = (timestamp - this.lastTime) / 1000;
    this.lastTime = timestamp;

    this.update(this.deltaTime);
    this.render();

    requestAnimationFrame(() => this.gameLoop());
  }

  handleInput() {
    if (this.gameState === 'start' || this.gameState === 'gameover') {
      this.start();
    }
  }

  start() {
    this.handleInput();
    if (!this.isRunning) {
      this.lastTime = performance.now();
      this.gameLoop();
    }
  }
}

// Initialize game
const game = new GhostyGame('gameCanvas');
document.addEventListener('keydown', e => {
  if (e.code === 'Space' || e.code === 'ArrowUp') {
    game.start();
  }
});
```
