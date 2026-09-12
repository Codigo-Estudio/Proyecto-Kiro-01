# Gravity and Input Steering File

This steering file defines gravity simulation, responsive input handling, and obstacle spawning logic for the Ghosty game.

## Table of Contents

- [Gravity Simulation](#gravity-simulation)
- [Input Handling Responsiveness](#input-handling-responsiveness)
- [Obstacle Spawning Logic](#obstacle-spawning-logic)
- [Physics Engine Implementation](#physics-engine-implementation)

---

## Gravity Simulation

### 1. Variable Gravity Model

```javascript
const GravityModel = {
  // Standard gravity (pixels per second squared)
  GRAVITY: 980,
  
  // Reduced gravity for ghost-like float
  GHOST_GRAVITY: 200,
  
  // Inverted gravity (for anti-gravity zones)
  INVERTED_GRAVITY: -400,
  
  // Zero gravity threshold
  ZERO_GRAVITY: 0,
  
  // Gravity falloff (for distance-based gravity)
  GRAVITY_FALLOFF: 0.01,
  
  // Gravity wells (attractors)
  GRAVITY_WELL_STRENGTH: 150000
};

class VariableGravity {
  constructor() {
    this.currentGravity = GravityModel.GHOST_GRAVITY;
    this.targetGravity = GravityModel.GHOST_GRAVITY;
    this.gravityTransitionSpeed = 2;
    this.gravityActive = true;
  }

  setGravity(value) {
    this.targetGravity = value;
  }

  setGhostGravity() {
    this.targetGravity = GravityModel.GHOST_GRAVITY;
  }

  setStandardGravity() {
    this.targetGravity = GravityModel.GRAVITY;
  }

  setInvertedGravity() {
    this.targetGravity = GravityModel.INVERTED_GRAVITY;
  }

  setZeroGravity() {
    this.targetGravity = GravityModel.ZERO_GRAVITY;
  }

  update(deltaTime) {
    // Smoothly transition between gravity values
    if (this.gravityActive) {
      const diff = this.targetGravity - this.currentGravity;
      this.currentGravity += diff * this.gravityTransitionSpeed * deltaTime;
    }
  }

  applyGravity(entity, deltaTime) {
    if (!this.gravityActive || !entity.physics) return;

    entity.physics.velocity.y += this.currentGravity * deltaTime;
  }

  // Distance-based gravity (attract/repel)
  applyDistanceGravity(entity, attractorX, attractorY, strength) {
    if (!entity.physics) return;

    const dx = attractorX - (entity.x + entity.width / 2);
    const dy = attractorY - (entity.y + entity.height / 2);
    const distance = Math.sqrt(dx * dx + dy * dy);

    if (distance > 0) {
      const force = strength / (distance * distance);
      entity.physics.velocity.x += (dx / distance) * force * 1000 * deltaTime;
      entity.physics.velocity.y += (dy / distance) * force * 1000 * deltaTime;
    }
  }
}
```

### 2. Air Resistance and Drag

```javascript
class AirResistance {
  constructor() {
    this.density = 0.001225; // Air density
    this.dragCoefficient = 0.47; // Sphere drag coefficient
    this.termVelocity = 60; // Terminal velocity (pixels per second)
  }

  applyDrag(entity, deltaTime) {
    if (!entity.physics) return;

    const v = Math.sqrt(entity.physics.velocity.x ** 2 + entity.physics.velocity.y ** 2);
    
    if (v > 0) {
      // Quadratic drag model
      const dragForce = 0.5 * this.density * this.dragCoefficient * v ** 2;
      const dragAcceleration = dragForce / (entity.mass || 1);
      
      entity.physics.velocity.x -= (entity.physics.velocity.x / v) * dragAcceleration * deltaTime;
      entity.physics.velocity.y -= (entity.physics.velocity.y / v) * dragAcceleration * deltaTime;
      
      // Clamp to terminal velocity
      const maxV = this.termVelocity;
      if (Math.abs(entity.physics.velocity.x) > maxV) {
        entity.physics.velocity.x = Math.sign(entity.physics.velocity.x) * maxV;
      }
      if (Math.abs(entity.physics.velocity.y) > maxV) {
        entity.physics.velocity.y = Math.sign(entity.physics.velocity.y) * maxV;
      }
    }
  }

  // Simplified linear drag (more performant)
  applyLinearDrag(entity, deltaTime, dragFactor = 0.98) {
    if (!entity.physics) return;
    
    entity.physics.velocity.x *= dragFactor;
    entity.physics.velocity.y *= dragFactor;
  }
}
```

### 3. Buoyancy and Floating

```javascript
class BuoyancySystem {
  constructor(fluidDensity = 1000) {
    this.fluidDensity = fluidDensity;
    this.buoyancyFactor = 1.0;
  }

  applyBuoyancy(entity, deltaTime, fluidLevel) {
    if (!entity.physics || !entity.volume) return;

    const entityCenter = entity.y + entity.height / 2;
    const submergedDepth = Math.max(0, fluidLevel - entityCenter - entity.height / 2);
    const submergedVolume = entity.volume * (submergedDepth / entity.height);
    const displacedMass = submergedVolume * this.fluidDensity;
    const buoyantForce = displacedMass * GravityModel.GRAVITY;

    entity.physics.velocity.y -= buoyantForce * deltaTime * this.buoyancyFactor;

    return submergedDepth;
  }

  setBuoyancyFactor(factor) {
    this.buoyancyFactor = factor;
  }
}
```

---

## Input Handling Responsiveness

### 1. Input Buffer System

```javascript
class InputBuffer {
  constructor(bufferSize = 6) {
    this.buffer = [];
    this.bufferSize = bufferSize;
    this.lastInputTime = 0;
    this.inputWindow = 0.2; // 200ms input window
  }

  addInput(inputType) {
    const currentTime = performance.now() / 1000;
    
    // Remove old inputs outside window
    this.buffer = this.buffer.filter(
      input => currentTime - input.timestamp < this.inputWindow
    );

    // Add new input
    this.buffer.push({
      type: inputType,
      timestamp: currentTime,
      frame: performance.now()
    });

    this.lastInputTime = currentTime;
  }

  hasBufferedInput() {
    return this.buffer.length > 0;
  }

  consumeBufferedInput() {
    if (this.buffer.length > 0) {
      return this.buffer.shift();
    }
    return null;
  }

  clear() {
    this.buffer = [];
  }

  getBufferCount() {
    return this.buffer.length;
  }

  // Check if specific input is buffered
  hasInputType(type) {
    return this.buffer.some(input => input.type === type);
  }
}
```

### 2. Responsive Input Handler

```javascript
class ResponsiveInputHandler {
  constructor() {
    this.keys = new Map();
    this.mouse = { x: 0, y: 0, button: false };
    this.touch = { x: 0, y: 0, active: false };
    this.lastPressedKeys = new Map();
    this.inputBuffer = new InputBuffer(6);
    
    this.bindEvents();
  }

  bindEvents() {
    window.addEventListener('keydown', e => this.handleKeyDown(e));
    window.addEventListener('keyup', e => this.handleKeyUp(e));
    window.addEventListener('mousedown', e => this.handleMouseDown(e));
    window.addEventListener('mouseup', e => this.handleMouseUp(e));
    window.addEventListener('mousemove', e => this.handleMouseMove(e));
    
    // Touch events
    window.addEventListener('touchstart', e => this.handleTouchStart(e), { passive: false });
    window.addEventListener('touchend', e => this.handleTouchEnd(e));
    window.addEventListener('touchmove', e => this.handleTouchMove(e), { passive: false });
  }

  handleKeyDown(e) {
    const prevPressed = this.keys.get(e.code);
    this.keys.set(e.code, true);
    
    if (!prevPressed) {
      // Add to buffer on initial press
      this.inputBuffer.addInput(e.code);
    }
    
    this.lastPressedKeys.set(e.code, performance.now());
  }

  handleKeyUp(e) {
    this.keys.set(e.code, false);
    this.lastPressedKeys.delete(e.code);
  }

  handleMouseDown(e) {
    const rect = e.target.getBoundingClientRect();
    this.mouse.x = e.clientX - rect.left;
    this.mouse.y = e.clientY - rect.top;
    this.mouse.button = true;
    this.inputBuffer.addInput('mousedown');
  }

  handleMouseUp(e) {
    this.mouse.button = false;
  }

  handleMouseMove(e) {
    const rect = e.target.getBoundingClientRect();
    this.mouse.x = e.clientX - rect.left;
    this.mouse.y = e.clientY - rect.top;
  }

  handleTouchStart(e) {
    e.preventDefault();
    this.touch.active = true;
    const rect = e.target.getBoundingClientRect();
    this.touch.x = e.touches[0].clientX - rect.left;
    this.touch.y = e.touches[0].clientY - rect.top;
    this.inputBuffer.addInput('touch');
  }

  handleTouchEnd(e) {
    this.touch.active = false;
  }

  handleTouchMove(e) {
    e.preventDefault();
    const rect = e.target.getBoundingClientRect();
    this.touch.x = e.touches[0].clientX - rect.left;
    this.touch.y = e.touches[0].clientY - rect.top;
  }

  // Input state checks
  isPressed(code) {
    return this.keys.get(code) || false;
  }

  isJustPressed(code) {
    const lastTime = this.lastPressedKeys.get(code);
    return lastTime && performance.now() - lastTime < 150;
  }

  isClicked() {
    return this.mouse.button;
  }

  isTouched() {
    return this.touch.active;
  }

  getMousePosition() {
    return { x: this.mouse.x, y: this.mouse.y };
  }

  getTouchPosition() {
    return { x: this.touch.x, y: this.touch.y };
  }

  // Input type detection
  getPrimaryInput() {
    if (this.touch.active) return 'touch';
    if (this.mouse.button) return 'mouse';
    for (const [code, pressed] of this.keys) {
      if (pressed) return code;
    }
    return null;
  }

  // Input smoothing for analog control
  getSmoothInput(key, smoothing = 0.1) {
    const current = this.keys.get(key) ? 1 : 0;
    const prev = this.lastPressedKeys.get(key + '_smooth') || 0;
    const value = prev * (1 - smoothing) + current * smoothing;
    this.lastPressedKeys.set(key + '_smooth', value);
    return value;
  }

  // Multi-input detection
  getPressedKeys() {
    const pressed = [];
    for (const [code, isPressed] of this.keys) {
      if (isPressed) pressed.push(code);
    }
    return pressed;
  }

  // Input reset
  reset() {
    this.keys.clear();
    this.lastPressedKeys.clear();
    this.mouse.button = false;
    this.touch.active = false;
    this.inputBuffer.clear();
  }
}
```

### 3. Input Response System

```javascript
class InputResponseSystem {
  constructor(inputHandler) {
    this.input = inputHandler;
    this.responses = new Map();
    this.responseThresholds = {
      click: 150,        // ms
      doubleClick: 300,  // ms
      hold: 500,         // ms
      swipe: 200         // ms
    };
    this.lastClickTime = 0;
    this.clickCount = 0;
  }

  registerResponse(inputType, callback) {
    this.responses.set(inputType, callback);
  }

  processInput() {
    // Check buffered inputs
    if (this.input.inputBuffer.hasBufferedInput()) {
      const input = this.input.inputBuffer.consumeBufferedInput();
      if (this.responses.has(input.type)) {
        this.responses.get(input.type)(input);
      }
    }

    // Check mouse click with double-click detection
    if (this.input.isClicked()) {
      const currentTime = performance.now();
      const timeDiff = currentTime - this.lastClickTime;
      
      if (timeDiff < this.responseThresholds.doubleClick) {
        this.clickCount++;
        if (this.clickCount === 2) {
          this.handleDoubleClick();
          this.clickCount = 0;
        }
      } else {
        this.clickCount = 1;
        this.handleSingleClick();
      }
      
      this.lastClickTime = currentTime;
    }
  }

  handleSingleClick() {
    if (this.responses.has('click')) {
      this.responses.get('click')({ type: 'click', count: 1 });
    }
  }

  handleDoubleClick() {
    if (this.responses.has('doubleclick')) {
      this.responses.get('doubleclick')({ type: 'doubleclick' });
    }
  }

  // Input duration tracking
  getKeyPressDuration(code) {
    const startTime = this.input.lastPressedKeys.get(code);
    if (startTime) {
      return performance.now() - startTime;
    }
    return 0;
  }

  // Swipe detection
  detectSwipe() {
    if (!this.input.touch.active) return null;
    
    // Would be implemented with swipe tracking
    return null;
  }
}
```

---

## Obstacle Spawning Logic

### 1. Smart Spawner

```javascript
class SmartSpawner {
  constructor(gameWidth, gameHeight) {
    this.gameWidth = gameWidth;
    this.gameHeight = gameHeight;
    this.obstacles = [];
    this.spawnTimer = 0;
    this.spawnInterval = 2000;
    this.minSpawnInterval = 600;
    this.maxSpawnInterval = 3000;
    this.baseSpeed = 100;
    this.lastSpawnX = gameWidth;
    this.difficulty = 1;
  }

  update(deltaTime, playerProgress) {
    // Update spawn timer
    this.spawnTimer -= deltaTime * 1000;

    if (this.spawnTimer <= 0) {
      this.spawnObstacle();
      
      // Increase difficulty
      this.difficulty += 0.1;
      this.spawnInterval = Math.max(
        this.minSpawnInterval,
        this.maxSpawnInterval - (this.difficulty * 50)
      );
      
      // Increase speed
      this.baseSpeed += 2;
    }

    // Update obstacle positions
    for (let i = this.obstacles.length - 1; i >= 0; i--) {
      const obstacle = this.obstacles[i];
      obstacle.x -= this.baseSpeed * deltaTime * (1 + playerProgress * 0.1);
      obstacle.x -= obstacle.speedOffset * deltaTime;

      // Remove off-screen obstacles
      if (obstacle.x + obstacle.width < -50) {
        this.obstacles.splice(i, 1);
      }
    }
  }

  spawnObstacle() {
    const type = this.getRandomObstacleType();
    
    let obstacle;
    
    switch (type) {
      case 'wall':
        obstacle = this.createWallObstacle();
        break;
      case 'gap':
        obstacle = this.createGapObstacle();
        break;
      case 'platform':
        obstacle = this.createPlatformObstacle();
        break;
      case 'moving':
        obstacle = this.createMovingObstacle();
        break;
      case 'rotating':
        obstacle = this.createRotatingObstacle();
        break;
      default:
        obstacle = this.createWallObstacle();
    }

    this.obstacles.push(obstacle);
    this.lastSpawnX = this.gameWidth;
  }

  createWallObstacle() {
    const gapY = this.getSafeGapPosition();
    
    return {
      x: this.gameWidth,
      y: 0,
      width: 60,
      height: gapY,
      type: 'wall',
      color: `hsl(${200 + Math.random() * 60}, 70%, 50%)`,
      passed: false
    };
  }

  createGapObstacle() {
    const platformY = Math.random() * (this.gameHeight - 150) + 50;
    
    return {
      x: this.gameWidth,
      y: platformY,
      width: 100,
      height: 20,
      type: 'platform',
      color: '#FFD700',
      passed: false,
      speedOffset: (Math.random() - 0.5) * 50
    };
  }

  createPlatformObstacle() {
    return {
      x: this.gameWidth,
      y: this.gameHeight - 100 - Math.random() * 50,
      width: 80 + Math.random() * 40,
      height: 20,
      type: 'platform',
      color: '#32CD32',
      passed: false,
      moving: true,
      speedOffset: (Math.random() - 0.5) * 30
    };
  }

  createMovingObstacle() {
    return {
      x: this.gameWidth,
      y: Math.random() * (this.gameHeight - 100),
      width: 40,
      height: 40,
      type: 'moving',
      color: '#FF6347',
      passed: false,
      speedOffset: (Math.random() - 0.5) * 100,
      verticalSpeed: (Math.random() - 0.5) * 100
    };
  }

  createRotatingObstacle() {
    return {
      x: this.gameWidth,
      y: this.gameHeight / 2,
      width: 30,
      height: 30,
      type: 'rotating',
      color: '#FF69B4',
      passed: false,
      rotation: 0,
      rotationSpeed: (Math.random() - 0.5) * 5
    };
  }

  getSafeGapPosition() {
    // Ensure gap is not too close to previous obstacles
    const minGap = 100;
    const maxGapY = this.gameHeight - 150;
    
    // Check recent obstacles
    let safeY = Math.random() * maxGapY + 50;
    
    if (this.obstacles.length > 0) {
      const lastObstacle = this.obstacles[this.obstacles.length - 1];
      const distance = Math.abs(lastObstacle.y - safeY);
      
      if (distance < minGap) {
        safeY = (safeY + minGap) % (this.gameHeight - minGap);
      }
    }
    
    return safeY;
  }

  getRandomObstacleType() {
    const types = ['wall', 'wall', 'wall', 'platform', 'moving', 'rotating'];
    return types[Math.floor(Math.random() * types.length)];
  }

  // Difficulty scaling
  scaleDifficulty(playerScore) {
    this.difficulty = 1 + playerScore * 0.05;
    this.spawnInterval = Math.max(
      this.minSpawnInterval,
      this.maxSpawnInterval - (playerScore * 30)
    );
    this.baseSpeed = 100 + playerScore * 3;
  }

  reset() {
    this.obstacles = [];
    this.spawnTimer = 0;
    this.spawnInterval = 2000;
    this.baseSpeed = 100;
    this.difficulty = 1;
    this.lastSpawnX = this.gameWidth;
  }

  getObstacleCount() {
    return this.obstacles.length;
  }
}
```

### 2. Pattern-Based Spawner

```javascript
class PatternSpawner {
  constructor(gameWidth, gameHeight) {
    this.gameWidth = gameWidth;
    this.gameHeight = gameHeight;
    this.obstacles = [];
    this.patterns = [];
    this.currentPattern = 0;
    this.patternTimer = 0;
    this.baseSpeed = 100;
  }

  addPattern(pattern) {
    this.patterns.push(pattern);
  }

  update(deltaTime) {
    if (this.patterns.length === 0) return;

    const pattern = this.patterns[this.currentPattern];
    this.patternTimer += deltaTime;

    // Check if pattern should spawn
    if (this.patternTimer >= pattern.spawnInterval) {
      this.spawnPatternObstacles(pattern);
      this.patternTimer = 0;
    }

    // Update obstacle positions
    this.updateObstacles(deltaTime);
  }

  spawnPatternObstacles(pattern) {
    const positions = this.calculatePatternPositions(pattern);
    
    positions.forEach(pos => {
      this.obstacles.push({
        x: this.gameWidth,
        y: pos.y,
        width: pattern.width || 60,
        height: pattern.height || 100,
        type: pattern.type || 'wall',
        color: pattern.color || '#FFF',
        passed: false
      });
    });
  }

  calculatePatternPositions(pattern) {
    const positions = [];
    const gapCount = pattern.gaps || 1;
    const gapSize = pattern.gapSize || 120;
    
    // Calculate evenly spaced gaps
    const totalGapSpace = gapCount * gapSize;
    const availableHeight = this.gameHeight - 100 - totalGapSpace;
    const gapSpacing = availableHeight / (gapCount + 1);
    
    for (let i = 1; i <= gapCount; i++) {
      const gapY = gapSpacing * i + (gapSize - pattern.height || 100) / 2;
      positions.push({ y: gapY });
    }
    
    return positions;
  }

  updateObstacles(deltaTime) {
    for (let i = this.obstacles.length - 1; i >= 0; i--) {
      const obstacle = this.obstacles[i];
      obstacle.x -= this.baseSpeed * deltaTime;

      if (obstacle.x + obstacle.width < -50) {
        this.obstacles.splice(i, 1);
      }
    }
  }

  // Pattern definitions
  static patternDoubleGap() {
    return {
      spawnInterval: 2,
      type: 'wall',
      height: 100,
      gaps: 2,
      gapSize: 150,
      color: '#4169E1'
    };
  }

  static patternMovingObstacles() {
    return {
      spawnInterval: 1.5,
      type: 'moving',
      width: 40,
      height: 40,
      color: '#FF4500',
      speedVariation: 50
    };
  }

  static patternAlternating() {
    return {
      spawnInterval: 2.5,
      type: 'alternating',
      height: 80,
      gaps: 2,
      gapSize: 120,
      color: '#32CD32'
    };
  }
}
```

### 3. Probability-Based Spawner

```javascript
class ProbabilitySpawner {
  constructor(gameWidth, gameHeight) {
    this.gameWidth = gameWidth;
    this.gameHeight = gameHeight;
    this.obstacles = [];
    this.baseSpawnChance = 0.02;
    this.spawnTimer = 0;
    this.baseSpeed = 100;
    
    this.probabilities = {
      'wall': 0.5,
      'platform': 0.2,
      'moving': 0.15,
      'rotating': 0.1,
      'bonus': 0.05
    };
  }

  update(deltaTime) {
    this.spawnTimer += deltaTime;
    
    // Chance-based spawning
    if (this.spawnTimer > 0.1) {
      this.spawnTimer = 0;
      
      const roll = Math.random();
      if (roll < this.baseSpawnChance) {
        this.spawnRandomObstacle();
      }
    }

    // Update obstacle positions
    for (let i = this.obstacles.length - 1; i >= 0; i--) {
      const obstacle = this.obstacles[i];
      obstacle.x -= this.baseSpeed * deltaTime;
      
      // Moving obstacles
      if (obstacle.speedOffset) {
        obstacle.x += obstacle.speedOffset * deltaTime;
      }

      if (obstacle.x + obstacle.width < -50) {
        this.obstacles.splice(i, 1);
      }
    }
  }

  spawnRandomObstacle() {
    const type = this.selectRandomType();
    
    switch (type) {
      case 'wall':
        this.spawnWall();
        break;
      case 'platform':
        this.spawnPlatform();
        break;
      case 'moving':
        this.spawnMoving();
        break;
      case 'rotating':
        this.spawnRotating();
        break;
      case 'bonus':
        this.spawnBonus();
        break;
    }
  }

  selectRandomType() {
    const roll = Math.random();
    let cumulative = 0;
    
    for (const [type, probability] of Object.entries(this.probabilities)) {
      cumulative += probability;
      if (roll <= cumulative) return type;
    }
    
    return 'wall';
  }

  spawnWall() {
    const gapY = Math.random() * (this.gameHeight - 120) + 50;
    
    this.obstacles.push({
      x: this.gameWidth,
      y: 0,
      width: 60,
      height: gapY,
      type: 'wall',
      color: '#6495ED',
      passed: false
    });
    
    this.obstacles.push({
      x: this.gameWidth,
      y: gapY + 120,
      width: 60,
      height: this.gameHeight - gapY - 120,
      type: 'wall',
      color: '#6495ED',
      passed: false
    });
  }

  spawnPlatform() {
    this.obstacles.push({
      x: this.gameWidth,
      y: this.gameHeight - 80 - Math.random() * 40,
      width: 80,
      height: 20,
      type: 'platform',
      color: '#32CD32',
      passed: false
    });
  }

  spawnMoving() {
    this.obstacles.push({
      x: this.gameWidth,
      y: Math.random() * (this.gameHeight - 100),
      width: 30,
      height: 30,
      type: 'moving',
      color: '#FF6347',
      passed: false,
      speedOffset: (Math.random() - 0.5) * 100,
      verticalSpeed: (Math.random() - 0.5) * 50
    });
  }

  spawnRotating() {
    this.obstacles.push({
      x: this.gameWidth,
      y: this.gameHeight / 2,
      width: 25,
      height: 25,
      type: 'rotating',
      color: '#FF69B4',
      passed: false,
      rotation: 0,
      rotationSpeed: (Math.random() - 0.5) * 10
    });
  }

  spawnBonus() {
    this.obstacles.push({
      x: this.gameWidth,
      y: Math.random() * (this.gameHeight - 100),
      width: 30,
      height: 30,
      type: 'bonus',
      color: '#FFD700',
      passed: false,
      value: 5
    });
  }

  reset() {
    this.obstacles = [];
    this.spawnTimer = 0;
    this.baseSpawnChance = 0.02;
  }

  scaleDifficulty(playerScore) {
    this.baseSpawnChance = Math.min(0.1, 0.02 + playerScore * 0.003);
    this.baseSpeed = 100 + playerScore * 2;
  }
}
```
