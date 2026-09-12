# Canvas Optimization and Visual Feedback Steering File

This steering file defines Canvas drawing optimization techniques, sprite atlas usage, and visual feedback patterns for high-performance game rendering.

## Table of Contents

- [Canvas Drawing Optimization](#canvas-drawing-optimization)
- [Sprite Atlas Usage](#sprite-atlas-usage)
- [Visual Feedback Patterns](#visual-feedback-patterns)
- [Rendering Systems Implementation](#rendering-systems-implementation)

---

## Canvas Drawing Optimization

### 1. Batch Rendering System

```javascript
class BatchRenderer {
  constructor(maxBatchSize = 100) {
    this.maxBatchSize = maxBatchSize;
    this.batches = new Map();
    this.currentBatch = null;
    this.renderCount = 0;
    this.clear();
  }

  clear() {
    this.batches.clear();
    this.currentBatch = null;
    this.renderCount = 0;
  }

  begin(ctx, textureKey) {
    if (!this.batches.has(textureKey)) {
      this.batches.set(textureKey, []);
    }
    this.currentBatch = this.batches.get(textureKey);
  }

  draw(x, y, width, height, rotation = 0, flipX = false, flipY = false) {
    if (!this.currentBatch) return;

    if (this.currentBatch.length >= this.maxBatchSize) {
      this.flush();
      this.currentBatch = [];
    }

    this.currentBatch.push({
      x, y, width, height, rotation, flipX, flipY
    });
    this.renderCount++;
  }

  flush() {
    const textureKey = this.getTextureKey();
    const batch = this.batches.get(textureKey);
    
    if (batch && batch.length > 0) {
      this.renderBatch(textureKey, batch);
      batch.length = 0;
    }
  }

  getTextureKey() {
    return this._currentTextureKey;
  }

  setTextureKey(key) {
    this._currentTextureKey = key;
  }

  renderBatch(textureKey, batch) {
    // This would be implemented with sprite batch rendering
    // In a real implementation, this would use WebGL or 
    // optimized Canvas drawImage calls
  }
}

// Optimized Canvas renderer
class OptimizedCanvasRenderer {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d', { alpha: false });
    this.batcher = new BatchRenderer();
    this.dirtyRects = [];
    this.optimizedDraw = true;
  }

  // Single pass rendering with clipping
  beginFrame() {
    this.ctx.save();
    
    // Use save/restore efficiently - avoid nested calls
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    this.dirtyRects = [];
  }

  endFrame() {
    this.ctx.restore();
  }

  // Clipped rendering
  drawClipped(image, srcX, srcY, srcW, srcH, destX, destY, destW, destH) {
    this.ctx.save();
    
    // Create clipping region
    this.ctx.beginPath();
    this.ctx.rect(destX, destY, destW, destH);
    this.ctx.clip();
    
    // Draw image
    this.ctx.drawImage(
      image,
      srcX, srcY, srcW, srcH,
      destX, destY, destW, destH
    );
    
    this.ctx.restore();
  }

  // Backface culling for rotated sprites
  drawRotated(image, x, y, width, height, rotation, opacity = 1) {
    const cos = Math.cos(rotation);
    const sin = Math.sin(rotation);

    // Calculate bounding box
    const hw = width / 2;
    const hh = height / 2;
    
    const points = [
      { x: -hw * cos - hh * sin, y: -hw * sin + hh * cos },
      { x: hw * cos - hh * sin, y: hw * sin + hh * cos },
      { x: hw * cos + hh * sin, y: hw * sin - hh * cos },
      { x: -hw * cos + hh * sin, y: -hw * sin - hh * cos }
    ];

    // Calculate bounding box dimensions
    let minX = Infinity, maxX = -Infinity;
    let minY = Infinity, maxY = -Infinity;

    points.forEach(p => {
      minX = Math.min(minX, p.x);
      maxX = Math.max(maxX, p.x);
      minY = Math.min(minY, p.y);
      maxY = Math.max(maxY, p.y);
    });

    const bw = maxX - minX;
    const bh = maxY - minY;

    // Cull if completely off-screen
    if (maxX < 0 || minX > this.canvas.width ||
        maxY < 0 || minY > this.canvas.height) {
      return;
    }

    // Draw if visible
    this.ctx.save();
    this.ctx.translate(x, y);
    this.ctx.rotate(rotation);
    this.ctx.globalAlpha = opacity;
    this.ctx.drawImage(
      image,
      -hw, -hh, width, height
    );
    this.ctx.restore();
  }

  // Sprite batching with state caching
  drawSpriteBatch(sprites) {
    // Sort by texture to minimize texture switches
    sprites.sort((a, b) => {
      if (a.texture === b.texture) return 0;
      return a.texture < b.texture ? -1 : 1;
    });

    let currentTexture = null;

    this.ctx.save();

    sprites.forEach(sprite => {
      if (sprite.texture !== currentTexture) {
        this.ctx.restore();
        this.ctx.save();
        currentTexture = sprite.texture;
        // Apply texture-specific settings
      }

      this.ctx.translate(sprite.x, sprite.y);
      this.ctx.rotate(sprite.rotation);
      
      if (currentTexture) {
        this.ctx.drawImage(currentTexture, -sprite.width/2, -sprite.height/2);
      }
      
      this.ctx.setTransform(1, 0, 0, 1, 0, 0);
    });

    this.ctx.restore();
  }
}
```

### 2. Viewport Culling

```javascript
class ViewportCuller {
  constructor(viewport) {
    this.viewport = viewport;
    this.visibleObjects = [];
    this.culledObjects = [];
  }

  updateViewport(x, y, width, height) {
    this.viewport = { x, y, width, height };
  }

  isVisible(object) {
    if (!object.bounds) return true;
    
    const bounds = object.bounds;
    return (
      bounds.x < this.viewport.x + this.viewport.width &&
      bounds.x + bounds.width > this.viewport.x &&
      bounds.y < this.viewport.y + this.viewport.height &&
      bounds.y + bounds.height > this.viewport.y
    );
  }

  cull(objects) {
    this.visibleObjects = [];
    this.culledObjects = [];

    objects.forEach(object => {
      if (this.isVisible(object)) {
        this.visibleObjects.push(object);
      } else {
        this.culledObjects.push(object);
      }
    });

    return this.visibleObjects;
  }

  cullAndRender(objects, renderer) {
    const visible = this.cull(objects);
    
    visible.forEach(object => {
      renderer.render(object);
    });
  }

  // Camera-based culling
  cullWithCamera(camera, objects) {
    const viewport = {
      x: camera.x,
      y: camera.y,
      width: camera.width,
      height: camera.height
    };
    
    this.updateViewport(viewport.x, viewport.y, viewport.width, viewport.height);
    return this.cull(objects);
  }

  // Distance-based culling (for performance)
  cullByDistance(objects, center, maxDistance) {
    return objects.filter(object => {
      const dx = object.x - center.x;
      const dy = object.y - center.y;
      const distance = Math.sqrt(dx * dx + dy * dy);
      return distance <= maxDistance;
    });
  }
}
```

### 3. Render Layer Manager

```javascript
class RenderLayerManager {
  constructor() {
    this.layers = new Map();
    this.layerOrder = [];
    this.defaultLayers = [
      'background', 'parallax', 'world', 'entities', 'ui', 'overlay'
    ];
  }

  addLayer(name, zIndex = 0) {
    this.layers.set(name, {
      name,
      zIndex,
      objects: [],
      visible: true
    });
    this.layerOrder.push(name);
    this.layerOrder.sort((a, b) => {
      return this.layers.get(a).zIndex - this.layers.get(b).zIndex;
    });
  }

  addObject(layerName, object) {
    const layer = this.layers.get(layerName);
    if (layer) {
      layer.objects.push(object);
    }
  }

  removeObject(layerName, object) {
    const layer = this.layers.get(layerName);
    if (layer) {
      const index = layer.objects.indexOf(object);
      if (index > -1) {
        layer.objects.splice(index, 1);
      }
    }
  }

  setLayerVisible(name, visible) {
    const layer = this.layers.get(name);
    if (layer) {
      layer.visible = visible;
    }
  }

  render(ctx, renderer) {
    this.layerOrder.forEach(layerName => {
      const layer = this.layers.get(layerName);
      if (layer.visible && layer.objects.length > 0) {
        layer.objects.forEach(object => {
          renderer.render(object);
        });
      }
    });
  }

  // Get objects in a layer
  getObjects(layerName) {
    const layer = this.layers.get(layerName);
    return layer ? layer.objects : [];
  }

  // Move object between layers
  moveObject(object, fromLayer, toLayer) {
    this.removeObject(fromLayer, object);
    this.addObject(toLayer, object);
  }

  // Create default layer setup
  createDefaultLayers() {
    this.addLayer('background', -100);
    this.addLayer('parallax', -50);
    this.addLayer('world', 0);
    this.addLayer('entities', 100);
    this.addLayer('ui', 200);
    this.addLayer('overlay', 300);
  }

  // Render to texture (for post-processing)
  renderToTexture(renderer, texture) {
    const textureCanvas = document.createElement('canvas');
    textureCanvas.width = texture.width;
    textureCanvas.height = texture.height;
    const textureCtx = textureCanvas.getContext('2d');

    this.render(textureCtx, renderer);

    return textureCanvas;
  }
}
```

### 4. Object Pool for Rendering

```javascript
class RenderingPool {
  constructor(createFn, resetFn, initialSize = 50) {
    this.createFn = createFn;
    this.resetFn = resetFn;
    this.pool = [];
    this.active = new Set();
    
    for (let i = 0; i < initialSize; i++) {
      this.pool.push(this.createFn());
    }
  }

  acquire() {
    if (this.pool.length > 0) {
      const obj = this.pool.pop();
      this.active.add(obj);
      return obj;
    }
    const obj = this.createFn();
    this.active.add(obj);
    return obj;
  }

  release(obj) {
    if (this.active.has(obj)) {
      this.resetFn(obj);
      this.active.delete(obj);
      this.pool.push(obj);
    }
  }

  releaseAll() {
    this.active.forEach(obj => {
      this.resetFn(obj);
      this.pool.push(obj);
    });
    this.active.clear();
  }

  size() {
    return this.pool.length;
  }

  activeSize() {
    return this.active.size;
  }

  // Object pool with ID tracking
  static withId(createFn, resetFn, initialSize = 50) {
    let nextId = 0;
    
    return new RenderingPool(
      () => {
        const obj = createFn();
        obj.id = nextId++;
        return obj;
      },
      obj => {
        resetFn(obj);
        obj.id = -1;
      },
      initialSize
    );
  }
}

// Usage example
const particlePool = new RenderingPool(
  () => ({ x: 0, y: 0, vx: 0, vy: 0, life: 0, active: false }),
  particle => {
    particle.x = 0;
    particle.y = 0;
    particle.vx = 0;
    particle.vy = 0;
    particle.life = 0;
    particle.active = false;
  },
  100
);

// In game loop
const particle = particlePool.acquire();
particle.x = 100;
particle.y = 100;
particle.vx = 50;
particle.vy = -100;
particle.life = 1.0;
particle.active = true;

// When particle expires
particlePool.release(particle);
```

---

## Sprite Atlas Usage

### 1. Sprite Atlas Manager

```javascript
class SpriteAtlasManager {
  constructor() {
    this.atlases = new Map();
    this.atlasSize = 2048; // Default maximum texture size
  }

  async createAtlas(key, images) {
    const atlasCanvas = document.createElement('canvas');
    let width = this.atlasSize;
    let height = this.atlasSize;
    
    // Simple packing algorithm
    const slots = [];
    let currentX = 0;
    let currentY = 0;
    let rowHeight = 0;

    images.forEach((image, index) => {
      const slot = {
        image,
        x: currentX,
        y: currentY,
        width: image.width,
        height: image.height,
        name: `sprite_${index}`
      };

      slots.push(slot);
      
      currentX += image.width;
      rowHeight = Math.max(rowHeight, image.height);

      // Wrap to next row
      if (currentX + image.width > width) {
        currentX = 0;
        currentY += rowHeight;
        rowHeight = 0;
      }
    });

    // Handle overflow - create additional atlas pages
    if (currentY + rowHeight > height) {
      // Would implement multi-page atlas logic here
    }

    atlasCanvas.width = width;
    atlasCanvas.height = currentY + rowHeight;
    const ctx = atlasCanvas.getContext('2d');

    slots.forEach(slot => {
      ctx.drawImage(slot.image, slot.x, slot.y);
    });

    const atlas = {
      canvas: atlasCanvas,
      ctx: ctx,
      slots,
      width,
      height
    };

    this.atlases.set(key, atlas);
    return atlas;
  }

  getAtlas(key) {
    return this.atlases.get(key);
  }

  draw(ctx, atlasKey, slotName, x, y, width, height, flipX = false, flipY = false, rotation = 0) {
    const atlas = this.atlases.get(atlasKey);
    if (!atlas) return;

    const slot = atlas.slots.find(s => s.name === slotName);
    if (!slot) return;

    ctx.save();
    ctx.translate(x + width / 2, y + height / 2);
    
    if (flipX) ctx.scale(-1, 1);
    if (flipY) ctx.scale(1, -1);
    if (rotation) ctx.rotate(rotation);

    ctx.drawImage(
      atlas.canvas,
      slot.x, slot.y, slot.width, slot.height,
      -width / 2, -height / 2, width, height
    );

    ctx.restore();
  }

  // Texture atlas with metadata
  createMetaAtlas(atlasKey, spriteDefinitions) {
    const atlas = this.atlases.get(atlasKey);
    if (!atlas) return;

    // Store metadata for each sprite
    spriteDefinitions.forEach(def => {
      const slot = atlas.slots.find(s => s.name === def.name);
      if (slot) {
        slot.metadata = def.metadata;
        slot.region = def.region;
      }
    });
  }

  // Dynamic atlas updates
  addSpriteToAtlas(atlasKey, image, name) {
    const atlas = this.atlases.get(atlasKey);
    if (!atlas) return null;

    // Find empty slot or add new one
    // (simplified - would need actual packing algorithm in production)
    const slot = {
      name,
      image,
      x: atlas.width / 2,
      y: atlas.height / 2,
      width: image.width,
      height: image.height
    };

    atlas.slots.push(slot);
    
    // Redraw atlas
    const ctx = atlas.ctx;
    ctx.drawImage(image, slot.x, slot.y);

    return slot;
  }
}

// Texture atlas renderer
class AtlasRenderer {
  constructor(atlasManager) {
    this.atlasManager = atlasManager;
    this.currentAtlas = null;
  }

  setAtlas(atlasKey) {
    this.currentAtlas = this.atlasManager.getAtlas(atlasKey);
  }

  drawSprite(ctx, spriteName, x, y, width, height, rotation = 0) {
    if (!this.currentAtlas) return;

    const slot = this.currentAtlas.slots.find(s => s.name === spriteName);
    if (!slot) return;

    ctx.save();
    ctx.translate(x + width / 2, y + height / 2);
    if (rotation) ctx.rotate(rotation);

    ctx.drawImage(
      this.currentAtlas.canvas,
      slot.x, slot.y, slot.width, slot.height,
      -width / 2, -height / 2, width, height
    );

    ctx.restore();
  }
}
```

### 2. Texture Packing

```javascript
class TexturePacker {
  constructor() {
    this.nodes = [];
    this.width = 2048;
    this.height = 2048;
  }

  pack(images) {
    this.nodes = [{ x: 0, y: 0, width: this.width, height: this.height, used: false }];
    const slots = [];

    images.forEach(image => {
      const slot = this.findSlot(image.width, image.height);
      if (slot) {
        this.markUsed(slot, image.width, image.height);
        slots.push({
          name: image.name,
          x: slot.x,
          y: slot.y,
          width: image.width,
          height: image.height
        });
      }
    });

    return slots;
  }

  findSlot(width, height) {
    for (const node of this.nodes) {
      if (!node.used && node.width >= width && node.height >= height) {
        return node;
      }
    }
    return null;
  }

  markUsed(node, width, height) {
    node.used = true;
    node.width = width;
    node.height = height;

    // Split remaining space
    const remainingX = this.width - (node.x + width);
    const remainingY = this.height - (node.y + height);

    if (remainingX > 0) {
      this.nodes.push({
        x: node.x + width,
        y: node.y,
        width: remainingX,
        height: height,
        used: false
      });
    }

    if (remainingY > 0) {
      this.nodes.push({
        x: node.x,
        y: node.y + height,
        width: width,
        height: remainingY,
        used: false
      });
    }
  }
}
```

### 3. Dynamic Texture Generation

```javascript
class DynamicTextureGenerator {
  constructor() {
    this.textures = new Map();
  }

  createTextTexture(text, options = {}) {
    const {
      font = '20px Arial',
      color = '#FFF',
      backgroundColor = 'transparent',
      padding = 4
    } = options;

    // Create temporary canvas
    const tempCanvas = document.createElement('canvas');
    const ctx = tempCanvas.getContext('2d');
    
    // Measure text
    ctx.font = font;
    const metrics = ctx.measureText(text);
    const textWidth = metrics.width;
    const textHeight = parseInt(font) || 20;

    // Set canvas size with padding
    tempCanvas.width = textWidth + padding * 2;
    tempCanvas.height = textHeight + padding * 2;

    // Draw background
    if (backgroundColor !== 'transparent') {
      ctx.fillStyle = backgroundColor;
      ctx.fillRect(0, 0, tempCanvas.width, tempCanvas.height);
    }

    // Draw text
    ctx.font = font;
    ctx.fillStyle = color;
    ctx.textBaseline = 'top';
    ctx.fillText(text, padding, padding);

    // Store texture
    const texture = tempCanvas;
    this.textures.set(text, texture);

    return texture;
  }

  createButtonTexture(width, height, options = {}) {
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');

    const {
      color = '#4169E1',
      hoverColor = '#6495ED',
      borderColor = '#FFF',
      borderWidth = 2,
      borderRadius = 4
    } = options;

    // Rounded rectangle
    ctx.fillStyle = color;
    this.drawRoundedRect(ctx, 0, 0, width, height, borderRadius);
    ctx.fill();

    // Border
    ctx.strokeStyle = borderColor;
    ctx.lineWidth = borderWidth;
    this.drawRoundedRect(ctx, 0, 0, width, height, borderRadius);
    ctx.stroke();

    return canvas;
  }

  createIconTexture(type, options = {}) {
    const canvas = document.createElement('canvas');
    const size = options.size || 32;
    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext('2d');

    const color = options.color || '#FFF';

    switch (type) {
      case 'play':
        this.drawPlayIcon(ctx, size, color);
        break;
      case 'pause':
        this.drawPauseIcon(ctx, size, color);
        break;
      case 'restart':
        this.drawRestartIcon(ctx, size, color);
        break;
      case 'sound':
        this.drawSoundIcon(ctx, size, color);
        break;
      case 'settings':
        this.drawSettingsIcon(ctx, size, color);
        break;
      case 'heart':
        this.drawHeartIcon(ctx, size, color);
        break;
      case 'star':
        this.drawStarIcon(ctx, size, color);
        break;
    }

    return canvas;
  }

  drawRoundedRect(ctx, x, y, width, height, radius) {
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

  drawPlayIcon(ctx, size, color) {
    const center = size / 2;
    const radius = size * 0.3;
    
    ctx.fillStyle = color;
    ctx.beginPath();
    ctx.moveTo(center - radius, center - radius * 0.5);
    ctx.lineTo(center + radius, center);
    ctx.lineTo(center - radius, center + radius * 0.5);
    ctx.closePath();
    ctx.fill();
  }

  drawPauseIcon(ctx, size, color) {
    const center = size / 2;
    const barWidth = size * 0.2;
    
    ctx.fillStyle = color;
    ctx.fillRect(center - barWidth, center - size * 0.3, barWidth, size * 0.6);
    ctx.fillRect(center + barWidth * 0.5, center - size * 0.3, barWidth, size * 0.6);
  }

  drawRestartIcon(ctx, size, color) {
    const center = size / 2;
    const radius = size * 0.35;
    
    ctx.strokeStyle = color;
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.arc(center, center, radius, 0.5, Math.PI * 1.5);
    ctx.stroke();

    ctx.beginPath();
    ctx.moveTo(center - radius * 0.7, center - radius * 0.7);
    ctx.lineTo(center, center - radius);
    ctx.lineTo(center + radius * 0.7, center - radius * 0.7);
    ctx.stroke();
  }

  drawSoundIcon(ctx, size, color) {
    const center = size / 2;
    const radius = size * 0.3;
    
    ctx.strokeStyle = color;
    ctx.lineWidth = 2;
    
    // Sound waves
    ctx.beginPath();
    ctx.arc(center, center, radius, 0, Math.PI * 2);
    ctx.stroke();

    ctx.beginPath();
    ctx.moveTo(center + radius, center - radius * 0.5);
    ctx.lineTo(center + radius * 1.5, center - radius * 0.25);
    ctx.moveTo(center + radius, center + radius * 0.5);
    ctx.lineTo(center + radius * 1.5, center + radius * 0.25);
    ctx.stroke();
  }

  drawHeartIcon(ctx, size, color) {
    const center = size / 2;
    const size2 = size * 0.3;
    
    ctx.fillStyle = color;
    ctx.beginPath();
    ctx.moveTo(center, center + size2 * 0.5);
    ctx.bezierCurveTo(center, center - size2 * 0.5, center - size2, center - size2 * 0.5, center - size2, center);
    ctx.bezierCurveTo(center - size2, center + size2 * 0.5, center, center + size2, center, center + size2 * 1.5);
    ctx.bezierCurveTo(center, center + size2 * 0.5, center + size2, center + size2 * 0.5, center + size2, center);
    ctx.bezierCurveTo(center + size2, center - size2 * 0.5, center, center - size2 * 0.5, center, center + size2 * 0.5);
    ctx.fill();
  }

  drawStarIcon(ctx, size, color) {
    const center = size / 2;
    const outerRadius = size * 0.4;
    const innerRadius = size * 0.2;
    
    ctx.fillStyle = color;
    ctx.beginPath();
    
    for (let i = 0; i < 5; i++) {
      const angle = (Math.PI / 2) + i * (Math.PI * 2 / 5);
      const x = center + Math.cos(angle) * outerRadius;
      const y = center + Math.sin(angle) * outerRadius;
      
      if (i === 0) {
        ctx.moveTo(x, y);
      } else {
        ctx.lineTo(x, y);
      }
      
      const innerAngle = angle + Math.PI / 5;
      const innerX = center + Math.cos(innerAngle) * innerRadius;
      const innerY = center + Math.sin(innerAngle) * innerRadius;
      ctx.lineTo(innerX, innerY);
    }
    
    ctx.closePath();
    ctx.fill();
  }

  // Cache textures
  getTexture(key) {
    return this.textures.get(key);
  }

  // Create dynamic UI elements
  createUITexture(type, text, options = {}) {
    switch (type) {
      case 'button':
        return this.createButtonTexture(120, 40, options);
      case 'text':
        return this.createTextTexture(text, options);
      case 'icon':
        return this.createIconTexture(text, options);
      default:
        return null;
    }
  }
}
```

---

## Visual Feedback Patterns

### 1. Screen Shake

```javascript
class ScreenShake {
  constructor() {
    this.active = false;
    this.intensity = 0;
    this.duration = 0;
    this.startTime = 0;
    this.x = 0;
    this.y = 0;
  }

  shake(intensity = 10, duration = 0.5) {
    this.active = true;
    this.intensity = intensity;
    this.duration = duration;
    this.startTime = performance.now();
  }

  update(deltaTime) {
    if (!this.active) return;

    const elapsed = (performance.now() - this.startTime) / 1000;
    
    if (elapsed >= this.duration) {
      this.active = false;
      this.x = 0;
      this.y = 0;
      return;
    }

    const progress = elapsed / this.duration;
    const remainingIntensity = this.intensity * (1 - progress);

    this.x = (Math.random() - 0.5) * remainingIntensity;
    this.y = (Math.random() - 0.5) * remainingIntensity;
  }

  getOffset() {
    return { x: this.x, y: this.y };
  }

  isActive() {
    return this.active;
  }
}
```

### 2. Hit Flash

```javascript
class HitFlash {
  constructor() {
    this.active = false;
    this.alpha = 0;
    this.color = '#FFF';
    this.duration = 0;
    this.startTime = 0;
  }

  flash(color = '#FFF', duration = 0.1) {
    this.active = true;
    this.color = color;
    this.duration = duration;
    this.startTime = performance.now();
    this.alpha = 1;
  }

  update(deltaTime) {
    if (!this.active) return;

    const elapsed = (performance.now() - this.startTime) / 1000;
    
    if (elapsed >= this.duration) {
      this.active = false;
      this.alpha = 0;
      return;
    }

    this.alpha = 1 - (elapsed / this.duration);
  }

  draw(ctx, width, height) {
    if (!this.active || this.alpha <= 0) return;

    ctx.fillStyle = this.color;
    ctx.globalAlpha = this.alpha;
    ctx.fillRect(0, 0, width, height);
    ctx.globalAlpha = 1;
  }

  isActive() {
    return this.active;
  }
}
```

### 3. Damage Numbers

```javascript
class DamageNumberSystem {
  constructor() {
    this.numbers = [];
    this.maxNumbers = 20;
  }

  spawn(x, y, amount, isCritical = false) {
    if (this.numbers.length >= this.maxNumbers) {
      this.numbers.shift();
    }

    this.numbers.push({
      x,
      y,
      amount,
      isCritical,
      life: 1.0,
      maxLife: 1.0,
      velocityY: -50,
      velocityX: (Math.random() - 0.5) * 30
    });
  }

  update(deltaTime) {
    for (let i = this.numbers.length - 1; i >= 0; i--) {
      const n = this.numbers[i];
      
      n.x += n.velocityX * deltaTime;
      n.y += n.velocityY * deltaTime;
      n.life -= deltaTime;
      
      if (n.life <= 0) {
        this.numbers.splice(i, 1);
      }
    }
  }

  draw(ctx) {
    this.numbers.forEach(n => {
      if (n.life <= 0) return;

      ctx.save();
      ctx.translate(n.x, n.y);
      
      const scale = 1 + (1 - n.life);
      ctx.scale(scale, scale);
      ctx.globalAlpha = n.life;

      ctx.font = n.isCritical ? 'bold 24px Arial' : 'bold 18px Arial';
      ctx.fillStyle = n.isCritical ? '#FFD700' : '#FFF';
      ctx.strokeStyle = '#000';
      ctx.lineWidth = 2;
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      
      ctx.strokeText(n.amount, 0, 0);
      ctx.fillText(n.amount, 0, 0);

      ctx.restore();
    });
  }

  clear() {
    this.numbers = [];
  }
}
```

### 4. Combo System Visual Feedback

```javascript
class ComboVisualFeedback {
  constructor() {
    this.combo = 0;
    this.lastHitTime = 0;
    this.window = 2.0;
    this.particles = [];
    this.maxParticles = 30;
  }

  addHit() {
    this.combo++;
    this.lastHitTime = performance.now() / 1000;
    this.spawnComboParticles();
  }

  update(deltaTime) {
    const currentTime = performance.now() / 1000;
    
    if (currentTime - this.lastHitTime > this.window) {
      this.combo = 0;
    }

    this.particles.forEach(p => {
      p.x += p.vx * deltaTime;
      p.y += p.vy * deltaTime;
      p.life -= deltaTime;
      p.scale += p.scaleChange * deltaTime;
    });

    this.particles = this.particles.filter(p => p.life > 0);
  }

  spawnComboParticles() {
    const comboColor = this.getComboColor();
    
    for (let i = 0; i < 10; i++) {
      this.particles.push({
        x: 0, // Would be player position
        y: 0,
        vx: (Math.random() - 0.5) * 100,
        vy: -50 - Math.random() * 50,
        life: 0.5 + Math.random() * 0.5,
        maxLife: 1.0,
        color: comboColor,
        scale: 0.5 + Math.random() * 0.5,
        scaleChange: 0.5
      });
    }
  }

  getComboColor() {
    if (this.combo < 5) return '#FFF';
    if (this.combo < 10) return '#4169E1';
    if (this.combo < 20) return '#FFD700';
    if (this.combo < 50) return '#FF4500';
    return '#DC143C';
  }

  draw(ctx) {
    // Draw combo text
    if (this.combo > 1) {
      ctx.save();
      ctx.translate(100, 50); // Position on screen
      
      const scale = 1 + Math.sin(performance.now() / 200) * 0.1;
      ctx.scale(scale, scale);
      
      ctx.font = 'bold 36px Arial';
      ctx.fillStyle = this.getComboColor();
      ctx.strokeStyle = '#000';
      ctx.lineWidth = 3;
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      
      ctx.strokeText(this.combo + ' COMBO', 0, 0);
      ctx.fillText(this.combo + ' COMBO', 0, 0);
      
      ctx.restore();
    }

    // Draw particles
    this.particles.forEach(p => {
      if (p.life <= 0) return;

      ctx.save();
      ctx.translate(p.x, p.y);
      ctx.scale(p.scale, p.scale);
      ctx.globalAlpha = p.life;
      ctx.fillStyle = p.color;
      ctx.beginPath();
      ctx.arc(0, 0, 3, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    });
  }

  reset() {
    this.combo = 0;
    this.lastHitTime = 0;
    this.particles = [];
  }

  getComboCount() {
    return this.combo;
  }

  getMaxCombo() {
    // Would track maximum combo reached
    return this.combo;
  }
}
```

### 5. Particle System with Visual Feedback

```javascript
class FeedbackParticleSystem {
  constructor() {
    this.particles = [];
    this.maxParticles = 100;
  }

  // Hit impact effect
  hitImpact(x, y, color = '#FFF') {
    this.emit(x, y, {
      count: 15,
      color: color,
      speed: 100,
      life: 0.5,
      shape: 'square',
      scale: 2,
      scaleChange: -2
    });
  }

  // Score bonus effect
  scoreBonus(x, y, amount) {
    this.emit(x, y, {
      count: 10,
      color: '#FFD700',
      speed: 80,
      life: 1.0,
      shape: 'star',
      scale: 1,
      scaleChange: 0.5
    });
  }

  // Powerup collection
  powerupCollect(x, y) {
    this.emit(x, y, {
      count: 20,
      color: '#00FFFF',
      speed: 120,
      life: 0.8,
      shape: 'circle',
      scale: 3,
      scaleChange: -3
    });
  }

  // Level up effect
  levelUp(x, y) {
    this.emit(x, y, {
      count: 50,
      color: '#FFD700',
      speed: 150,
      life: 1.5,
      shape: 'circle',
      scale: 2,
      scaleChange: -2
    });
  }

  emit(x, y, options) {
    for (let i = 0; i < options.count; i++) {
      if (this.particles.length >= this.maxParticles) {
        this.particles.shift();
      }

      const angle = Math.random() * Math.PI * 2;
      const speed = options.speed || 100;

      this.particles.push({
        x,
        y,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        life: options.life || 1.0,
        maxLife: options.life || 1.0,
        color: options.color || '#FFF',
        shape: options.shape || 'circle',
        scale: options.scale || 1,
        scaleChange: options.scaleChange || 0,
        rotation: Math.random() * Math.PI * 2,
        rotationSpeed: (Math.random() - 0.5) * 10
      });
    }
  }

  update(deltaTime) {
    for (let i = this.particles.length - 1; i >= 0; i--) {
      const p = this.particles[i];

      p.x += p.vx * deltaTime;
      p.y += p.vy * deltaTime;
      p.life -= deltaTime;
      p.scale += p.scaleChange * deltaTime;
      p.rotation += p.rotationSpeed * deltaTime;

      if (p.life <= 0 || p.scale <= 0) {
        this.particles.splice(i, 1);
      }
    }
  }

  draw(ctx) {
    this.particles.forEach(p => {
      if (p.life <= 0) return;

      ctx.save();
      ctx.translate(p.x, p.y);
      ctx.rotate(p.rotation);
      ctx.scale(p.scale, p.scale);
      ctx.globalAlpha = p.life;

      switch (p.shape) {
        case 'circle':
          ctx.fillStyle = p.color;
          ctx.beginPath();
          ctx.arc(0, 0, 4, 0, Math.PI * 2);
          ctx.fill();
          break;
        case 'square':
          ctx.fillStyle = p.color;
          ctx.fillRect(-4, -4, 8, 8);
          break;
        case 'triangle':
          ctx.fillStyle = p.color;
          ctx.beginPath();
          ctx.moveTo(0, -4);
          ctx.lineTo(4, 4);
          ctx.lineTo(-4, 4);
          ctx.closePath();
          ctx.fill();
          break;
        case 'star':
          ctx.fillStyle = p.color;
          this.drawStar(ctx, 0, 0, 5, 4, 2);
          break;
      }

      ctx.restore();
    });
  }

  drawStar(ctx, cx, cy, spikes, outerRadius, innerRadius) {
    let rot = Math.PI / 2 * 3;
    let x = cx;
    let y = cy;
    const step = Math.PI / spikes;

    ctx.beginPath();
    ctx.moveTo(cx, cy - outerRadius);
    
    for (let i = 0; i < spikes; i++) {
      x = cx + Math.cos(rot) * outerRadius;
      y = cy + Math.sin(rot) * outerRadius;
      ctx.lineTo(x, y);
      rot += step;

      x = cx + Math.cos(rot) * innerRadius;
      y = cy + Math.sin(rot) * innerRadius;
      ctx.lineTo(x, y);
      rot += step;
    }

    ctx.lineTo(cx, cy - outerRadius);
    ctx.closePath();
    ctx.fill();
  }

  clear() {
    this.particles = [];
  }
}
```

### 6. Visual Feedback Manager

```javascript
class VisualFeedbackManager {
  constructor() {
    this.shake = new ScreenShake();
    this.flash = new HitFlash();
    this.damageNumbers = new DamageNumberSystem();
    this.combo = new ComboVisualFeedback();
    this.particles = new FeedbackParticleSystem();
  }

  update(deltaTime) {
    this.shake.update(deltaTime);
    this.flash.update(deltaTime);
    this.damageNumbers.update(deltaTime);
    this.combo.update(deltaTime);
    this.particles.update(deltaTime);
  }

  draw(ctx, width, height) {
    this.shake.draw(ctx, width, height);
    this.flash.draw(ctx, width, height);
    this.damageNumbers.draw(ctx);
    this.combo.draw(ctx);
    this.particles.draw(ctx);
  }

  // Convenience methods
  triggerShake(intensity = 10, duration = 0.5) {
    this.shake.shake(intensity, duration);
  }

  triggerHitFlash(color = '#FFF', duration = 0.1) {
    this.flash.flash(color, duration);
  }

  spawnDamageNumber(x, y, amount, isCritical = false) {
    this.damageNumbers.spawn(x, y, amount, isCritical);
  }

  addComboHit() {
    this.combo.addHit();
  }

  spawnHitImpact(x, y, color = '#FFF') {
    this.particles.hitImpact(x, y, color);
  }

  // Clear all effects
  clear() {
    this.shake.x = 0;
    this.shake.y = 0;
    this.flash.active = false;
    this.damageNumbers.clear();
    this.combo.reset();
    this.particles.clear();
  }

  // Get combined offset for screen shake
  getScreenOffset() {
    return this.shake.getOffset();
  }

  // Check if any effects are active
  hasActiveEffects() {
    return (
      this.shake.isActive() ||
      this.flash.isActive() ||
      this.damageNumbers.numbers.length > 0 ||
      this.combo.getComboCount() > 1 ||
      this.particles.particles.length > 0
    );
  }
}
```
