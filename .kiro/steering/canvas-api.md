# Canvas API and Game Rendering Steering File

This steering file defines patterns for Canvas API usage, animation frame handling, and efficient collision detection algorithms for the Kiro game project.

## Table of Contents

- [Canvas API Patterns](#canvas-api-patterns)
- [Animation Frame Handling](#animation-frame-handling)
- [Efficient Collision Detection Algorithms](#efficient-collision-detection-algorithms)
- [Rendering Best Practices](#rendering-best-practices)

---

## Canvas API Patterns

### Canvas Setup and Configuration

```javascript
class GameCanvas {
  constructor(width, height) {
    this.canvas = document.createElement('canvas');
    this.canvas.width = width;
    this.canvas.height = height;
    this.ctx = this.canvas.getContext('2d', { alpha: false });
    this.width = width;
    this.height = height;
  }

  mount(elementId) {
    const container = document.getElementById(elementId);
    container.appendChild(this.canvas);
  }

  resize(width, height) {
    this.canvas.width = width;
    this.canvas.height = height;
    this.width = width;
    this.height = height;
  }

  clear() {
    this.ctx.clearRect(0, 0, this.width, this.height);
  }

  getPixel(x, y) {
    if (x < 0 || x >= this.width || y < 0 || y >= this.height) {
      return null;
    }
    return this.ctx.getImageData(x, y, 1, 1).data;
  }

  drawImage(image, x, y, width, height) {
    if (image && image.complete) {
      this.ctx.drawImage(image, x, y, width, height);
    }
  }
}
```

### Image and Sprite Handling

```javascript
class SpriteManager {
  constructor() {
    this.sprites = new Map();
  }

  async load(path) {
    if (this.sprites.has(path)) {
      return this.sprites.get(path);
    }

    return new Promise((resolve, reject) => {
      const img = new Image();
      img.src = path;
      img.onload = () => {
        this.sprites.set(path, img);
        resolve(img);
      };
      img.onerror = () => reject(new Error(`Failed to load sprite: ${path}`));
    });
  }

  get(path) {
    return this.sprites.get(path);
  }

  async loadMultiple(paths) {
    return Promise.all(paths.map(path => this.load(path)));
  }

  draw(ctx, path, x, y, width, height, rotation = 0, flipX = false, flipY = false) {
    const sprite = this.get(path);
    if (!sprite) return;

    ctx.save();
    ctx.translate(x + width / 2, y + height / 2);
    ctx.rotate(rotation);
    if (flipX) ctx.scale(-1, 1);
    if (flipY) ctx.scale(1, -1);
    ctx.drawImage(sprite, -width / 2, -height / 2, width, height);
    ctx.restore();
  }
}
```

### Drawing Utilities

```javascript
class DrawingUtils {
  static drawRoundedRect(ctx, x, y, width, height, radius = 4) {
    ctx.beginPath();
    ctx.moveTo(x + radius, y);
    ctx.lineTo(x + width - radius, y);
    ctx.quadraticCurveTo(x + width, y, x + width, y + radius);
    ctx.lineTo(x + width, y + height - radius);
    ctx.quadraticCurveTo(x + width, y + height, x + width - radius, y + height);
    ctx.lineTo(x + radius, y + height);
    ctx.quadraticCurveTo(x, y + height, x, y + height - radius);
    ctx.lineTo(x, y + radius);
    ctx.quadraticCurveTo(x, y, x + radius, y);
    ctx.closePath();
  }

  static drawCircle(ctx, x, y, radius, color) {
    ctx.beginPath();
    ctx.arc(x, y, radius, 0, Math.PI * 2);
    ctx.fillStyle = color;
    ctx.fill();
  }

  static drawLine(ctx, x1, y1, x2, y2, color = '#000', width = 1) {
    ctx.beginPath();
    ctx.moveTo(x1, y1);
    ctx.lineTo(x2, y2);
    ctx.strokeStyle = color;
    ctx.lineWidth = width;
    ctx.stroke();
  }

  static drawText(ctx, text, x, y, options = {}) {
    const {
      font = '16px Arial',
      color = '#000',
      align = 'left',
      baseline = 'top'
    } = options;

    ctx.font = font;
    ctx.fillStyle = color;
    ctx.textAlign = align;
    ctx.textBaseline = baseline;
    ctx.fillText(text, x, y);
  }
}
```

---

## Animation Frame Handling

### Game Loop Pattern

```javascript
class GameLoop {
  constructor() {
    this.lastTime = 0;
    this.accumulator = 0;
    this.frameTime = 1000 / 60; // 60 FPS target
    this.isRunning = false;
    this.updateCallback = null;
    this.renderCallback = null;
  }

  start() {
    if (this.isRunning) return;
    this.isRunning = true;
    this.lastTime = performance.now();
    requestAnimationFrame(timestamp => this.loop(timestamp));
  }

  stop() {
    this.isRunning = false;
  }

  loop(timestamp) {
    if (!this.isRunning) return;

    const deltaTime = timestamp - this.lastTime;
    this.lastTime = timestamp;
    this.accumulator += deltaTime;

    // Update at fixed time step
    while (this.accumulator >= this.frameTime) {
      if (this.updateCallback) {
        this.updateCallback(this.frameTime / 1000);
      }
      this.accumulator -= this.frameTime;
    }

    // Render as fast as possible
    if (this.renderCallback) {
      this.renderCallback(deltaTime / 1000);
    }

    requestAnimationFrame(timestamp => this.loop(timestamp));
  }

  onUpdate(callback) {
    this.updateCallback = callback;
  }

  onRender(callback) {
    this.renderCallback = callback;
  }

  setTargetFPS(fps) {
    this.frameTime = 1000 / fps;
  }
}
```

### Animation System

```javascript
class Animation {
  constructor(frames, frameDuration, loop = true) {
    this.frames = frames;
    this.frameDuration = frameDuration;
    this.loop = loop;
    this.currentFrame = 0;
    this.elapsed = 0;
    this.isFinished = false;
  }

  update(deltaTime) {
    if (this.isFinished) return;

    this.elapsed += deltaTime * 1000;
    
    if (this.elapsed >= this.frameDuration) {
      this.elapsed -= this.frameDuration;
      this.currentFrame++;

      if (this.currentFrame >= this.frames.length) {
        if (this.loop) {
          this.currentFrame = 0;
        } else {
          this.currentFrame = this.frames.length - 1;
          this.isFinished = true;
        }
      }
    }
  }

  getCurrentFrame() {
    return this.frames[this.currentFrame];
  }

  reset() {
    this.currentFrame = 0;
    this.elapsed = 0;
    this.isFinished = false;
  }
}

class AnimationManager {
  constructor() {
    this.animations = new Map();
  }

  addAnimation(name, animation) {
    this.animations.set(name, animation);
  }

  getAnimation(name) {
    return this.animations.get(name);
  }

  update(name, deltaTime) {
    const animation = this.animations.get(name);
    if (animation) {
      animation.update(deltaTime);
    }
  }
}
```

### Particle System with Canvas

```javascript
class Particle {
  constructor(x, y, options = {}) {
    this.x = x;
    this.y = y;
    this.vx = options.vx || (Math.random() - 0.5) * 100;
    this.vy = options.vy || (Math.random() - 0.5) * 100;
    this.life = options.life || 1.0;
    this.maxLife = options.life || 1.0;
    this.color = options.color || '#fff';
    this.size = options.size || 4;
    this.decay = options.decay || 1.0;
  }

  update(deltaTime) {
    this.x += this.vx * deltaTime;
    this.y += this.vy * deltaTime;
    this.life -= this.decay * deltaTime;
  }

  isDead() {
    return this.life <= 0;
  }

  draw(ctx) {
    ctx.globalAlpha = this.life / this.maxLife;
    ctx.fillStyle = this.color;
    ctx.beginPath();
    ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
    ctx.fill();
    ctx.globalAlpha = 1.0;
  }
}

class ParticleSystem {
  constructor() {
    this.particles = [];
    this.maxParticles = 100;
  }

  emit(x, y, count, options = {}) {
    for (let i = 0; i < count; i++) {
      if (this.particles.length >= this.maxParticles) {
        this.particles.shift();
      }
      this.particles.push(new Particle(x, y, options));
    }
  }

  update(deltaTime) {
    for (let i = this.particles.length - 1; i >= 0; i--) {
      this.particles[i].update(deltaTime);
      if (this.particles[i].isDead()) {
        this.particles.splice(i, 1);
      }
    }
  }

  draw(ctx) {
    this.particles.forEach(particle => particle.draw(ctx));
  }

  clear() {
    this.particles = [];
  }
}
```

---

## Efficient Collision Detection Algorithms

### AABB (Axis-Aligned Bounding Box)

```javascript
class AABB {
  constructor(x, y, width, height) {
    this.x = x;
    this.y = y;
    this.width = width;
    this.height = height;
  }

  get right() {
    return this.x + this.width;
  }

  get bottom() {
    return this.y + this.height;
  }

  containsPoint(x, y) {
    return x >= this.x && x < this.right && y >= this.y && y < this.bottom;
  }

  intersects(other) {
    return (
      this.x < other.right &&
      this.right > other.x &&
      this.y < other.bottom &&
      this.bottom > other.y
    );
  }

  contains(other) {
    return (
      this.x <= other.x &&
      this.right >= other.right &&
      this.y <= other.y &&
      this.bottom >= other.bottom
    );
  }
}

class AABBSystem {
  constructor() {
    this.entities = [];
  }

  add(entity) {
    this.entities.push(entity);
  }

  remove(entity) {
    const index = this.entities.indexOf(entity);
    if (index > -1) {
      this.entities.splice(index, 1);
    }
  }

  checkCollisions() {
    const collisions = [];
    
    for (let i = 0; i < this.entities.length; i++) {
      for (let j = i + 1; j < this.entities.length; j++) {
        const a = this.entities[i];
        const b = this.entities[j];
        
        if (a.bounds && b.bounds && a.bounds.intersects(b.bounds)) {
          collisions.push({ a, b });
        }
      }
    }
    
    return collisions;
  }
}
```

### Circle Collision Detection

```javascript
class Circle {
  constructor(x, y, radius) {
    this.x = x;
    this.y = y;
    this.radius = radius;
  }

  containsPoint(x, y) {
    const dx = x - this.x;
    const dy = y - this.y;
    return dx * dx + dy * dy <= this.radius * this.radius;
  }

  intersects(other) {
    const dx = this.x - other.x;
    const dy = this.y - other.y;
    const distance = Math.sqrt(dx * dx + dy * dy);
    return distance <= this.radius + other.radius;
  }
}

class CircleSystem {
  constructor() {
    this.entities = [];
  }

  add(entity) {
    this.entities.push(entity);
  }

  checkCollisions() {
    const collisions = [];
    
    for (let i = 0; i < this.entities.length; i++) {
      for (let j = i + 1; j < this.entities.length; j++) {
        const a = this.entities[i];
        const b = this.entities[j];
        
        if (a.circle && b.circle && a.circle.intersects(b.circle)) {
          collisions.push({ a, b });
        }
      }
    }
    
    return collisions;
  }
}
```

### Spatial Partitioning with Grid

```javascript
class SpatialGrid {
  constructor(cellSize, maxEntitiesPerCell = 4) {
    this.cellSize = cellSize;
    this.maxEntitiesPerCell = maxEntitiesPerCell;
    this.grid = new Map();
  }

  clear() {
    this.grid.clear();
  }

  getKey(x, y) {
    const col = Math.floor(x / this.cellSize);
    const row = Math.floor(y / this.cellSize);
    return `${col},${row}`;
  }

  add(entity) {
    const bounds = entity.bounds || entity.circle;
    if (!bounds) return;

    // Calculate all cells the entity occupies
    const cells = this.getOccupiedCells(bounds);
    
    cells.forEach(cellKey => {
      if (!this.grid.has(cellKey)) {
        this.grid.set(cellKey, []);
      }
      
      const cell = this.grid.get(cellKey);
      
      // Only add if not already in cell
      if (!cell.includes(entity)) {
        cell.push(entity);
      }
    });
  }

  getOccupiedCells(bounds) {
    const cells = new Set();
    
    if (bounds.x !== undefined) {
      // AABB bounds
      const leftCol = Math.floor(bounds.x / this.cellSize);
      const rightCol = Math.floor((bounds.right || bounds.x + bounds.width) / this.cellSize);
      const topRow = Math.floor(bounds.y / this.cellSize);
      const bottomRow = Math.floor((bounds.bottom || bounds.y + bounds.height) / this.cellSize);
      
      for (let col = leftCol; col <= rightCol; col++) {
        for (let row = topRow; row <= bottomRow; row++) {
          cells.add(this.getKey(col * this.cellSize, row * this.cellSize));
        }
      }
    } else if (bounds.radius !== undefined) {
      // Circle bounds
      const centerCol = Math.floor(bounds.x / this.cellSize);
      const centerRow = Math.floor(bounds.y / this.cellSize);
      const radiusCells = Math.ceil(bounds.radius / this.cellSize);
      
      for (let col = centerCol - radiusCells; col <= centerCol + radiusCells; col++) {
        for (let row = centerRow - radiusCells; row <= centerRow + radiusCells; row++) {
          cells.add(this.getKey(col * this.cellSize, row * this.cellSize));
        }
      }
    }
    
    return Array.from(cells);
  }

  getNearby(x, y) {
    const nearby = new Set();
    const cells = this.getOccupiedCells({ x, y, radius: 0 });
    
    cells.forEach(cellKey => {
      const cell = this.grid.get(cellKey);
      if (cell) {
        cell.forEach(entity => nearby.add(entity));
      }
    });
    
    return Array.from(nearby);
  }

  checkCollisions(detection = 'aabb') {
    const collisions = [];
    
    this.grid.forEach(entities => {
      for (let i = 0; i < entities.length; i++) {
        for (let j = i + 1; j < entities.length; j++) {
          const a = entities[i];
          const b = entities[j];
          
          if (detection === 'aabb' && a.bounds && b.bounds && a.bounds.intersects(b.bounds)) {
            collisions.push({ a, b });
          } else if (detection === 'circle' && a.circle && b.circle && a.circle.intersects(b.circle)) {
            collisions.push({ a, b });
          }
        }
      }
    });
    
    return collisions;
  }
}
```

### Sweep and Prune Algorithm

```javascript
class SweepAndPrune {
  constructor() {
    this.entities = [];
    this.axis = 'x'; // 'x' or 'y'
  }

  add(entity) {
    this.entities.push(entity);
  }

  updateBounds() {
    // Update entity bounds before sweep
    this.entities.forEach(entity => {
      if (entity.bounds) {
        entity.min = this.axis === 'x' ? entity.bounds.x : entity.bounds.y;
        entity.max = this.axis === 'x' 
          ? (entity.bounds.right || entity.bounds.x + entity.bounds.width)
          : (entity.bounds.bottom || entity.bounds.y + entity.bounds.height);
      }
    });
  }

  sweep() {
    // Sort entities by min bound
    this.entities.sort((a, b) => a.min - b.min);
    
    const collisions = [];
    
    for (let i = 0; i < this.entities.length; i++) {
      const a = this.entities[i];
      
      for (let j = i + 1; j < this.entities.length; j++) {
        const b = this.entities[j];
        
        // Early exit if no overlap
        if (b.min > a.max) break;
        
        // Check collision
        if (a.bounds && b.bounds && a.bounds.intersects(b.bounds)) {
          collisions.push({ a, b });
        }
      }
    }
    
    return collisions;
  }

  checkCollisions() {
    this.updateBounds();
    return this.sweep();
  }
}
```

---

## Rendering Best Practices

### Object Pooling for Canvas Objects

```javascript
class CanvasObjectPool {
  constructor(createFn, resetFn, initialSize = 10) {
    this.createFn = createFn;
    this.resetFn = resetFn;
    this.pool = [];
    
    for (let i = 0; i < initialSize; i++) {
      this.pool.push(this.createFn());
    }
  }

  acquire() {
    if (this.pool.length > 0) {
      return this.pool.pop();
    }
    return this.createFn();
  }

  release(obj) {
    this.resetFn(obj);
    this.pool.push(obj);
  }

  get size() {
    return this.pool.length;
  }
}

// Usage example for bullets
const bulletPool = new CanvasObjectPool(
  () => ({ x: 0, y: 0, width: 4, height: 4, active: false }),
  bullet => { bullet.active = false; },
  50
);
```

### Efficient Canvas Clearing

```javascript
class EfficientCanvas {
  constructor(width, height) {
    this.canvas = document.createElement('canvas');
    this.canvas.width = width;
    this.canvas.height = height;
    this.ctx = this.canvas.getContext('2d', { alpha: false });
    this.width = width;
    this.height = height;
    
    // Pre-create clear rect
    this.clearRect = { x: 0, y: 0, width, height };
  }

  clear() {
    this.ctx.fillRect(0, 0, this.width, this.height);
    // Alternative: this.ctx.clearRect(0, 0, this.width, this.height);
  }

  clearRegion(x, y, width, height) {
    this.ctx.clearRect(x, y, width, height);
  }
}
```

### Viewport Culling

```javascript
class Viewport {
  constructor(x, y, width, height) {
    this.x = x;
    this.y = y;
    this.width = width;
    this.height = height;
  }

  contains(entity) {
    if (!entity.bounds) return false;
    return (
      entity.bounds.x + entity.bounds.width > this.x &&
      entity.bounds.x < this.x + this.width &&
      entity.bounds.y + entity.bounds.height > this.y &&
      entity.bounds.y < this.y + this.height
    );
  }

  getVisibleEntities(entities) {
    return entities.filter(entity => this.contains(entity));
  }
}
```
