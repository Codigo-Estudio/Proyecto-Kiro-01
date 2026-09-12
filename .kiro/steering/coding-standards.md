# Coding Standards for JavaScript Game Development

This steering file defines coding standards, patterns, and conventions for JavaScript game development in the Kiro project.

## Table of Contents

- [Class Naming Conventions](#class-naming-conventions)
- [JavaScript Game Patterns](#javascript-game-patterns)
- [Performance Optimization Guidelines](#performance-optimization-guidelines)
- [File Organization](#file-organization)

---

## Class Naming Conventions

### General Rules

- Use **PascalCase** for class names
- Use descriptive, meaningful names that clearly indicate the class'"'"'s purpose
- Prefix interface names with `I` (e.g., `IDrawable`, `IUpdateable`)
- Use singular nouns for class names (e.g., `Player`, `Enemy`, `Bullet`)

### Specific Patterns

| Pattern | Example | Description |
|---------|---------|-------------|
| Entity Classes | `Player`, `Enemy`, `PowerUp` | Game objects that exist in the world |
| Component Classes | `PhysicsComponent`, `RenderComponent` | Reusable behaviors attached to entities |
| Manager Classes | `AssetManager`, `InputManager`, `GameManager` | Singleton-like controllers for resources |
| System Classes | `CollisionSystem`, `RenderingSystem` | Processing units that operate on entities |

### Constructor Patterns

```javascript
// Good: Clear, concise constructor
class Player {
  constructor(x, y, options = {}) {
    this.x = x;
    this.y = y;
    this.speed = options.speed || 200;
    this.health = options.health || 100;
  }
}

// Avoid: Unclear parameter order or missing defaults
class Player {
  constructor(a, b, c, d) {
    this.x = a;
    this.y = b;
    this.z = c;
    this.w = d;
  }
}
```

---

## JavaScript Game Patterns

### 1. Game Loop Pattern

Use a consistent game loop pattern for frame updates:

```javascript
class Game {
  constructor() {
    this.lastTime = 0;
    this.entities = [];
    this.isRunning = false;
  }

  start() {
    if (this.isRunning) return;
    this.isRunning = true;
    this.lastTime = performance.now();
    requestAnimationFrame((timestamp) => this.loop(timestamp));
  }

  loop(timestamp) {
    if (!this.isRunning) return;
    
    const deltaTime = (timestamp - this.lastTime) / 1000;
    this.lastTime = timestamp;
    
    this.update(deltaTime);
    this.render();
    
    requestAnimationFrame((timestamp) => this.loop(timestamp));
  }

  update(deltaTime) {
    this.entities.forEach(entity => entity.update(deltaTime));
  }

  render() {
    // Clear canvas
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    
    // Render entities
    this.entities.forEach(entity => entity.render(this.ctx));
  }
}
```

### 2. Component-Based Entity Pattern

Use composition over inheritance for game entities:

```javascript
class Entity {
  constructor(x, y) {
    this.x = x;
    this.y = y;
    this.components = new Map();
  }

  addComponent(name, component) {
    this.components.set(name, component);
    component.entity = this;
    return this;
  }

  getComponent(name) {
    return this.components.get(name);
  }

  hasComponent(name) {
    return this.components.has(name);
  }

  update(deltaTime) {
    this.components.forEach(component => {
      if (component.update) component.update(deltaTime);
    });
  }

  render(ctx) {
    this.components.forEach(component => {
      if (component.render) component.render(ctx);
    });
  }
}

// Example usage
const player = new Entity(100, 100)
  .addComponent('"'"'physics'"'"', new PhysicsComponent({ speed: 200 }))
  .addComponent('"'"'render'"'"', new SpriteRenderComponent('"'"'assets/ghosty.png'"'"'))
  .addComponent('"'"'input'"'"', new InputComponent());
```

### 3. Asset Management Pattern

Load and cache assets efficiently:

```javascript
class AssetManager {
  constructor() {
    this.assets = new Map();
    this.loadQueue = [];
  }

  async load(url, type = '"'"'image'"'"') {
    return new Promise((resolve, reject) => {
      if (this.assets.has(url)) {
        resolve(this.assets.get(url));
        return;
      }

      const loadRequest = new Promise((resolve, reject) => {
        const asset = type === '"'"'image'"'"' ? new Image() : new Audio();
        
        asset.onload = () => {
          this.assets.set(url, asset);
          resolve(asset);
        };
        
        asset.onerror = () => reject(new Error(`Failed to load: ${url}`));
        asset.src = url;
      });

      this.loadQueue.push(loadRequest);
      loadRequest.then(resolve, reject);
    });
  }

  async waitForAll() {
    await Promise.all(this.loadQueue);
  }

  get(url) {
    return this.assets.get(url);
  }

  has(url) {
    return this.assets.has(url);
  }
}
```

### 4. State Management Pattern

Use a clear state machine for game states:

```javascript
const GameState = {
  MENU: '"'"'menu'"'"',
  PLAYING: '"'"'playing'"'"',
  PAUSED: '"'"'paused'"'"',
  GAME_OVER: '"'"'gameover'"'"'
};

class StateManager {
  constructor() {
    this.states = new Map();
    this.currentState = null;
    this.previousState = null;
  }

  addState(name, state) {
    this.states.set(name, state);
  }

  transition(toState, data = {}) {
    if (this.currentState) {
      this.currentState.exit(data);
      this.previousState = this.currentState;
    }

    this.currentState = this.states.get(toState);
    this.currentState.enter(data);
  }

  update(deltaTime) {
    if (this.currentState) this.currentState.update(deltaTime);
  }

  render(ctx) {
    if (this.currentState) this.currentState.render(ctx);
  }
}
```

### 5. Collision Detection Pattern

Implement efficient collision detection:

```javascript
class CollisionSystem {
  static aabb(a, b) {
    return (
      a.x < b.x + b.width &&
      a.x + a.width > b.x &&
      a.y < b.y + b.height &&
      a.y + a.height > b.y
    );
  }

  static circle(a, b) {
    const dx = a.x - b.x;
    const dy = a.y - b.y;
    const distance = Math.sqrt(dx * dx + dy * dy);
    return distance < a.radius + b.radius;
  }

  static checkAll(entities, detection = '"'"'aabb'"'"') {
    const collisions = [];
    
    for (let i = 0; i < entities.length; i++) {
      for (let j = i + 1; j < entities.length; j++) {
        const a = entities[i];
        const b = entities[j];
        
        if (detection === '"'"'aabb'"'"' && CollisionSystem.aabb(a, b)) {
          collisions.push({ entityA: a, entityB: b });
        } else if (detection === '"'"'circle'"'"' && CollisionSystem.circle(a, b)) {
          collisions.push({ entityA: a, entityB: b });
        }
      }
    }
    
    return collisions;
  }
}
```

---

## Performance Optimization Guidelines

### 1. Rendering Optimization

- Use object pooling for frequently created/destroyed objects (bullets, particles)
- Batch render calls when possible
- Use `requestAnimationFrame` for smooth 60fps updates
- Avoid allocations in the game loop

```javascript
// Object Pool Pattern
class ObjectPool {
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
}

// Usage
const bulletPool = new ObjectPool(
  () => new Bullet(0, 0, 0),
  bullet => bullet.reset(0, 0, 0)
);

// In game loop
const bullet = bulletPool.acquire();
bullet.activate(playerX, playerY, velocity);
// When bullet dies
bulletPool.release(bullet);
```

### 2. Memory Management

- Clean up event listeners when entities are destroyed
- Release audio resources when no longer needed
- Use weak references where appropriate

```javascript
class Entity {
  constructor() {
    this.eventListeners = [];
  }

  addEventListener(event, callback) {
    this.eventListeners.push({ event, callback });
    // ... add actual listener
  }

  destroy() {
    this.eventListeners.forEach(({ event, callback }) => {
      // ... remove actual listener
    });
    this.eventListeners = [];
  }
}
```

### 3. Math Optimization

- Cache frequently used math operations
- Use bit shifting for powers of 2
- Consider using `Math.fround()` for single-precision floats

```javascript
// Cache expensive operations
const PI_2 = Math.PI * 2;

class MathUtils {
  static randomRange(min, max) {
    return min + Math.random() * (max - min);
  }

  static clamp(value, min, max) {
    return Math.min(Math.max(value, min), max);
  }

  static lerp(start, end, t) {
    return start * (1 - t) + end * t;
  }

  static distance(x1, y1, x2, y2) {
    const dx = x2 - x1;
    const dy = y2 - y1;
    return Math.sqrt(dx * dx + dy * dy);
  }
}
```

### 4. Asset Loading Optimization

- Preload critical assets before game start
- Use sprite sheets for related images
- Compress audio files appropriately

```javascript
// Preload sequence
async function preloadAssets(assetManager) {
  try {
    await assetManager.load('"'"'assets/sprites/ghosty.png'"'"', '"'"'image'"'"');
    await assetManager.load('"'"'assets/audio/jump.wav'"'"', '"'"'audio'"'"');
    await assetManager.load('"'"'assets/audio/game_over.wav'"'"', '"'"'audio'"'"');
    console.log('"'"'All assets loaded successfully'"'"');
  } catch (error) {
    console.error('"'"'Asset loading failed:'"'"', error);
  }
}
```

### 5. Update Loop Optimization

- Sort entities by update frequency
- Use spatial partitioning for large numbers of entities
- Skip updates for off-screen entities when possible

```javascript
// Spatial grid for collision optimization
class SpatialGrid {
  constructor(cellSize) {
    this.cellSize = cellSize;
    this.grid = new Map();
  }

  getKey(x, y) {
    const col = Math.floor(x / this.cellSize);
    const row = Math.floor(y / this.cellSize);
    return `${col},${row}`;
  }

  clear() {
    this.grid.clear();
  }

  add(entity) {
    const key = this.getKey(entity.x, entity.y);
    if (!this.grid.has(key)) {
      this.grid.set(key, []);
    }
    this.grid.get(key).push(entity);
  }

  getNearby(x, y) {
    const key = this.getKey(x, y);
    return this.grid.get(key) || [];
  }
}
```

---

## File Organization

```
src/
+-- game/               # Game-specific code
¦   +-- entities/      # Entity definitions
¦   +-- systems/       # Game systems
¦   +-- states/        # Game states
¦   +-- utils/         # Game utilities
+-- core/              # Core framework code
¦   +-- components/    # Base components
¦   +-- systems/       # Framework systems
¦   +-- utils/         # Core utilities
+-- assets/            # Game assets (referenced by path)
+-- main.js            # Entry point
+-- config.js          # Configuration
```

---

## Style Guidelines

### Code Formatting

- Use 2-space indentation
- Use semicolons at the end of statements
- Prefer single quotes for strings
- Use strict equality (`===`) over loose equality (`==`)

### Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Constants | UPPERCASE | `MAX_SPEED`, `GRAVITY` |
| Variables | camelCase | `playerScore`, `gameTime` |
| Functions | camelCase | `updateScore()`, `checkCollision()` |
| Classes | PascalCase | `Player`, `GameLoop` |
| Private members | _prefix | `_x`, `_updateTimer` |

### Documentation

```javascript
/**
 * Represents a player entity in the game
 * @extends Entity
 */
class Player extends Entity {
  /**
   * Creates a new player
   * @param {number} x - Initial x position
   * @param {number} y - Initial y position
   * @param {Object} options - Player configuration
   * @param {number} [options.speed=200] - Movement speed
   * @param {number} [options.health=100] - Starting health
   */
  constructor(x, y, options = {}) {
    super(x, y);
    this.speed = options.speed || 200;
    this.health = options.health || 100;
  }
}
```

---

## Testing Standards

- Write unit tests for all critical game logic
- Use a testing framework like Jest or Mocha
- Aim for at least 80% code coverage
- Include integration tests for core game flows

```javascript
// Example test structure
describe('"'"'Player'"'"', () => {
  let player;

  beforeEach(() => {
    player = new Player(100, 100);
  });

  describe('"'"'update()'"'"', () => {
    it('"'"'moves player when input is applied'"'"', () => {
      player.input.right = true;
      player.update(0.016); // 60fps
      expect(player.x).toBeGreaterThan(100);
    });
  });

  describe('"'"'takeDamage()'"'"', () => {
    it('"'"'reduces health'"'"', () => {
      player.takeDamage(25);
      expect(player.health).toBe(75);
    });
  });
});
```
