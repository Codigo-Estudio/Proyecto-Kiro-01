# Physics Steering File

This steering file defines precise collision boundaries, momentum calculations, and smooth animation interpolation for the Ghosty game.

## Table of Contents

- [Precise Collision Boundaries](#precise-collision-boundaries)
- [Momentum Calculations](#momentum-calculations)
- [Smooth Animation Interpolation](#smooth-animation-interpolation)
- [Physics Engine Integration](#physics-engine-integration)

---

## Precise Collision Boundaries

### 1. Pixel-Perfect Collision

```javascript
class PixelPerfectCollision {
  constructor() {
    this.pixelThreshold = 0.5; // 50% overlap threshold
    this.collisionPadding = 0; // Padding for hitbox
  }

  checkPixelCollision(entity1, entity2) {
    // Get bounding boxes
    const box1 = this.getSafeBounds(entity1);
    const box2 = this.getSafeBounds(entity2);

    // Check AABB overlap
    if (!this.aabbOverlaps(box1, box2)) {
      return false;
    }

    // Get overlapping region
    const overlap = this.getOverlapRegion(box1, box2);
    
    if (!overlap) return false;

    // Check pixel overlap
    return this.checkPixelOverlap(entity1, entity2, overlap);
  }

  getSafeBounds(entity) {
    return {
      x: entity.x || 0,
      y: entity.y || 0,
      width: entity.width || 0,
      height: entity.height || 0,
      right: (entity.x || 0) + (entity.width || 0),
      bottom: (entity.y || 0) + (entity.height || 0)
    };
  }

  aabbOverlaps(a, b) {
    return (
      a.x < b.right &&
      a.right > b.x &&
      a.y < b.bottom &&
      a.bottom > b.y
    );
  }

  getOverlapRegion(a, b) {
    const x = Math.max(a.x, b.x);
    const y = Math.max(a.y, b.y);
    const right = Math.min(a.right, b.right);
    const bottom = Math.min(a.bottom, b.bottom);

    if (right <= x || bottom <= y) {
      return null;
    }

    return { x, y, width: right - x, height: bottom - y };
  }

  // Check if entities have pixel-level collision
  checkPixelOverlap(entity1, entity2, overlap) {
    // Check if entities have collision masks
    if (entity1.collisionMask && entity2.collisionMask) {
      return this.checkMaskOverlap(entity1, entity2, overlap);
    }

    // Default: check bounding box with padding
    const padding1 = entity1.collisionPadding || this.collisionPadding;
    const padding2 = entity2.collisionPadding || this.collisionPadding;

    const padded1 = this.padBounds(entity1, padding1);
    const padded2 = this.padBounds(entity2, padding2);

    return this.aabbOverlaps(padded1, padded2);
  }

  padBounds(entity, padding) {
    const bounds = this.getSafeBounds(entity);
    return {
      x: bounds.x - padding,
      y: bounds.y - padding,
      width: bounds.width + padding * 2,
      height: bounds.height + padding * 2,
      right: bounds.right + padding,
      bottom: bounds.bottom + padding
    };
  }

  // Collision mask checking
  checkMaskOverlap(entity1, entity2, overlap) {
    const mask1 = entity1.collisionMask;
    const mask2 = entity2.collisionMask;
    
    const relativeX = Math.floor(overlap.x - entity1.x);
    const relativeY = Math.floor(overlap.y - entity1.y);

    // Sample pixels in overlap region
    let totalPixels = 0;
    let overlappingPixels = 0;

    for (let y = 0; y < overlap.height; y += 2) {
      for (let x = 0; x < overlap.width; x += 2) {
        totalPixels++;

        const pixel1 = this.getMaskPixel(mask1, relativeX + x, relativeY + y);
        const pixel2 = this.getMaskPixel(mask2, 
          Math.floor(overlap.x - entity2.x + x),
          Math.floor(overlap.y - entity2.y + y)
        );

        if (pixel1 && pixel2) {
          overlappingPixels++;
          if (overlappingPixels / totalPixels > this.pixelThreshold) {
            return true;
          }
        }
      }
    }

    return overlappingPixels / totalPixels > this.pixelThreshold;
  }

  getMaskPixel(mask, x, y) {
    if (!mask || !mask.data) return false;
    
    if (x < 0 || y < 0 || x >= mask.width || y >= mask.height) {
      return false;
    }

    const index = (y * mask.width + x) * 4;
    return mask.data[index + 3] > 0; // Alpha channel
  }
}
```

### 2. Circle-Based Collision

```javascript
class CircleCollision {
  constructor() {
    this.radiusPadding = 0;
  }

  checkCollision(entity1, entity2) {
    const circle1 = this.getCircle(entity1);
    const circle2 = this.getCircle(entity2);

    if (!circle1 || !circle2) return false;

    const dx = circle1.x - circle2.x;
    const dy = circle1.y - circle2.y;
    const distance = Math.sqrt(dx * dx + dy * dy);

    return distance < circle1.radius + circle2.radius;
  }

  getCircle(entity) {
    // Check for explicit circle property
    if (entity.circle) {
      return {
        x: entity.circle.x || (entity.x || 0) + (entity.width || 0) / 2,
        y: entity.circle.y || (entity.y || 0) + (entity.height || 0) / 2,
        radius: entity.circle.radius || (entity.width || 0) / 2
      };
    }

    // Fall back to bounding circle from bounds
    const bounds = this.getBounds(entity);
    return {
      x: bounds.x + bounds.width / 2,
      y: bounds.y + bounds.height / 2,
      radius: Math.min(bounds.width, bounds.height) / 2 + this.radiusPadding
    };
  }

  getBounds(entity) {
    return {
      x: entity.x || 0,
      y: entity.y || 0,
      width: entity.width || 0,
      height: entity.height || 0,
      right: (entity.x || 0) + (entity.width || 0),
      bottom: (entity.y || 0) + (entity.height || 0)
    };
  }

  // Collision response with circle-circle resolution
  resolveCollision(entity1, entity2) {
    const circle1 = this.getCircle(entity1);
    const circle2 = this.getCircle(entity2);

    const dx = circle2.x - circle1.x;
    const dy = circle2.y - circle1.y;
    const distance = Math.sqrt(dx * dx + dy * dy);

    if (distance === 0) return { normal: { x: 1, y: 0 }, depth: 0 };

    const overlap = (circle1.radius + circle2.radius) - distance;

    if (overlap <= 0) return null;

    const normal = { x: dx / distance, y: dy / distance };

    return {
      normal,
      depth: overlap,
      entity1,
      entity2
    };
  }
}
```

### 3. Polygon Collision (SAT)

```javascript
class PolygonCollision {
  constructor() {
    this.axes = [];
  }

  checkCollision(poly1, poly2) {
    // Get vertices
    const vertices1 = this.getVertices(poly1);
    const vertices2 = this.getVertices(poly2);

    // Check all edges as potential separating axes
    for (let i = 0; i < vertices1.length; i++) {
      const edge = this.getEdge(vertices1, i);
      const axis = this.getPerpendicular(edge);

      if (!this.overlapOnAxis(vertices1, vertices2, axis)) {
        return false; // Separating axis found
      }
    }

    for (let i = 0; i < vertices2.length; i++) {
      const edge = this.getEdge(vertices2, i);
      const axis = this.getPerpendicular(edge);

      if (!this.overlapOnAxis(vertices1, vertices2, axis)) {
        return false; // Separating axis found
      }
    }

    return true; // No separating axis found
  }

  getVertices(entity) {
    if (entity.vertices) {
      return entity.vertices;
    }

    // Default: create rectangle from bounds
    const bounds = this.getBounds(entity);
    return [
      { x: bounds.x, y: bounds.y },
      { x: bounds.right, y: bounds.y },
      { x: bounds.right, y: bounds.bottom },
      { x: bounds.x, y: bounds.bottom }
    ];
  }

  getEdge(vertices, index) {
    const current = vertices[index];
    const next = vertices[(index + 1) % vertices.length];
    return {
      x: next.x - current.x,
      y: next.y - current.y
    };
  }

  getPerpendicular(edge) {
    const length = Math.sqrt(edge.x * edge.x + edge.y * edge.y);
    return {
      x: -edge.y / length,
      y: edge.x / length
    };
  }

  overlapOnAxis(vertices1, vertices2, axis) {
    const min1 = this.project(vertices1, axis);
    const min2 = this.project(vertices2, axis);

    return min1.max >= min2.min && min2.max >= min1.min;
  }

  project(vertices, axis) {
    let min = Infinity;
    let max = -Infinity;

    for (const vertex of vertices) {
      const dot = vertex.x * axis.x + vertex.y * axis.y;
      min = Math.min(min, dot);
      max = Math.max(max, dot);
    }

    return { min, max };
  }

  getBounds(entity) {
    return {
      x: entity.x || 0,
      y: entity.y || 0,
      width: entity.width || 0,
      height: entity.height || 0,
      right: (entity.x || 0) + (entity.width || 0),
      bottom: (entity.y || 0) + (entity.height || 0)
    };
  }
}
```

### 4. Grid-Based Collision

```javascript
class GridCollision {
  constructor(cellSize = 32) {
    this.cellSize = cellSize;
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

  addEntity(entity) {
    const bounds = this.getBounds(entity);
    const cells = this.getOccupiedCells(bounds);
    
    cells.forEach(cellKey => {
      if (!this.grid.has(cellKey)) {
        this.grid.set(cellKey, []);
      }
      this.grid.get(cellKey).push(entity);
    });
  }

  getBounds(entity) {
    return {
      x: entity.x || 0,
      y: entity.y || 0,
      width: entity.width || 0,
      height: entity.height || 0,
      right: (entity.x || 0) + (entity.width || 0),
      bottom: (entity.y || 0) + (entity.height || 0)
    };
  }

  getOccupiedCells(bounds) {
    const cells = new Set();
    
    const leftCol = Math.floor(bounds.x / this.cellSize);
    const rightCol = Math.floor(bounds.right / this.cellSize);
    const topRow = Math.floor(bounds.y / this.cellSize);
    const bottomRow = Math.floor(bounds.bottom / this.cellSize);
    
    for (let col = leftCol; col <= rightCol; col++) {
      for (let row = topRow; row <= bottomRow; row++) {
        cells.add(this.getKey(col * this.cellSize, row * this.cellSize));
      }
    }
    
    return Array.from(cells);
  }

  checkCollisions() {
    const collisions = [];
    const checked = new Set();

    for (const [key, entities] of this.grid) {
      for (let i = 0; i < entities.length; i++) {
        for (let j = i + 1; j < entities.length; j++) {
          const a = entities[i];
          const b = entities[j];
          
          const pairKey = `${a.id || a}_{b.id || b}`;
          if (checked.has(pairKey)) continue;
          
          checked.add(pairKey);
          
          if (this.aabbOverlaps(this.getBounds(a), this.getBounds(b))) {
            collisions.push({ a, b });
          }
        }
      }
    }

    return collisions;
  }

  aabbOverlaps(a, b) {
    return (
      a.x < b.right &&
      a.right > b.x &&
      a.y < b.bottom &&
      a.bottom > b.y
    );
  }

  getNearbyEntities(x, y, radius) {
    const nearby = new Set();
    const bounds = {
      x: x - radius,
      y: y - radius,
      width: radius * 2,
      height: radius * 2,
      right: x + radius,
      bottom: y + radius
    };
    
    const cells = this.getOccupiedCells(bounds);
    
    cells.forEach(cellKey => {
      const cell = this.grid.get(cellKey);
      if (cell) {
        cell.forEach(entity => nearby.add(entity));
      }
    });
    
    return Array.from(nearby);
  }
}
```

---

## Momentum Calculations

### 1. Velocity and Acceleration

```javascript
class MomentumSystem {
  constructor() {
    this.friction = 0.98;
    this.maxSpeed = 500;
  }

  applyForce(entity, force, deltaTime) {
    if (!entity.physics) return;

    // F = ma, so a = F/m
    const mass = entity.mass || 1;
    const acceleration = {
      x: force.x / mass,
      y: force.y / mass
    };

    // Update velocity
    entity.physics.velocity.x += acceleration.x * deltaTime;
    entity.physics.velocity.y += acceleration.y * deltaTime;
  }

  updateVelocity(entity, deltaTime) {
    if (!entity.physics) return;

    // Apply friction
    entity.physics.velocity.x *= this.friction;
    entity.physics.velocity.y *= this.friction;

    // Clamp to max speed
    const speed = Math.sqrt(
      entity.physics.velocity.x ** 2 + 
      entity.physics.velocity.y ** 2
    );

    if (speed > this.maxSpeed) {
      const ratio = this.maxSpeed / speed;
      entity.physics.velocity.x *= ratio;
      entity.physics.velocity.y *= ratio;
    }
  }

  updatePosition(entity, deltaTime) {
    if (!entity.physics) return;

    entity.x += entity.physics.velocity.x * deltaTime;
    entity.y += entity.physics.velocity.y * deltaTime;
  }

  getMomentum(entity) {
    if (!entity.physics) return 0;
    
    const mass = entity.mass || 1;
    const speed = Math.sqrt(
      entity.physics.velocity.x ** 2 + 
      entity.physics.velocity.y ** 2
    );

    return mass * speed;
  }

  getKineticEnergy(entity) {
    if (!entity.physics) return 0;
    
    const mass = entity.mass || 1;
    const speed = Math.sqrt(
      entity.physics.velocity.x ** 2 + 
      entity.physics.velocity.y ** 2
    );

    return 0.5 * mass * speed ** 2;
  }
}
```

### 2. Impulse and Collision Response

```javascript
class ImpulseSystem {
  constructor() {
    this.restitution = 0.7; // Bounciness
    this.friction = 0.3;
  }

  applyImpulse(entity, impulse) {
    if (!entity.physics) return;

    const mass = entity.mass || 1;
    entity.physics.velocity.x += impulse.x / mass;
    entity.physics.velocity.y += impulse.y / mass;
  }

  resolveCollision(a, b, collisionInfo) {
    const { normal, depth } = collisionInfo;

    // Calculate relative velocity
    const relativeVelocity = {
      x: a.physics.velocity.x - b.physics.velocity.x,
      y: a.physics.velocity.y - b.physics.velocity.y
    };

    // Velocity along normal
    const velocityAlongNormal = 
      relativeVelocity.x * normal.x + 
      relativeVelocity.y * normal.y;

    // Do not resolve if velocities are separating
    if (velocityAlongNormal > 0) return;

    // Calculate impulse scalar
    let impulseScalar = -(1 + this.restitution) * velocityAlongNormal;
    impulseScalar /= 1 / (a.mass || 1) + 1 / (b.mass || 1);

    // Apply impulse
    const impulse = {
      x: impulseScalar * normal.x,
      y: impulseScalar * normal.y
    };

    this.applyImpulse(a, impulse);
    this.applyImpulse(b, { x: -impulse.x, y: -impulse.y });

    // Positional correction (prevent sinking)
    const percent = 0.2; // Penetration percentage to correct
    const slop = 0.01; // Penetration allowance
    const correction = {
      x: (depth - slop) / (1 / (a.mass || 1) + 1 / (b.mass || 1)) * percent * normal.x,
      y: (depth - slop) / (1 / (a.mass || 1) + 1 / (b.mass || 1)) * percent * normal.y
    };

    a.x += correction.x / (a.mass || 1);
    a.y += correction.y / (a.mass || 1);
    b.x -= correction.x / (b.mass || 1);
    b.y -= correction.y / (b.mass || 1);
  }

  applyFriction(a, b) {
    const relativeVelocity = {
      x: a.physics.velocity.x - b.physics.velocity.x,
      y: a.physics.velocity.y - b.physics.velocity.y
    };

    // Project velocity to tangent plane
    const normal = { x: 0, y: 1 }; // Vertical surface
    const dot = relativeVelocity.x * normal.x + relativeVelocity.y * normal.y;
    const tangent = {
      x: relativeVelocity.x - dot * normal.x,
      y: relativeVelocity.y - dot * normal.y
    };

    // Apply friction to tangent
    const frictionForce = {
      x: -tangent.x * this.friction,
      y: -tangent.y * this.friction
    };

    this.applyImpulse(a, frictionForce);
    this.applyImpulse(b, { x: -frictionForce.x, y: -frictionForce.y });
  }
}
```

### 3. Spring and Damping

```javascript
class SpringSystem {
  constructor() {
    this.springConstant = 100;
    this.damping = 0.9;
    this.restLength = 0;
  }

  applySpringForce(entity, anchorX, anchorY) {
    if (!entity.physics) return;

    const currentX = entity.x + (entity.width || 0) / 2;
    const currentY = entity.y + (entity.height || 0) / 2;

    const dx = anchorX - currentX;
    const dy = anchorY - currentY;
    const distance = Math.sqrt(dx * dx + dy * dy);

    if (distance === 0) return;

    // Hooke's Law: F = -k * x
    const forceMagnitude = (distance - this.restLength) * this.springConstant;
    const force = {
      x: (dx / distance) * forceMagnitude,
      y: (dy / distance) * forceMagnitude
    };

    entity.physics.velocity.x += force.x * deltaTime;
    entity.physics.velocity.y += force.y * deltaTime;
  }

  applyDamping(entity) {
    if (!entity.physics) return;

    entity.physics.velocity.x *= this.damping;
    entity.physics.velocity.y *= this.damping;
  }

  // Simple spring constraint
  constrainToSpring(entity, anchorX, anchorY, maxDistance) {
    const currentX = entity.x + (entity.width || 0) / 2;
    const currentY = entity.y + (entity.height || 0) / 2;

    const dx = currentX - anchorX;
    const dy = currentY - anchorY;
    const distance = Math.sqrt(dx * dx + dy * dy);

    if (distance > maxDistance) {
      const ratio = maxDistance / distance;
      entity.x = anchorX + dx * ratio - (entity.width || 0) / 2;
      entity.y = anchorY + dy * ratio - (entity.height || 0) / 2;
      
      // Dampen velocity
      entity.physics.velocity.x *= this.damping;
      entity.physics.velocity.y *= this.damping;
    }
  }
}
```

### 4. Momentum Conservation

```javascript
class MomentumConservation {
  constructor() {
    this.energyLoss = 0.1; // 10% energy loss per collision
  }

  resolveElasticCollision(a, b) {
    const massA = a.mass || 1;
    const massB = b.mass || 1;

    const vA = {
      x: a.physics.velocity.x,
      y: a.physics.velocity.y
    };
    
    const vB = {
      x: b.physics.velocity.x,
      y: b.physics.velocity.y
    };

    // Normal vector
    const nx = b.x - a.x;
    const ny = b.y - a.y;
    const length = Math.sqrt(nx * nx + ny * ny);

    if (length === 0) return;

    const normal = { x: nx / length, y: ny / length };

    // Relative velocity
    const rv = {
      x: vA.x - vB.x,
      y: vA.y - vB.y
    };

    // Velocity along normal
    const velAlongNormal = rv.x * normal.x + rv.y * normal.y;

    // Do not resolve if velocities are separating
    if (velAlongNormal > 0) return;

    // Impulse scalar
    let j = -(1 + (1 - this.energyLoss)) * velAlongNormal;
    j /= 1 / massA + 1 / massB;

    // Apply impulse
    const impulse = {
      x: j * normal.x,
      y: j * normal.y
    };

    a.physics.velocity.x += impulse.x / massA;
    a.physics.velocity.y += impulse.y / massA;
    b.physics.velocity.x -= impulse.x / massB;
    b.physics.velocity.y -= impulse.y / massB;
  }

  // Perfectly inelastic collision (objects stick together)
  resolveInelasticCollision(a, b) {
    const massA = a.mass || 1;
    const massB = b.mass || 1;

    const totalMass = massA + massB;
    const totalMomentum = {
      x: massA * a.physics.velocity.x + massB * b.physics.velocity.x,
      y: massA * a.physics.velocity.y + massB * b.physics.velocity.y
    };

    const finalVelocity = {
      x: totalMomentum.x / totalMass,
      y: totalMomentum.y / totalMass
    };

    a.physics.velocity.x = finalVelocity.x;
    a.physics.velocity.y = finalVelocity.y;
    b.physics.velocity.x = finalVelocity.x;
    b.physics.velocity.y = finalVelocity.y;
  }

  // Calculate total system momentum
  getSystemMomentum(entities) {
    return entities.reduce((momentum, entity) => {
      const mass = entity.mass || 1;
      return {
        x: momentum.x + mass * entity.physics.velocity.x,
        y: momentum.y + mass * entity.physics.velocity.y
      };
    }, { x: 0, y: 0 });
  }

  // Calculate total system energy
  getSystemEnergy(entities) {
    return entities.reduce((energy, entity) => {
      const mass = entity.mass || 1;
      const speed = Math.sqrt(
        entity.physics.velocity.x ** 2 + 
        entity.physics.velocity.y ** 2
      );
      return energy + 0.5 * mass * speed ** 2;
    }, 0);
  }
}
```

---

## Smooth Animation Interpolation

### 1. Lerp and Slerp

```javascript
class Interpolation {
  constructor() {
    this.lerpFactor = 0.1; // Linear interpolation factor
    this.slerpFactor = 0.1; // Spherical interpolation factor
  }

  // Linear interpolation
  lerp(start, end, t) {
    return start * (1 - t) + end * t;
  }

  // Smooth step interpolation
  smoothStep(start, end, t) {
    const t2 = t * t;
    const t3 = t2 * t;
    return start + (end - start) * (3 * t2 - 2 * t3);
  }

  // Smoothest step interpolation
  smootherStep(start, end, t) {
    return start + (end - start) * (6 * t ** 5 - 15 * t ** 4 + 10 * t ** 3);
  }

  // Exponential interpolation
  easeIn(start, end, t, power = 2) {
    const tPow = Math.pow(t, power);
    return start + (end - start) * tPow;
  }

  easeOut(start, end, t, power = 2) {
    const tPow = Math.pow(t, power);
    return start + (end - start) * (1 - Math.pow(1 - t, power));
  }

  easeInOut(start, end, t, power = 2) {
    if (t < 0.5) {
      return this.easeIn(start, end, t * 2, power) / 2;
    }
    return this.easeOut(start, end, (t - 0.5) * 2, power) / 2 + start / 2;
  }

  // Spherical linear interpolation for rotations
  slerp(q1, q2, t) {
    const dot = q1.x * q2.x + q1.y * q2.y + q1.z * q2.z + q1.w * q2.w;
    
    if (dot < 0) {
      q1 = { x: -q1.x, y: -q1.y, z: -q1.z, w: -q1.w };
    }
    
    const theta = Math.acos(Math.max(-1, Math.min(1, dot)));
    
    if (theta < 0.001) {
      return this.lerpQuaternion(q1, q2, t);
    }
    
    const sinTheta = Math.sin(theta);
    const factor1 = Math.sin((1 - t) * theta) / sinTheta;
    const factor2 = Math.sin(t * theta) / sinTheta;
    
    return {
      x: q1.x * factor1 + q2.x * factor2,
      y: q1.y * factor1 + q2.y * factor2,
      z: q1.z * factor1 + q2.z * factor2,
      w: q1.w * factor1 + q2.w * factor2
    };
  }

  lerpQuaternion(q1, q2, t) {
    return {
      x: q1.x * (1 - t) + q2.x * t,
      y: q1.y * (1 - t) + q2.y * t,
      z: q1.z * (1 - t) + q2.z * t,
      w: q1.w * (1 - t) + q2.w * t
    };
  }
}
```

### 2. Animation Interpolation

```javascript
class AnimationInterpolation {
  constructor() {
    this.interpolation = new Interpolation();
    this.entities = new Map();
    this.previousPositions = new Map();
  }

  startAnimation(entity, targetX, targetY, duration, easeFn = 'smoothStep') {
    const startTime = performance.now();
    
    this.entities.set(entity, {
      targetX,
      targetY,
      startX: entity.x,
      startY: entity.y,
      startTime,
      duration,
      easeFn,
      completed: false
    });
  }

  updateAnimation(entity, deltaTime) {
    const anim = this.entities.get(entity);
    if (!anim || anim.completed) return;

    const elapsed = (performance.now() - anim.startTime) / 1000;
    const t = Math.min(elapsed / anim.duration, 1);

    // Apply easing function
    let easedT = t;
    switch (anim.easeFn) {
      case 'smoothStep':
        easedT = this.interpolation.smoothStep(0, 1, t);
        break;
      case 'easeIn':
        easedT = this.interpolation.easeIn(0, 1, t);
        break;
      case 'easeOut':
        easedT = this.interpolation.easeOut(0, 1, t);
        break;
      case 'easeInOut':
        easedT = this.interpolation.easeInOut(0, 1, t);
        break;
      default:
        easedT = t;
    }

    // Interpolate position
    entity.x = this.interpolation.lerp(anim.startX, anim.targetX, easedT);
    entity.y = this.interpolation.lerp(anim.startY, anim.targetY, easedT);

    if (t >= 1) {
      anim.completed = true;
    }
  }

  updateRotation(entity, targetRotation, deltaTime, duration = 0.3) {
    const currentRotation = entity.rotation || 0;
    const startRotation = currentRotation;
    
    const startTime = performance.now();
    
    // Store animation data
    const anim = {
      startRotation,
      targetRotation,
      startTime,
      duration,
      completed: false
    };

    this.entities.set(`${entity}_rotation`, anim);
  }

  // Smooth follow animation
  followEntity(follower, target, smoothing = 0.1) {
    const targetX = target.x + (target.width || 0) / 2;
    const targetY = target.y + (target.height || 0) / 2;
    
    const currentX = follower.x + (follower.width || 0) / 2;
    const currentY = follower.y + (follower.height || 0) / 2;
    
    const dx = targetX - currentX;
    const dy = targetY - currentY;
    
    follower.x += dx * smoothing;
    follower.y += dy * smoothing;
  }
}
```

### 3. Camera Smoothing

```javascript
class SmoothCamera {
  constructor(width, height) {
    this.width = width;
    this.height = height;
    this.x = 0;
    this.y = 0;
    this.targetX = 0;
    this.targetY = 0;
    this.smoothing = 0.05;
    this.maxSpeed = 200;
  }

  followTarget(target, padding = 100) {
    // Calculate target position (center on target with padding)
    this.targetX = target.x + (target.width || 0) / 2 - this.width / 2;
    this.targetY = target.y + (target.height || 0) / 2 - this.height / 2;
  }

  update(deltaTime) {
    // Calculate velocity
    const dx = this.targetX - this.x;
    const dy = this.targetY - this.y;
    
    // Apply smoothing
    this.x += dx * this.smoothing;
    this.y += dy * this.smoothing;
    
    // Clamp to max speed
    const speed = Math.sqrt(dx * dx + dy * dy);
    if (speed > this.maxSpeed) {
      const ratio = this.maxSpeed / speed;
      this.x += dx * ratio * deltaTime;
      this.y += dy * ratio * deltaTime;
    }
  }

  // Smooth zoom
  zoomTo(zoomLevel, duration = 1) {
    this.targetZoom = zoomLevel;
    this.zoomStartTime = performance.now();
    this.zoomStart = this.zoom || 1;
    this.zoomDuration = duration;
  }

  updateZoom(deltaTime) {
    if (this.zoomStartTime) {
      const elapsed = (performance.now() - this.zoomStartTime) / 1000;
      const t = Math.min(elapsed / this.zoomDuration, 1);
      
      this.zoom = this.interpolation.lerp(this.zoomStart, this.targetZoom, t);
      
      if (t >= 1) {
        this.zoomStartTime = null;
      }
    }
  }

  // Screen shake
  shake(intensity = 10, duration = 0.5) {
    this.shakeIntensity = intensity;
    this.shakeStartTime = performance.now();
    this.shakeDuration = duration;
  }

  updateShake(deltaTime) {
    if (this.shakeStartTime) {
      const elapsed = (performance.now() - this.shakeStartTime) / 1000;
      
      if (elapsed < this.shakeDuration) {
        this.x += (Math.random() - 0.5) * this.shakeIntensity;
        this.y += (Math.random() - 0.5) * this.shakeIntensity;
      } else {
        this.shakeStartTime = null;
        this.shakeIntensity = 0;
      }
    }
  }
}
```
