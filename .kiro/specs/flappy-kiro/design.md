# Design Document: Flappy Kiro

## Overview

Flappy Kiro is a browser-based endless scroller game where players guide a ghost character through a series of vertically arranged pipes. The game features a simple control scheme (spacebar or click to flap), progressive difficulty, and persistent high score tracking. The game runs in a 4:3 aspect ratio container with responsive layout capabilities.

## Architecture

The game follows a modular architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                      Game Manager                            │
│  (State management, scene transitions, main game loop)      │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌─────────────────┐ ┌───────────────┐ ┌──────────────────┐
│   Input Handler │ │ Physics Engine│ │   Pipe Spawner   │
│   (Controls)    │ │ (Movement,    │ │ (Pipe generation│
│                 │ │  gravity)     │ │   and movement)  │
└─────────────────┘ └───────────────┘ └──────────────────┘
        │                   │                   │
        ▼                   ▼                   ▼
┌─────────────────┐ ┌───────────────┐ ┌──────────────────┐
│ Collision       │ │ Score Manager │ │  Cloud Manager   │
│   Detector      │ │ (Scoring)     │ │ (Perspective     │
│                 │ │               │ │   effects)       │
└─────────────────┘ └───────────────┘ └──────────────────┘
        │                   │                   │
        ▼                   ▼                   ▼
┌─────────────────┐ ┌───────────────┐ ┌──────────────────┐
│   Renderer      │ │ Audio Manager │ │  Persistence     │
│   (Visuals)     │ │ (Sound FX)    │ │  (High Score)    │
└─────────────────┘ └───────────────┘ └──────────────────┘
        │                   │                   │
        ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                      Layout Manager                          │
│  (Responsive design, aspect ratio, window resize)           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                   Difficulty Manager                         │
│  (Progressive difficulty scaling)                            │
└───────────────────────────────────────���─────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                   Performance Monitor                        │
│  (Frame rate, memory usage optimization)                     │
└─────────────────────────────────────────────────────────────┘
```

## Components and Interfaces

### Core Components

1. **Game Manager**
   - Manages game states (Start, Playing, Game Over)
   - Controls main game loop
   - Coordinates component updates
   - Handles state transitions

2. **Input Handler**
   - Handles keyboard (spacebar), mouse (click), and touch inputs
   - Prioritizes most recent input when multiple occur
   - Maps inputs to game actions (flap)

3. **Physics Engine**
   - Applies gravity (500 px/s² downward)
   - Calculates flap velocity (200 px/s upward)
   - Uses delta time for frame-rate-independent movement
   - Updates ghost position each frame

4. **Pipe Spawner**
   - Creates pipes at regular intervals
   - Randomly positions pipe gaps (100-250px tall)
   - Ensures gap stays within bounds (50px minimum from edges)
   - Moves pipes leftward at configurable speed

5. **Cloud Manager**
   - Manages background cloud elements
   - Assigns transparency (30-70%)
   - Moves clouds at different speeds (near: 100px/s, far: 50px/s)
   - Recycles clouds that exit the screen

6. **Collision Detector**
   - Detects ghost-pipe intersections
   - Detects ghost-screen boundary intersections
   - Triggers game over state
   - Records collision time for visual effects

7. **Score Manager**
   - Tracks current score (pipes passed)
   - Manages high score persistence
   - Preserves final score on game over
   - Resets score on restart

8. **Renderer**
   - Draws game elements (ghost, pipes, clouds)
   - Renders UI elements (title, score, game over screen)
   - Implements state-specific rendering
   - Handles partial redraws on state changes

9. **Audio Manager**
   - Plays sound effects (jump, game over, score)
   - Manages background music loop
   - Gracefully degrades on audio errors
   - Handles audio context initialization

10. **Layout Manager**
    - Maintains 4:3 aspect ratio (with flexibility for small screens)
    - Handles window resize events
    - Repositions elements proportionally
    - Enforces maximum dimensions (1024x768)

11. **Difficulty Manager**
    - Increases pipe speed by 10% every 5 points
    - Decreases gap size by 15% every 10 points
    - Increases gravity by 5% every 15 points
    - Caps maximum pipe speed at 300px/s

12. **Persistence Manager**
    - Stores high score in local storage
    - Retrieves high score on load
    - Compares and updates high score on game over
    - Handles missing/faulty stored values

13. **Performance Monitor**
    - Tracks frame rate (target: 60 FPS)
    - Monitors memory usage (alert at 100MB)
    - Reduces visual effects on low FPS
    - Triggers memory cleanup when needed

14. **Reset Manager**
    - Moves ghost to starting position
    - Removes all active pipes
    - Resets score and difficulty
    - Resets cloud positions

## Collision Detection Algorithms

The collision detection system implements precise mathematical algorithms to detect interactions between the ghost character and game obstacles.

### Ghost Collision Detection - Circular Collision

The Ghost is modeled as a circle with radius = ghostWidth / 2 for precise collision detection. This circular representation provides smoother gameplay compared to rectangular bounding boxes.

**Implementation Details:**
- Ghost radius: `ghostRadius = ghost.width / 2 = 20 pixels`
- Ghost position: Center point at `(ghost.x, ghost.y)`
- Circular collision formula: `distance(center1, center2) < (radius1 + radius2)`
- Distance calculation using Pythagorean theorem: `sqrt((x2-x1)² + (y2-y1)²)`

**Advantages:**
- Smooth collision boundaries that match the ghost's circular sprite
- Consistent collision behavior regardless of orientation
- More natural gameplay feel for the ghost character

### Pipe Collision Detection - Rectangular Bounds

Pipes are rectangular obstacles with top and bottom segments. The collision detection uses AABB (Axis-Aligned Bounding Box) with an expanded boundary to account for the ghost's circular shape.

**Implementation Details:**
- Pipe rectangle: `(pipe.x, pipe.topPipeHeight)` to `(pipe.x + pipe.width, pipe.bottomPipeY)`
- Collision detection: Check if ghost circle center is within pipe rectangle expanded by ghost radius
- Formula: `ghostX - ghostRadius < pipeRight AND ghostX + ghostRadius > pipeLeft AND ghostY - ghostRadius < pipeBottom AND ghostY + ghostRadius > pipeTop`

**Algorithm:**
1. Find the closest point on the rectangle boundary to the circle center
2. Calculate distance from closest point to circle center
3. If distance < ghostRadius, collision has occurred

**Edge Cases:**
- Ghost completely inside pipe rectangle (center point collision)
- Ghost touching pipe corner (corner collision detection)
- Ghost brushing pipe edge (edge collision detection)

### Ground/Ceiling Detection

Screen boundaries are detected using simple boundary checks against the ghost's circular radius.

**Implementation Details:**
- Ground collision: `ghostY + ghostRadius > gameHeight`
- Ceiling collision: `ghostY - ghostRadius < 0`

**Calculation:**
- Game height: Determined by layout manager (default: 768 pixels)
- Ghost radius: 20 pixels
- Ground trigger: When ghost bottom edge touches screen bottom
- Ceiling trigger: When ghost top edge touches screen top

**Visual Feedback:**
- Both collisions trigger Game Over state
- Collision time recorded for explosion effects
- Collision type stored (ground/ceiling/pipe) for potential visual variations

### Combined Collision Detection Flow

The collision detection system follows a hierarchical approach with early-exit optimization:

```
Step 1: Check ground/ceiling (fast, simple)
  ├─ If ground collision detected → Return immediately
  └─ If ceiling collision detected → Return immediately

Step 2: Check pipe collisions (for each active pipe)
  ├─ For each pipe in activePipes:
  │  ├─ Calculate closest point on pipe rectangle
  │  ├─ Calculate distance from ghost center
  │  └─ If distance < ghostRadius → Return with pipe collision
  └─ If no pipe collisions → Continue

Step 3: Return collision result with details
  └─ Return { collided: false } if no collisions found
```

**Optimization:**
- Ground/ceiling checks are O(1) and checked first
- Pipe checks are O(n) where n = active pipes count
- Early exit after first collision minimizes unnecessary calculations
- Only active pipes (on-screen) are checked

### Collision Resolution

When a collision is detected, the system resolves it consistently:

**Steps:**
1. Trigger Game Over state in Game Manager
2. Record collision timestamp for visual effects
3. Store collision type for potential effects variation:
   - `'ground'`: Collision with bottom screen boundary
   - `'ceiling'`: Collision with top screen boundary  
   - `'pipe'`: Collision with pipe obstacle
4. If pipe collision, store pipe ID for potential effects

**Data Structure:**
```typescript
interface CollisionResult {
  collided: boolean;
  type?: 'ground' | 'ceiling' | 'pipe';
  pipeId?: string;  // Only present if type === 'pipe'
  timestamp: number;  // Game time when collision occurred
}
```

**Effect Integration:**
- Collision type can trigger different visual effects
- Timestamp enables synchronized particle effects
- Pipe ID allows for pipe-specific effects (if implemented)

### Code Implementation Examples

```typescript
// Circular collision detection between two circles
function isCircleColliding(circle1: Circle, circle2: Circle): boolean {
  const dx = circle1.x - circle2.x;
  const dy = circle1.y - circle2.y;
  const distance = Math.sqrt(dx * dx + dy * dy);
  return distance < (circle1.radius + circle2.radius);
}

// Ghost-pipe collision (ghost circle vs pipe rectangle)
function isGhostCollidingWithPipe(ghost: Ghost, pipe: Pipe): boolean {
  // Ghost is a circle centered at (ghost.x, ghost.y) with radius = ghost.width / 2
  const ghostRadius = ghost.width / 2;
  
  // Pipe rectangle boundaries
  const pipeLeft = pipe.x;
  const pipeRight = pipe.x + pipe.width;
  const pipeTop = pipe.topPipeHeight;
  const pipeBottom = pipe.bottomPipeY;
  
  // Find closest point on rectangle to circle center
  const closestX = Math.max(pipeLeft, Math.min(ghost.x, pipeRight));
  const closestY = Math.max(pipeTop, Math.min(ghost.y, pipeBottom));
  
  // Calculate distance from closest point to circle center
  const dx = ghost.x - closestX;
  const dy = ghost.y - closestY;
  const distance = Math.sqrt(dx * dx + dy * dy);
  
  return distance < ghostRadius;
}

// Ground collision detection
function isGroundCollision(ghost: Ghost, gameHeight: number): boolean {
  return (ghost.y + ghost.width / 2) > gameHeight;
}

// Ceiling collision detection
function isCeilingCollision(ghost: Ghost): boolean {
  return (ghost.y - ghost.width / 2) < 0;
}

// Combined collision detection system
interface CollisionResult {
  collided: boolean;
  type?: 'ground' | 'ceiling' | 'pipe';
  pipeId?: string;
  timestamp: number;
}

function detectCollision(ghost: Ghost, pipes: Pipe[], gameHeight: number, gameTime: number): CollisionResult {
  const ghostRadius = ghost.width / 2;
  
  // Step 1: Check ground collision (fast, simple)
  if (isGroundCollision(ghost, gameHeight)) {
    return { collided: true, type: 'ground', timestamp: gameTime };
  }
  
  // Step 2: Check ceiling collision
  if (isCeilingCollision(ghost)) {
    return { collided: true, type: 'ceiling', timestamp: gameTime };
  }
  
  // Step 3: Check pipe collisions
  for (const pipe of pipes) {
    if (isGhostCollidingWithPipe(ghost, pipe)) {
      return { 
        collided: true, 
        type: 'pipe', 
        pipeId: pipe.id,
        timestamp: gameTime 
      };
    }
  }
  
  // No collision detected
  return { collided: false, timestamp: gameTime };
}
```

### Integration with Game Components

The collision detection system integrates with existing components:

**Physics Engine:**
- Uses collision detection results to terminate movement
- Records final position at collision time

**Score Manager:**
- Pauses score increment on collision
- Finalizes score for high score comparison

**Renderer:**
- Triggers Game Over visual state
- Displays final score and high score

**Reset Manager:**
- Clears collision state on game restart
- Resets all game entities

## Data Models

### Ghost
```typescript
interface Ghost {
  x: number;              // Horizontal position (fixed)
  y: number;              // Vertical position (variable)
  velocity: number;       // Current vertical velocity
  width: number;          // 40 pixels
  height: number;         // 40 pixels
  color: string;          // "rgba(200, 200, 200, 0.9)"
}
```

### Pipe
```typescript
interface Pipe {
  id: string;             // Unique identifier
  x: number;              // Horizontal position
  gapY: number;           // Vertical position of gap center
  gapHeight: number;      // Height of gap (100-250px)
  topPipeHeight: number;  // Calculated from gapY - gapHeight/2
  bottomPipeY: number;    // Calculated from gapY + gapHeight/2
  passed: boolean;        // Whether ghost has passed this pipe
  width: number;          // 60 pixels
  color: string;          // "rgba(100, 200, 100, 1.0)"
}
```

### Cloud
```typescript
interface Cloud {
  id: string;             // Unique identifier
  x: number;              // Horizontal position
  y: number;              // Vertical position
  size: number;           // 50-100 pixels
  speed: number;          // 50 or 100 pixels per second
  opacity: number;        // 0.3-0.7
  color: string;          // "rgba(255, 255, 255, opacity)"
}
```

### Game State
```typescript
type GameState = 'START' | 'PLAYING' | 'GAME_OVER';

interface GameSession {
  state: GameState;
  score: number;
  highScore: number;
  ghost: Ghost;
  pipes: Pipe[];
  clouds: Cloud[];
  startTime: number;      // Timestamp when game started
  lastFrameTime: number;  // Timestamp of last frame
  gravity: number;        // Current gravity value
  pipeSpeed: number;      // Current pipe speed
}
```

### Difficulty Settings
```typescript
interface DifficultySettings {
  basePipeSpeed: number;      // 150 pixels per second
  baseGapHeight: number;      // 175 pixels (average of 100-250)
  baseGravity: number;        // 500 pixels per second squared
  currentPipeSpeed: number;
  currentGapHeight: number;
  currentGravity: number;
  scoreThresholds: {
    speedIncreaseAt: number;  // Every 5 points
    gapDecreaseAt: number;    // Every 10 points
    gravityIncreaseAt: number // Every 15 points
  };
  maxPipeSpeed: number;       // 300 pixels per second
}
```

## Configuration System

### Overview

The Configuration System provides a centralized approach to managing all numerical game parameters. This system separates game values from implementation code, enabling easy balance tuning and supporting multiple difficulty profiles.

**Purpose:**
- Centralize all numerical parameters for easy tuning and testing
- Separate code from values for maintainability
- Enable easy balance tuning without code changes
- Support multiple difficulty profiles (easy, normal, hard)

**Benefits:**
- **Maintainability**: All game parameters in one place
- **Balancing**: Easy to tweak values for game balance
- **Testing**: Can generate varied configurations for property-based tests
- **Difficulty Profiles**: Support easy, normal, and hard modes
- **Environment-based Configuration**: Can override values via environment variables

### Config Structure

The configuration system defines all game parameters in a hierarchical structure:

```typescript
interface GameConfig {
  // Physics constants
  gravity: number;              // 500 px/s²
  flapVelocity: number;         // 200 px/s
  maxDelta: number;             // 0.1 seconds (frame time cap)
  
  // Entity dimensions
  ghostWidth: number;           // 40 pixels
  ghostHeight: number;          // 40 pixels
  pipeWidth: number;            // 60 pixels
  
  // Pipe settings
  pipeSpeed: number;            // 150 px/s
  pipeGapMin: number;           // 100 pixels
  pipeGapMax: number;           // 250 pixels
  pipeMargin: number;           // 50 pixels (minimum from screen edges)
  
  // Cloud settings
  cloudCount: number;           // 3 background clouds
  cloudNearSpeed: number;       // 100 px/s (foreground clouds)
  cloudFarSpeed: number;        // 50 px/s (background clouds)
  cloudOpacityMin: number;      // 0.3 (minimum transparency)
  cloudOpacityMax: number;      // 0.7 (maximum transparency)
  
  // Scoring
  scoreIncrement: number;       // 1 point per pipe
  
  // Difficulty progression
  difficulty: DifficultySettings;
  
  // Performance
  targetFPS: number;            // 60 frames per second
  minFPS: number;               // 30 frames per second (threshold for degradation)
  memoryLimitMB: number;        // 100 MB (cleanup trigger)
  
  // Layout
  aspectRatio: number;          // 4/3 (game aspect ratio)
  maxWidth: number;             // 1024 pixels (maximum width)
  maxHeight: number;            // 768 pixels (maximum height)
  
  // Colors
  colors: Colors;
}

interface DifficultySettings {
  // Base values (initial difficulty)
  basePipeSpeed: number;        // 150 pixels per second
  baseGapHeight: number;        // 175 pixels (average of 100-250)
  baseGravity: number;          // 500 pixels per second squared
  
  // Progression thresholds (score milestones)
  speedIncreaseAt: number;      // Every 5 points, increase speed
  gapDecreaseAt: number;        // Every 10 points, decrease gap
  gravityIncreaseAt: number;    // Every 15 points, increase gravity
  
  // Multipliers (progression factors)
  speedMultiplier: number;      // 1.10 (10% speed increase per threshold)
  gapMultiplier: number;        // 0.85 (15% gap decrease per threshold)
  gravityMultiplier: number;    // 1.05 (5% gravity increase per threshold)
  
  // Caps (maximum values)
  maxPipeSpeed: number;         // 300 pixels per second (maximum)
}

interface Colors {
  ghost: string;                // 'rgba(200, 200, 200, 0.9)'
  pipe: string;                 // 'rgba(100, 200, 100, 1.0)'
  cloud: string;                // 'rgba(255, 255, 255, 0.5)'
}
```

### Config Profiles

Three difficulty profiles are provided for different player experiences:

```typescript
// Easy profile - relaxed gameplay for beginners
const easyProfile: Partial<GameConfig> = {
  pipeSpeed: 120,           // Slower pipes
  pipeGapMin: 150,          // Larger gaps
  gravity: 400,             // Slower fall
  cloudNearSpeed: 80,       // Slower clouds
  cloudFarSpeed: 40
};

// Normal profile - balanced gameplay
const normalProfile: Partial<GameConfig> = {
  pipeSpeed: 150,           // Standard speed
  pipeGapMin: 100,          // Standard gap
  gravity: 500,             // Standard gravity
  cloudNearSpeed: 100,
  cloudFarSpeed: 50
};

// Hard profile - challenging gameplay for experienced players
const hardProfile: Partial<GameConfig> = {
  pipeSpeed: 200,           // Faster pipes
  pipeGapMin: 80,           // Smaller gaps
  gravity: 600,             // Faster fall
  cloudNearSpeed: 120,      // Faster clouds
  cloudFarSpeed: 60
};
```

### Usage in Components

All game components reference configuration values instead of hard-coded numbers:

#### Physics Engine
```typescript
// Uses: config.gravity, config.flapVelocity, config.maxDelta
class PhysicsEngine {
  constructor(private config: GameConfig) {}
  
  applyGravity(ghost: Ghost, deltaTime: number): void {
    const gravityForce = this.config.gravity * deltaTime;
    ghost.velocity += gravityForce;
  }
  
  applyFlap(ghost: Ghost): void {
    ghost.velocity = -this.config.flapVelocity;
  }
}
```

#### Pipe Spawner
```typescript
// Uses: config.pipeSpeed, config.pipeGapMin, config.pipeGapMax, config.pipeMargin
class PipeSpawner {
  constructor(private config: GameConfig) {}
  
  createPipePair(screenHeight: number): Pipe {
    const gapHeight = this.randomGap();
    const gapY = this.randomGapPosition(screenHeight);
    
    return {
      gapY,
      gapHeight,
      topPipeHeight: gapY - gapHeight / 2,
      bottomPipeY: gapY + gapHeight / 2,
      // ... other properties
    };
  }
  
  private randomGap(): number {
    return this.randomRange(
      this.config.pipeGapMin,
      this.config.pipeGapMax
    );
  }
}
```

#### Cloud Manager
```typescript
// Uses: config.cloudCount, config.cloudNearSpeed, config.cloudFarSpeed, config.cloudOpacityMin, config.cloudOpacityMax
class CloudManager {
  constructor(private config: GameConfig) {}
  
  createCloud(isNear: boolean): Cloud {
    return {
      speed: isNear ? this.config.cloudNearSpeed : this.config.cloudFarSpeed,
      opacity: this.randomRange(
        this.config.cloudOpacityMin,
        this.config.cloudOpacityMax
      ),
      // ... other properties
    };
  }
}
```

#### Difficulty Manager
```typescript
// Uses: config.difficulty settings
class DifficultyManager {
  constructor(private config: GameConfig) {}
  
  calculateDifficulty(score: number): DifficultySettings {
    const speedMultiplier = Math.pow(
      this.config.difficulty.speedMultiplier,
      Math.floor(score / this.config.difficulty.speedIncreaseAt)
    );
    
    const gapMultiplier = Math.pow(
      this.config.difficulty.gapMultiplier,
      Math.floor(score / this.config.difficulty.gapDecreaseAt)
    );
    
    const gravityMultiplier = Math.pow(
      this.config.difficulty.gravityMultiplier,
      Math.floor(score / this.config.difficulty.gravityIncreaseAt)
    );
    
    return {
      currentPipeSpeed: Math.min(
        this.config.difficulty.basePipeSpeed * speedMultiplier,
        this.config.difficulty.maxPipeSpeed
      ),
      currentGapHeight: this.config.difficulty.baseGapHeight * gapMultiplier,
      currentGravity: this.config.difficulty.baseGravity * gravityMultiplier
    };
  }
}
```

### Configuration Loading Strategy

The configuration system supports multiple loading approaches:

#### Default Configuration
```typescript
const defaultConfig: GameConfig = {
  // Physics constants
  gravity: 500,
  flapVelocity: 200,
  maxDelta: 0.1,
  
  // Entity dimensions
  ghostWidth: 40,
  ghostHeight: 40,
  pipeWidth: 60,
  
  // Pipe settings
  pipeSpeed: 150,
  pipeGapMin: 100,
  pipeGapMax: 250,
  pipeMargin: 50,
  
  // Cloud settings
  cloudCount: 3,
  cloudNearSpeed: 100,
  cloudFarSpeed: 50,
  cloudOpacityMin: 0.3,
  cloudOpacityMax: 0.7,
  
  // Scoring
  scoreIncrement: 1,
  
  // Difficulty progression
  difficulty: {
    basePipeSpeed: 150,
    baseGapHeight: 175,
    baseGravity: 500,
    speedIncreaseAt: 5,
    gapDecreaseAt: 10,
    gravityIncreaseAt: 15,
    speedMultiplier: 1.10,
    gapMultiplier: 0.85,
    gravityMultiplier: 1.05,
    maxPipeSpeed: 300
  },
  
  // Performance
  targetFPS: 60,
  minFPS: 30,
  memoryLimitMB: 100,
  
  // Layout
  aspectRatio: 4/3,
  maxWidth: 1024,
  maxHeight: 768,
  
  // Colors
  colors: {
    ghost: 'rgba(200, 200, 200, 0.9)',
    pipe: 'rgba(100, 200, 100, 1.0)',
    cloud: 'rgba(255, 255, 255, 0.5)'
  }
};
```

#### Environment-based Override
```typescript
function loadConfig(): GameConfig {
  const config = { ...defaultConfig };
  
  // Environment variables (for development/testing)
  if (process.env.GRAVITY) {
    config.gravity = parseFloat(process.env.GRAVITY);
  }
  
  if (process.env.PIPE_SPEED) {
    config.pipeSpeed = parseFloat(process.env.PIPE_SPEED);
  }
  
  return config;
}
```

#### Runtime Configuration
```typescript
// Load config with potential overrides
const gameConfig = loadConfig();

// Initialize components with config
const physicsEngine = new PhysicsEngine(gameConfig);
const pipeSpawner = new PipeSpawner(gameConfig);
const cloudManager = new CloudManager(gameConfig);
const difficultyManager = new DifficultyManager(gameConfig);
```

### Property-Based Testing with Config

The configuration system enables property-based testing with varied parameters:

```typescript
// Generate varied configurations for testing
function* generateConfigs(): Generator<GameConfig> {
  const gravityValues = [300, 400, 500, 600, 700];
  const pipeSpeeds = [100, 150, 200, 250];
  const gapHeights = [80, 100, 150, 200, 250];
  
  for (const gravity of gravityValues) {
    for (const speed of pipeSpeeds) {
      for (const gap of gapHeights) {
        yield {
          ...defaultConfig,
          gravity,
          pipeSpeed: speed,
          pipeGapMin: gap,
          pipeGapMax: gap + 100
        };
      }
    }
  }
}

// Test with varied configurations
QC.property(
  "Game remains playable with varied physics",
  generateConfigs(),
  (config) => {
    const game = createGame(config);
    // Test gameplay logic with this configuration
    return true; // Implementation-specific test
  }
);
```

## Performance Optimization Guidelines

### FPS Target and Monitoring

- **Target**: 60 FPS minimum on modern devices
- **Frame time budget**: ~16.67ms per frame
- **FPS monitoring**: Using `requestAnimationFrame` timestamps via `performance.now()`
- **Performance threshold triggers**: FPS < 30 triggers degradation mode

**Implementation**: The Performance Monitor tracks frame rate and triggers degradation mode when FPS drops below 30, reducing visual effects and enabling simpler rendering paths.

### Efficient Sprite Batching

- **Batch entities by type**: Group all entities of the same type for single draw calls
- **Minimize state changes**: Reduce canvas context state changes between draw calls
- **Draw call ordering**: Render entities in order: clouds first, then pipes, then ghost, then UI
- **Pre-render static elements**: Render static elements to offscreen canvas when possible
- **Texture atlases**: Use texture atlases for sprites (ghost, pipes, clouds) to minimize texture switches

### Memory Management with Object Pooling

- **Object pooling**: Implement object pooling for pipes and clouds
- **Pre-allocation**: Pre-allocate pool size based on expected maximum entities
- **Recycling**: Recycle objects instead of creating/destroying
- **Pool size recommendations**: 10 pipes, 10 clouds
- **Garbage collection optimization**: Reusing objects minimizes garbage collection pressure

**Pool Size Justification**:
- 10 pipes: Allows for sufficient pipe spacing while handling screen transitions
- 10 clouds: Handles background depth effect with multiple cloud layers

### Performance Considerations for Each Component

#### Physics Engine
- **Delta time capping**: Cap delta time at 0.1 seconds maximum to prevent physics anomalies on lag spikes
- **Velocity updates**: Use explicit Euler integration for predictable behavior
- **Collision checks**: Only check active pipes, skip inactive ones

#### Pipe Spawner
- **Spawn condition**: Only spawn new pipe when previous pipe is fully off-screen
- **Pool reuse**: Acquire pipes from object pool, don't create new instances
- **Early cleanup**: Release pipes to pool immediately when fully off-screen

#### Cloud Manager
- **Recycling**: Recycle clouds instead of creating new ones
- **Pool management**: Maintain separate pools for near and far clouds
- **Position reset**: Reset cloud position to start when recycled

#### Collision Detector
- **Early exit optimization**: Check ground/ceiling first (O(1)), then pipes (O(n))
- **Active pipes only**: Only check pipes that are on-screen
- **Skip passed pipes**: Don't check pipes that the ghost has already passed

#### Renderer
- **Partial redraws**: Only redraw areas that changed state
- **Dirty rectangle tracking**: Track regions that need redrawing
- **Batch rendering**: Render all entities of same type together
- **Offscreen caching**: Pre-render static elements to offscreen canvas

#### Audio Manager
- **Pre-load assets**: Load all audio assets on initialization
- **Cache audio objects**: Reuse audio objects for repeated sounds
- **Graceful degradation**: Continue operation if audio fails to initialize

#### Performance Monitor
- **FPS tracking**: Monitor FPS using `performance.now()` timestamps
- **Memory monitoring**: Track memory usage via `performance.memory` (if available)
- **Degradation triggers**: Enable degradation mode when FPS < 30 or memory > 80MB
- **Cleanup triggers**: Trigger cleanup when memory > 100MB

### Code Implementation Examples

```typescript
// Object Pool for Reusable Entities
class ObjectPool<T> {
  private pool: T[] = [];
  private createFn: () => T;
  
  constructor(createFn: () => T, initialSize: number = 10) {
    this.createFn = createFn;
    for (let i = 0; i < initialSize; i++) {
      this.pool.push(createFn());
    }
  }
  
  acquire(): T {
    const item = this.pool.pop();
    if (!item) return this.createFn();
    return item;
  }
  
  release(item: T): void {
    this.pool.push(item);
  }
  
  get size(): number {
    return this.pool.length;
  }
}

// FPS Monitor
class FPSMonitor {
  private lastTime: number = 0;
  private frameCount: number = 0;
  private fps: number = 0;
  
  start(): void {
    this.lastTime = performance.now();
    this.frameCount = 0;
    this.measure();
  }
  
  measure(): void {
    const now = performance.now();
    this.frameCount++;
    
    if (now - this.lastTime >= 1000) {
      this.fps = Math.round((this.frameCount * 1000) / (now - this.lastTime));
      this.frameCount = 0;
      this.lastTime = now;
    }
  }
  
  getFPS(): number {
    return this.fps;
  }
  
  shouldDegradate(): boolean {
    return this.fps < 30;
  }
}

// Sprite Batcher for Efficient Rendering
class SpriteBatcher {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private entities: { type: string; render: (ctx: CanvasRenderingContext2D) => void }[] = [];
  
  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d')!;
  }
  
  addEntity(type: string, render: (ctx: CanvasRenderingContext2D) => void): void {
    this.entities.push({ type, render });
  }
  
  render(): void {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    
    // Batch by type for efficiency
    const types = [...new Set(this.entities.map(e => e.type))];
    
    for (const type of types) {
      // Sort entities of same type
      const typeEntities = this.entities.filter(e => e.type === type);
      
      // Render all entities of this type
      for (const entity of typeEntities) {
        entity.render(this.ctx);
      }
    }
  }
  
  clear(): void {
    this.entities = [];
  }
}

// Performance Monitor with Memory Tracking
class PerformanceMonitor {
  private fpsMonitor: FPSMonitor;
  private memoryLimitMB: number;
  private degradationEnabled: boolean = false;
  
  constructor(memoryLimitMB: number = 100) {
    this.fpsMonitor = new FPSMonitor();
    this.memoryLimitMB = memoryLimitMB;
  }
  
  start(): void {
    this.fpsMonitor.start();
    this.degradationEnabled = false;
  }
  
  update(): void {
    this.fpsMonitor.measure();
    
    if (this.fpsMonitor.shouldDegradate()) {
      this.enableDegradation();
    }
  }
  
  enableDegradation(): void {
    if (!this.degradationEnabled) {
      this.degradationEnabled = true;
      console.log('Performance degradation enabled (FPS < 30)');
    }
  }
  
  shouldRenderHighQuality(): boolean {
    return !this.degradationEnabled;
  }
}

// Memory Manager with Cleanup Strategy
class MemoryManager {
  private cleanupTriggerMB: number = 100;
  
  monitorMemory(): boolean {
    if (!performance.memory) return false;
    
    const usedMemoryMB = performance.memory.usedJSHeapSize / (1024 * 1024);
    return usedMemoryMB > this.cleanupTriggerMB;
  }
  
  cleanup(): void {
    console.log('Memory cleanup triggered');
    // Cleanup hierarchy: effects → animations → cached assets
    this.clearEffects();
    this.clearAnimations();
    this.clearCachedAssets();
    this.forceGarbageCollection();
  }
  
  private clearEffects(): void {
    // Clear particle effects, temporary visual elements
  }
  
  private clearAnimations(): void {
    // Clear animation caches, temporary transforms
  }
  
  private clearCachedAssets(): void {
    // Clear offscreen canvases, pre-rendered assets
  }
  
  private forceGarbageCollection(): void {
    // Force garbage collection by clearing arrays
    // Note: May require --expose-gc flag in Node.js/V8
    if (globalThis.gc) {
      globalThis.gc();
    }
  }
}
```

### Memory Cleanup Strategy

- **Trigger cleanup**: When memory exceeds 100MB
- **Cleanup hierarchy**: 
  1. Effects (particle systems, temporary visuals)
  2. Animations (animation caches, temporary transforms)
  3. Cached assets (offscreen canvases, pre-rendered assets)
- **Force garbage collection**: Clear arrays and trigger GC after cleanup
- **Monitor effectiveness**: Track memory before/after cleanup and escalate if cleanup is ineffective

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Frame-rate independent movement

*For any* valid game session, the ghost's vertical position update using delta time should produce identical results regardless of frame rate, such that `y_new = y_old + velocity * delta_time` where delta_time is capped at 0.1 seconds.

**Validates: Requirements 13.3, 13.4**

### Property 2: Pipe spacing consistency

*For any* pair of consecutive pipes, the horizontal distance between their right edges should equal the game's pipe generation interval multiplied by the current pipe speed, ensuring consistent gameplay difficulty.

**Validates: Requirements 2.1**

### Property 3: Score accuracy

*For any* valid game session where the ghost passes N pipes, the final score should equal N, and each pipe should only be counted once even if the ghost passes through it multiple times.

**Validates: Requirements 4.1, 4.2**

### Property 4: High score preservation

*For any* game session that ends with a final score exceeding the stored high score, the Persistence Manager should save the new high score such that subsequent game loads retrieve this updated value.

**Validates: Requirements 9.1, 9.2, 9.3**

### Property 5: State transition validity

*For any* valid game state transition, the Game Manager should properly clean up the previous state's resources and initialize the next state's required components, ensuring no state-specific data leaks between transitions.

**Validates: Requirements 5.1, 5.2, 5.3, 5.4**

### Property 6: Collision detection completeness

*For any* frame where the ghost's bounding box intersects with a pipe or screen boundary, the Collision Detector should reliably trigger the Game Over state without fail, and the Failsafe Mechanism should ensure state transition occurs even if primary detection fails.

**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**

### Property 7: Difficulty progression monotonicity

*For any* game session that reaches score N, all difficulty parameters (pipe speed, gap size, gravity) should maintain their relationship such that `pipe_speed_N >= pipe_speed_M` for all `N > M`, with the maximum pipe speed never exceeding 300 pixels per second.

**Validates: Requirements 11.4**

### Property 8: Reset state consistency

*For any* game session that transitions from Game Over to Start state, the Reset Manager should ensure the Ghost is positioned at the starting position, all active pipes are removed, the score is reset to 0, and difficulty parameters are restored to initial values.

**Validates: Requirements 13.1, 13.2, 13.3, 13.4, 13.5**

## Error Handling

### Critical Errors

1. **Collision Detection Failure**
   - If collision detection fails to execute, the Failsafe Mechanism logs the error and triggers Game Over state
   - Error is logged with frame timestamp and ghost/pipes positions

2. **Audio Initialization Failure**
   - If audio context fails to initialize, the Audio Manager continues game operation without audio feedback
   - Error is logged but doesn't prevent game execution

3. **High Score Retrieval Failure**
   - If stored high score is unavailable, the Persistence Manager displays placeholder values (0 for score, N/A for high score)
   - Error is logged but game continues with default values

4. **Frame Update Failure**
   - If a frame update fails, the Game Loop skips the Render Phase and continues to the next frame
   - Error is logged with frame details for debugging

### Input Handling Errors

1. **Invalid Input State**
   - Input Handler only processes inputs when in Playing state
   - Inputs in other states are ignored and logged

2. **Touch Event Errors**
   - If touch events fail to register, the Input Handler logs the error but continues operation
   - Falls back to keyboard/mouse inputs if available

## Testing Strategy

### Unit Tests

Unit tests should verify specific examples, edge cases, and error conditions:

1. **Physics Engine Tests**
   - Ghost position update with delta time
   - Gravity application over time
   - Flap velocity application
   - Boundary collision detection

2. **Collision Detector Tests**
   - Ghost-pipe intersection detection
   - Ghost-screen boundary detection
   - Edge cases (ghost exactly touching pipe edge)

3. **Score Manager Tests**
   - Score increment on pipe passage
   - Score preservation on game over
   - Score reset on game start

4. **Difficulty Manager Tests**
   - Speed increase at score milestones
   - Gap size reduction at score milestones
   - Maximum speed cap enforcement

5. **Persistence Manager Tests**
   - High score storage and retrieval
   - High score comparison logic
   - Placeholder value handling

6. **Reset Manager Tests**
   - Ghost position reset
   - Pipe cleanup
   - Score and difficulty reset

### Property-Based Tests

Property-based tests should verify universal properties across all inputs:

1. **Property 1: Frame-rate independent movement**
   - Test that position updates are consistent across varying delta times
   - Generate delta times from 0.001 to 0.1 seconds
   - Verify same result for scaled velocities

2. **Property 2: Score accuracy**
   - Generate random pipe passing sequences
   - Verify score equals number of unique pipes passed
   - Test edge cases (ghost passes same pipe multiple times)

3. **Property 3: High score preservation**
   - Generate random final scores
   - Verify high score is updated when higher
   - Verify persistence across reload cycles

4. **Property 4: Difficulty progression**
   - Generate score progression sequences
   - Verify difficulty parameters increase monotonically
   - Verify maximum speed cap is never exceeded

5. **Property 5: Collision completeness**
   - Generate ghost and pipe position combinations
   - Verify all intersections trigger Game Over
   - Test edge cases (ghost barely touching pipes)

6. **Property 6: Reset consistency**
   - Generate random game states
   - Verify reset returns to initial conditions
   - Test edge cases (many pipes, high scores)

7. **Property 7: Input handling**
   - Generate input sequences for different states
   - Verify inputs only processed in Playing state
   - Test input priority when multiple inputs occur

### Integration Tests

Integration tests should verify end-to-end flows:

1. **Full Game Session**
   - Start to Game Over sequence
   - Verify all components coordinate correctly
   - Test score progression and difficulty

2. **State Transitions**
   - Start → Playing → Game Over → Start
   - Verify resource cleanup between transitions
   - Test multiple consecutive transitions

3. **Window Resize**
   - Generate resize events during gameplay
   - Verify elements reposition correctly
   - Test maximum dimension enforcement

4. **Persistence Flow**
   - Load game → Play → Game Over → Reload
   - Verify high score persistence
   - Test edge cases (no stored score, corrupted data)

### Performance Tests

1. **Frame Rate Stability**
   - Run game for extended periods
   - Verify 60 FPS on modern devices
   - Test frame rate drop handling

2. **Memory Usage**
   - Run game with memory-intensive operations
   - Verify memory cleanup triggers at 100MB
   - Test memory cleanup effectiveness

3. **Resource Management**
   - Generate many pipes and clouds
   - Verify recycling works correctly
   - Test long session memory stability

### Property Test Configuration

Each property-based test should:
- Run minimum 100 iterations
- Reference the corresponding design property
- Tag with **Feature: flappy-kiro, Property N: [property_text]**
- Include edge case generators for boundary conditions

### Test Files Structure

```
tests/
├── physics/
│   ├── position_update.test.ts
│   ├── gravity_application.test.ts
│   └── flap_mechanics.test.ts
├── collision/
│   ├── pipe_detection.test.ts
│   ├── boundary_detection.test.ts
│   └── edge_cases.test.ts
├── scoring/
│   ├── score_increment.test.ts
│   ├── score_preservation.test.ts
│   └── score_reset.test.ts
├── difficulty/
│   ├── progression.test.ts
│   ├── cap_enforcement.test.ts
│   └── milestone_triggers.test.ts
├── persistence/
│   ├── storage.test.ts
│   ├── retrieval.test.ts
│   └── fallback.test.ts
├── reset/
│   ├── state_reset.test.ts
│   └── resource_cleanup.test.ts
├── integration/
│   ├── full_game.test.ts
│   ├── state_transitions.test.ts
│   └── window_resize.test.ts
└── property/
    ├── frame_rate_independence.test.ts
    ├── score_accuracy.test.ts
    ├── high_score_preservation.test.ts
    ├── state_transition_validity.test.ts
    ├── collision_completeness.test.ts
    ├── difficulty_monotonicity.test.ts
    └── reset_consistency.test.ts
```

## Implementation Notes

### Key Design Decisions

1. **Delta Time for Movement**: All movement uses delta time for frame-rate independence, capped at 0.1 seconds to prevent physics anomalies on lag spikes.

2. **Pipe Generation Strategy**: Pipes are generated when the previous pipe exits the screen, ensuring consistent spacing based on current speed.

3. **Cloud Perspective**: Multiple cloud layers with different speeds create depth illusion. Clouds are recycled rather than destroyed/created for performance.

4. **Difficulty Scaling**: Progressive difficulty parameters scale based on score milestones rather than time, ensuring consistent challenge progression.

5. **Error Handling**: Components gracefully degrade on errors (audio, persistence) without preventing game operation.

6. **Memory Management**: Object pooling or recycling for pipes and clouds to minimize garbage collection during gameplay.

### Performance Considerations

1. **Canvas Rendering**: Use single canvas element with context clearing/redrawing optimized for state changes.

2. **Object Recycling**: Reuse pipe and cloud objects rather than creating/destroying to reduce garbage collection.

3. **Sprite Loading**: Pre-load all assets on game initialization to avoid loading delays during gameplay.

4. **Event Delegation**: Use event delegation for input handling rather than individual event listeners on game elements.

5. **Partial Redraws**: Implement dirty rectangle tracking for rendering optimization.

### Testing Considerations

1. **Property Generators**: Create generators for ghost positions, pipe configurations, cloud patterns, and difficulty parameters.

2. **Mock Dependencies**: Use mocks for audio context, local storage, and canvas rendering in unit tests.

3. **Integration Test Environment**: Set up test environment with controlled frame rates and input sequences.

4. **Performance Baselines**: Establish performance benchmarks for 60 FPS target and 100MB memory limit.

## Technical Stack Recommendations

### Language: TypeScript
- Type safety for game data structures
- Modern JavaScript features
- Good ecosystem for game development

### Canvas API
- Hardware-accelerated 2D rendering
- Good performance for 2D games
- Wide browser support

### Audio
- Web Audio API for sound effects
- Background music via HTML5 Audio
- Graceful degradation on unsupported browsers

### Storage
- localStorage for high score persistence
- Error handling for storage limitations
- Fallback to placeholder values