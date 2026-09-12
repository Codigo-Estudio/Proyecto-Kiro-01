# Game Architecture Standards for Flappy Kiro

This steering file defines modular game architecture patterns, event handling systems, and state management for the Flappy Kiro game project.

## Table of Contents

- [Modular System Architecture](#modular-system-architecture)
- [Event Handling Patterns](#event-handling-patterns)
- [State Management](#state-management)
- [Flappy Kiro Specific Architecture](#flappy-kiro-specific-architecture)

---

## Modular System Architecture

### System-Based Architecture Pattern

```javascript
// Base System Class
class System {
  constructor(game) {
    this.game = game;
    this.entities = [];
    this.enabled = true;
  }

  add(entity) {
    if (!this.entities.includes(entity)) {
      this.entities.push(entity);
    }
  }

  remove(entity) {
    const index = this.entities.indexOf(entity);
    if (index > -1) {
      this.entities.splice(index, 1);
    }
  }

  update(deltaTime) {
    if (!this.enabled) return;
    this.entities.forEach(entity => this.updateEntity(entity, deltaTime));
  }

  updateEntity(entity, deltaTime) {
    // Override in subclass
  }

  render(ctx) {
    if (!this.enabled) return;
    this.entities.forEach(entity => this.renderEntity(entity, ctx));
  }

  renderEntity(entity, ctx) {
    // Override in subclass
  }

  destroy() {
    this.entities = [];
    this.enabled = false;
  }
}

// Game Manager
class GameManager {
  constructor() {
    this.systems = new Map();
    this.entities = [];
    this.lastTime = 0;
    this.isRunning = false;
  }

  addSystem(name, system) {
    this.systems.set(name, system);
    system.game = this;
  }

  addEntity(entity) {
    this.entities.push(entity);
    this.systems.forEach(system => {
      if (system.shouldAdd(entity)) {
        system.add(entity);
      }
    });
  }

  removeEntity(entity) {
    const index = this.entities.indexOf(entity);
    if (index > -1) {
      this.entities.splice(index, 1);
      this.systems.forEach(system => system.remove(entity));
    }
  }

  update(deltaTime) {
    this.systems.forEach(system => system.update(deltaTime));
  }

  render(ctx) {
    this.systems.forEach(system => system.render(ctx));
  }

  start() {
    if (this.isRunning) return;
    this.isRunning = true;
    this.lastTime = performance.now();
    this.gameLoop();
  }

  stop() {
    this.isRunning = false;
  }

  gameLoop(timestamp) {
    if (!this.isRunning) return;
    
    const deltaTime = (timestamp - this.lastTime) / 1000;
    this.lastTime = timestamp;
    
    this.update(deltaTime);
    
    // Render only when canvas context is available
    if (this.ctx) {
      this.ctx.clearRect(0, 0, this.ctx.canvas.width, this.ctx.canvas.height);
      this.render(this.ctx);
    }
    
    requestAnimationFrame(timestamp => this.gameLoop(timestamp));
  }
}
```

### Component-Based Entity Pattern

```javascript
// Base Component
class Component {
  constructor(options = {}) {
    this.enabled = true;
    this.entity = null;
  }

  init() {}
  
  update(deltaTime) {}
  
  render(ctx) {}
  
  destroy() {
    this.enabled = false;
  }
}

// Entity with Component System
class Entity {
  constructor(id) {
    this.id = id || Entity.generateId();
    this.components = new Map();
    this.active = true;
  }

  static generateId() {
    return `entity-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  addComponent(name, component) {
    component.entity = this;
    this.components.set(name, component);
    if (component.init) component.init();
    return this;
  }

  removeComponent(name) {
    const component = this.components.get(name);
    if (component) {
      component.destroy();
      this.components.delete(name);
    }
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
      if (component.enabled && component.update) {
        component.update(deltaTime);
      }
    });
  }

  render(ctx) {
    this.components.forEach(component => {
      if (component.enabled && component.render) {
        component.render(ctx);
      }
    });
  }

  destroy() {
    this.components.forEach(component => component.destroy());
    this.components.clear();
    this.active = false;
  }
}

// Component Manager
class ComponentManager {
  constructor() {
    this.components = new Map();
    this.nextId = 0;
  }

  createComponent(type, options = {}) {
    const id = this.nextId++;
    const component = new type(options);
    component.id = id;
    this.components.set(id, component);
    return component;
  }

  destroyComponent(id) {
    const component = this.components.get(id);
    if (component) {
      component.destroy();
      this.components.delete(id);
    }
  }
}
```

### Specific Systems for Flappy Kiro

```javascript
// Physics System
class PhysicsSystem extends System {
  constructor(game) {
    super(game);
    this.gravity = 980; // pixels per second squared
    this.terminalVelocity = 600;
  }

  shouldAdd(entity) {
    return entity.hasComponent('physics');
  }

  updateEntity(entity, deltaTime) {
    const physics = entity.getComponent('physics');
    if (!physics || !physics.enabled) return;

    // Apply gravity
    physics.velocity.y += this.gravity * deltaTime;
    physics.velocity.y = Math.min(physics.velocity.y, this.terminalVelocity);

    // Update position
    entity.x += physics.velocity.x * deltaTime;
    entity.y += physics.velocity.y * deltaTime;

    // Collision with ground
    if (entity.y + entity.height > this.game.groundLevel) {
      entity.y = this.game.groundLevel - entity.height;
      physics.velocity.y = 0;
    }
  }
}

// Rendering System
class RenderSystem extends System {
  constructor(game) {
    super(game);
    this.ctx = game.ctx;
  }

  shouldAdd(entity) {
    return entity.hasComponent('render');
  }

  renderEntity(entity, ctx) {
    const render = entity.getComponent('render');
    if (!render || !render.enabled) return;

    ctx.save();
    ctx.translate(entity.x, entity.y);
    render.draw(ctx, entity);
    ctx.restore();
  }
}

// Input System
class InputSystem extends System {
  constructor(game) {
    super(game);
    this.keys = new Map();
    this.mouse = { x: 0, y: 0, button: false };
  }

  shouldAdd(entity) {
    return entity.hasComponent('input');
  }

  init() {
    this.bindEvents();
  }

  bindEvents() {
    window.addEventListener('keydown', e => {
      this.keys.set(e.code, true);
      this.handleInput(e.code, 'keydown');
    });

    window.addEventListener('keyup', e => {
      this.keys.set(e.code, false);
      this.handleInput(e.code, 'keyup');
    });

    window.addEventListener('mousedown', e => {
      this.mouse.button = true;
      this.handleInput('click', 'mousedown');
    });

    window.addEventListener('mouseup', e => {
      this.mouse.button = false;
    });

    window.addEventListener('mousemove', e => {
      const rect = this.game.canvas.getBoundingClientRect();
      this.mouse.x = e.clientX - rect.left;
      this.mouse.y = e.clientY - rect.top;
    });

    window.addEventListener('touchstart', e => {
      e.preventDefault();
      const rect = this.game.canvas.getBoundingClientRect();
      this.mouse.x = e.touches[0].clientX - rect.left;
      this.mouse.y = e.touches[0].clientY - rect.top;
      this.mouse.button = true;
      this.handleInput('touch', 'touchstart');
    }, { passive: false });

    window.addEventListener('touchend', e => {
      this.mouse.button = false;
    });
  }

  handleInput(input, type) {
    this.entities.forEach(entity => {
      const inputComponent = entity.getComponent('input');
      if (inputComponent && inputComponent.onInput) {
        inputComponent.onInput(input, type, entity);
      }
    });
  }

  isPressed(key) {
    return this.keys.get(key) || false;
  }

  isClicked() {
    return this.mouse.button;
  }
}
```

---

## Event Handling Patterns

### Event Bus Pattern

```javascript
class EventBus {
  constructor() {
    this.events = new Map();
  }

  on(eventType, callback, context = null) {
    if (!this.events.has(eventType)) {
      this.events.set(eventType, []);
    }
    this.events.get(eventType).push({ callback, context });
  }

  off(eventType, callback) {
    if (!this.events.has(eventType)) return;
    
    const handlers = this.events.get(eventType);
    const index = handlers.findIndex(h => h.callback === callback);
    
    if (index > -1) {
      handlers.splice(index, 1);
    }
  }

  once(eventType, callback) {
    const wrapper = (data) => {
      this.off(eventType, wrapper);
      callback(data);
    };
    this.on(eventType, wrapper);
  }

  emit(eventType, data = {}) {
    if (!this.events.has(eventType)) return;
    
    const handlers = this.events.get(eventType);
    handlers.forEach(handler => {
      handler.callback.call(handler.context, data);
    });
  }

  clear(eventType) {
    if (eventType) {
      this.events.delete(eventType);
    } else {
      this.events.clear();
    }
  }
}

// Usage
const eventBus = new EventBus();

// Subscribe
eventBus.on('player.jump', (data) => {
  console.log('Player jumped!', data);
});

// Emit
eventBus.emit('player.jump', { height: 100, velocity: 300 });

// One-time listener
eventBus.once('game.over', (data) => {
  console.log('Game over!', data);
});
```

### Entity Component Events

```javascript
class EntityEvents {
  constructor(eventBus) {
    this.eventBus = eventBus;
  }

  emitCreated(entity) {
    this.eventBus.emit('entity.created', { entity });
  }

  emitDestroyed(entity) {
    this.eventBus.emit('entity.destroyed', { entity });
  }

  emitCollision(entity, other) {
    this.eventBus.emit('entity.collided', { entity, other });
  }

  emitScoreChanged(score, previousScore) {
    this.eventBus.emit('score.changed', { score, previousScore });
  }

  emitGameOver() {
    this.eventBus.emit('game.gameover');
  }

  emitGameStart() {
    this.eventBus.emit('game.start');
  }

  emitPipePassed(pipe) {
    this.eventBus.emit('pipe.passed', { pipe });
  }
}
```

### Input Event Wrapper

```javascript
class InputEvents {
  constructor(inputSystem) {
    this.input = inputSystem;
    this.handlers = new Map();
  }

  onJump(callback) {
    this.handlers.set('jump', callback);
    return () => this.handlers.delete('jump');
  }

  onMove(callback) {
    this.handlers.set('move', callback);
    return () => this.handlers.delete('move');
  }

  onClick(callback) {
    this.handlers.set('click', callback);
    return () => this.handlers.delete('click');
  }

  update(deltaTime) {
    if (this.input.isPressed('Space') || this.input.isPressed('ArrowUp')) {
      this.invokeHandler('jump', { type: 'jump' });
    }

    if (this.input.isPressed('ArrowLeft') || this.input.isPressed('KeyA')) {
      this.invokeHandler('move', { type: 'move', direction: -1, deltaTime });
    }

    if (this.input.isPressed('ArrowRight') || this.input.isPressed('KeyD')) {
      this.invokeHandler('move', { type: 'move', direction: 1, deltaTime });
    }

    if (this.input.isClicked()) {
      this.invokeHandler('click', { type: 'click', x: this.input.mouse.x, y: this.input.mouse.y });
    }
  }

  invokeHandler(type, data) {
    const handler = this.handlers.get(type);
    if (handler) {
      handler(data);
    }
  }
}
```

---

## State Management

### State Machine Pattern

```javascript
const State = {
  MENU: 'menu',
  PLAYING: 'playing',
  PAUSED: 'paused',
  GAME_OVER: 'gameover'
};

class StateMachine {
  constructor(game) {
    this.game = game;
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
    this.game.eventBus.emit(`state.transitioned`, { 
      from: this.previousState, 
      to: this.currentState 
    });
  }

  update(deltaTime) {
    if (this.currentState) {
      this.currentState.update(deltaTime);
    }
  }

  render(ctx) {
    if (this.currentState) {
      this.currentState.render(ctx);
    }
  }

  is(state) {
    return this.currentState && this.currentState.name === state;
  }
}

// State Base Class
class GameState {
  constructor(name, game) {
    this.name = name;
    this.game = game;
  }

  enter(data) {
    // Override in subclass
  }

  exit(data) {
    // Override in subclass
  }

  update(deltaTime) {
    // Override in subclass
  }

  render(ctx) {
    // Override in subclass
  }
}
```

### Game States for Flappy Kiro

```javascript
// Menu State
class MenuState extends GameState {
  constructor(game) {
    super(State.MENU, game);
  }

  enter() {
    // Create menu entities
    this.game.addEntity(new Entity('title-screen'));
    this.game.addEntity(new Entity('start-button'));
    
    this.game.eventBus.emit('state.enter', { state: this.name });
  }

  update(deltaTime) {
    // Check for start game input
    if (this.game.inputSystem.isPressed('Space') || this.game.inputSystem.isClicked()) {
      this.game.stateMachine.transition(State.PLAYING);
    }
  }

  render(ctx) {
    ctx.fillStyle = '#87CEEB';
    ctx.fillRect(0, 0, this.game.width, this.game.height);

    ctx.fillStyle = '#FFF';
    ctx.textAlign = 'center';
    ctx.font = '48px Arial';
    ctx.fillText('FLAPPY KIRO', this.game.width / 2, this.game.height / 3);

    ctx.font = '24px Arial';
    ctx.fillText('Press SPACE or Click to Start', this.game.width / 2, this.game.height / 2);
  }
}

// Playing State
class PlayingState extends GameState {
  constructor(game) {
    super(State.PLAYING, game);
    this.score = 0;
    this.gameOver = false;
  }

  enter() {
    this.score = 0;
    this.gameOver = false;
    
    // Clear existing entities
    this.game.entities = [];
    
    // Create game entities
    this.createPlayer();
    this.createPipes();
    this.createBackground();
    
    this.game.eventBus.emit('game.start', { score: 0 });
  }

  createPlayer() {
    const player = new Entity('player');
    player.x = 100;
    player.y = this.game.height / 2;
    player.width = 34;
    player.height = 24;
    
    player.addComponent('physics', {
      velocity: { x: 0, y: 0 },
      acceleration: { x: 0, y: 0 }
    });
    
    player.addComponent('render', {
      draw(ctx) {
        ctx.fillStyle = '#FFD700';
        ctx.fillRect(0, 0, player.width, player.height);
      }
    });
    
    player.addComponent('input', {
      onInput(input, type, entity) {
        if (type === 'mousedown' || (input === 'Space' && type === 'keydown')) {
          player.getComponent('physics').velocity.y = -300;
          // Play jump sound
          this.game.audioSystem.play('jump');
        }
      }
    });
    
    this.game.addEntity(player);
  }

  createPipes() {
    // Create initial pipes
    for (let i = 0; i < 3; i++) {
      this.spawnPipe(i * 300 + 400);
    }
  }

  spawnPipe(x) {
    const pipe = new Entity(`pipe-${Date.now()}`);
    pipe.x = x;
    pipe.width = 52;
    pipe.height = this.game.height;
    
    pipe.y = Math.random() * (this.game.height - 300);
    
    pipe.addComponent('physics', {
      velocity: { x: -100, y: 0 },
      acceleration: { x: 0, y: 0 }
    });
    
    pipe.addComponent('render', {
      draw(ctx) {
        ctx.fillStyle = '#228B22';
        ctx.fillRect(0, 0, pipe.width, this.game.height);
      }
    });
    
    this.game.addEntity(pipe);
  }

  update(deltaTime) {
    // Check collisions
    const player = this.game.entities.find(e => e.id === 'player');
    if (player) {
      // Ground collision
      if (player.y + player.height > this.game.height - 50) {
        this.gameOver = true;
      }

      // Pipe collisions
      this.game.entities.forEach(entity => {
        if (entity.id.startsWith('pipe-')) {
          if (player.x < entity.x + entity.width &&
              player.x + player.width > entity.x &&
              player.y < entity.y + entity.height &&
              player.y + player.height > entity.y) {
            this.gameOver = true;
          }
        }
      });
    }

    if (this.gameOver) {
      this.game.stateMachine.transition(State.GAME_OVER, { score: this.score });
    }
  }

  render(ctx) {
    // Draw background
    ctx.fillStyle = '#87CEEB';
    ctx.fillRect(0, 0, this.game.width, this.game.height);

    // Draw score
    ctx.fillStyle = '#FFF';
    ctx.textAlign = 'center';
    ctx.font = '36px Arial';
    ctx.fillText(this.score, this.game.width / 2, 50);
  }
}

// Game Over State
class GameOverState extends GameState {
  constructor(game) {
    super(State.GAME_OVER, game);
  }

  enter(data) {
    this.game.eventBus.emit('game.gameover', { score: data.score });
  }

  update(deltaTime) {
    if (this.game.inputSystem.isPressed('Space') || this.game.inputSystem.isClicked()) {
      this.game.stateMachine.transition(State.MENU);
    }
  }

  render(ctx) {
    ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
    ctx.fillRect(0, 0, this.game.width, this.game.height);

    ctx.fillStyle = '#FFF';
    ctx.textAlign = 'center';
    ctx.font = '48px Arial';
    ctx.fillText('GAME OVER', this.game.width / 2, this.game.height / 3);

    ctx.font = '24px Arial';
    ctx.fillText(`Score: ${this.game.scoreSystem.lastScore}`, this.game.width / 2, this.game.height / 2);
    ctx.fillText('Press SPACE or Click to Restart', this.game.width / 2, this.game.height / 2 + 50);
  }
}
```

### Game Context with State

```javascript
class Game {
  constructor(width, height, canvasId) {
    this.width = width;
    this.height = height;
    this.canvas = document.getElementById(canvasId);
    this.ctx = this.canvas.getContext('2d');
    this.canvas.width = width;
    this.canvas.height = height;

    // Systems
    this.eventBus = new EventBus();
    this.entityEvents = new EntityEvents(this.eventBus);
    this.inputSystem = new InputSystem(this);
    this.physicsSystem = new PhysicsSystem(this);
    this.renderSystem = new RenderSystem(this);
    this.audioSystem = new AudioSystem();
    this.scoreSystem = new ScoreSystem(this);

    // Game entities
    this.entities = [];

    // State machine
    this.stateMachine = new StateMachine(this);
    this.stateMachine.addState(State.MENU, new MenuState(this));
    this.stateMachine.addState(State.PLAYING, new PlayingState(this));
    this.stateMachine.addState(State.GAME_OVER, new GameOverState(this));

    // Initialize
    this.stateMachine.transition(State.MENU);

    // Start game loop
    this.stateMachine.transition(State.PLAYING);
  }

  update(deltaTime) {
    this.stateMachine.update(deltaTime);
    this.physicsSystem.update(deltaTime);
    this.inputSystem.update(deltaTime);
  }

  render() {
    this.stateMachine.render(this.ctx);
    this.renderSystem.render(this.ctx);
  }

  start() {
    let lastTime = performance.now();
    
    const loop = (timestamp) => {
      const deltaTime = (timestamp - lastTime) / 1000;
      lastTime = timestamp;
      
      this.update(deltaTime);
      this.render();
      
      requestAnimationFrame(loop);
    };

    requestAnimationFrame(loop);
  }
}
```
