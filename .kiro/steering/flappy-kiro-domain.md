# Flappy Kiro Domain Steering File

This steering file defines game state management, score persistence, and difficulty progression patterns for the Flappy Kiro game.

## Table of Contents

- [Game State Management](#game-state-management)
- [Score Persistence](#score-persistence)
- [Difficulty Progression](#difficulty-progress)
- [Domain Model Implementation](#domain-model-implementation)

---

## Game State Management

### 1. Game State Machine

```javascript
const GameState = {
  START: 'start',
  PLAYING: 'playing',
  PAUSED: 'paused',
  GAME_OVER: 'gameover',
  LEVEL_COMPLETE: 'levelcomplete'
};

class GameStateManager {
  constructor() {
    this.currentState = GameState.START;
    this.previousState = null;
    this.states = new Map();
    this.transitionData = {};
  }

  addState(name, state) {
    this.states.set(name, state);
  }

  transition(toState, data = {}) {
    const fromState = this.currentState;
    
    if (this.states.has(fromState)) {
      this.states.get(fromState).exit(data);
    }

    this.previousState = fromState;
    this.currentState = toState;
    this.transitionData = { ...data, from: fromState, to: toState };

    if (this.states.has(toState)) {
      this.states.get(toState).enter(data);
    }

    this.onTransition && this.onTransition(this.transitionData);
  }

  update(deltaTime) {
    if (this.states.has(this.currentState)) {
      this.states.get(this.currentState).update(deltaTime);
    }
  }

  render(ctx) {
    if (this.states.has(this.currentState)) {
      this.states.get(this.currentState).render(ctx);
    }
  }

  is(state) {
    return this.currentState === state;
  }

  canTransition(toState) {
    const currentState = this.states.get(this.currentState);
    return currentState ? currentState.canTransition(toState) : true;
  }

  // State shortcuts
  start() {
    this.transition(GameState.PLAYING);
  }

  pause() {
    if (this.currentState === GameState.PLAYING) {
      this.transition(GameState.PAUSED);
    }
  }

  resume() {
    if (this.currentState === GameState.PAUSED) {
      this.transition(GameState.PLAYING);
    }
  }

  gameOver(score) {
    this.transition(GameState.GAME_OVER, { score });
  }

  levelComplete() {
    this.transition(GameState.LEVEL_COMPLETE);
  }

  // Event hooks
  onTransition(callback) {
    this.onTransition = callback;
  }
}

// Base State Class
class GameState {
  constructor(name) {
    this.name = name;
    this.game = null;
  }

  setGame(game) {
    this.game = game;
  }

  enter(data) {}
  
  exit(data) {}
  
  update(deltaTime) {}
  
  render(ctx) {}

  canTransition(toState) {
    return true;
  }
}
```

### 2. Game States

```javascript
// Start State
class StartState extends GameState {
  constructor() {
    super('start');
  }

  enter(data) {
    this.startTime = performance.now();
  }

  update(deltaTime) {
    if (this.game.inputHandler.isPressed('Space') || 
        this.game.inputHandler.isClicked()) {
      this.game.gameState.transition(GameState.PLAYING);
    }
  }

  render(ctx) {
    ctx.fillStyle = '#87CEEB';
    ctx.fillRect(0, 0, this.game.width, this.game.height);

    ctx.fillStyle = '#FFF';
    ctx.textAlign = 'center';
    ctx.font = 'bold 48px Arial';
    ctx.fillText('FLAPPY KIRO', this.game.width / 2, this.game.height / 3);

    ctx.font = '24px Arial';
    ctx.fillText('Press SPACE or Click to Start', this.game.width / 2, this.game.height / 2);
    
    // Draw high score
    ctx.fillStyle = '#FFD700';
    ctx.font = '20px Arial';
    ctx.fillText(`High Score: ${this.game.scoreManager.highScore}`, this.game.width / 2, this.game.height / 2 + 50);
  }
}

// Playing State
class PlayingState extends GameState {
  constructor() {
    super('playing');
    this.score = 0;
    this.gameOver = false;
  }

  enter(data) {
    this.score = 0;
    this.gameOver = false;
    this.startTime = performance.now();
    
    // Reset game entities
    this.game.player.reset();
    this.game.wallSpawner.reset();
    this.game.scoreManager.score = 0;
  }

  update(deltaTime) {
    // Check for pause
    if (this.game.inputHandler.isPressed('Escape')) {
      this.game.gameState.pause();
      return;
    }

    // Update game entities
    this.game.player.update(deltaTime);
    this.game.wallSpawner.update(deltaTime, this.score);
    this.game.scoreManager.update(deltaTime);

    // Check collisions
    if (this.checkCollisions()) {
      this.game.gameOver(this.score);
      return;
    }

    // Check score
    this.checkScore();

    // Check boundaries
    if (this.game.player.y > this.game.height - 50 || this.game.player.y < 0) {
      this.game.gameOver(this.score);
    }
  }

  checkCollisions() {
    const player = this.game.player;
    
    // Ground collision
    if (player.y + player.height > this.game.height - 50) {
      return true;
    }

    // Ceiling collision
    if (player.y < 0) {
      return true;
    }

    // Pipe collisions
    for (const wall of this.game.wallSpawner.walls) {
      if (player.x < wall.x + wall.width &&
          player.x + player.width > wall.x &&
          player.y < wall.y + wall.height &&
          player.y + player.height > wall.y) {
        return true;
      }
    }

    return false;
  }

  checkScore() {
    for (const wall of this.game.wallSpawner.walls) {
      if (wall.type === 'top' && !wall.passed && wall.x + wall.width < this.game.player.x) {
        wall.passed = true;
        this.score += 1;
        this.game.scoreManager.addScore(1);
        this.game.audioManager.playSound('score');
      }
    }
  }

  render(ctx) {
    ctx.fillStyle = '#87CEEB';
    ctx.fillRect(0, 0, this.game.width, this.game.height);

    // Draw game entities
    this.game.wallSpawner.draw(ctx);
    this.game.player.draw(ctx);

    // Draw score
    ctx.fillStyle = '#FFF';
    ctx.textAlign = 'center';
    ctx.font = 'bold 36px Arial';
    ctx.fillText(this.score, this.game.width / 2, 50);
  }
}

// Paused State
class PausedState extends GameState {
  constructor() {
    super('paused');
  }

  enter(data) {
    this.pausedTime = performance.now();
  }

  update(deltaTime) {
    if (this.game.inputHandler.isPressed('Escape') || 
        this.game.inputHandler.isPressed('Space') ||
        this.game.inputHandler.isClicked()) {
      this.game.gameState.resume();
    }
  }

  render(ctx) {
    ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
    ctx.fillRect(0, 0, this.game.width, this.game.height);

    ctx.fillStyle = '#FFF';
    ctx.textAlign = 'center';
    ctx.font = 'bold 48px Arial';
    ctx.fillText('PAUSED', this.game.width / 2, this.game.height / 2);
    
    ctx.font = '24px Arial';
    ctx.fillText('Press SPACE or Click to Resume', this.game.width / 2, this.game.height / 2 + 50);
  }
}

// Game Over State
class GameOverState extends GameState {
  constructor() {
    super('gameover');
  }

  enter(data) {
    this.game.scoreManager.updateHighScore();
    this.game.audioManager.playMusic('gameover');
  }

  update(deltaTime) {
    if (this.game.inputHandler.isPressed('Space') || 
        this.game.inputHandler.isClicked()) {
      this.game.gameState.transition(GameState.START);
    }
  }

  render(ctx) {
    ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
    ctx.fillRect(0, 0, this.game.width, this.game.height);

    ctx.fillStyle = '#FFF';
    ctx.textAlign = 'center';
    ctx.font = 'bold 48px Arial';
    ctx.fillText('GAME OVER', this.game.width / 2, this.game.height / 3);

    ctx.font = '24px Arial';
    ctx.fillText(`Score: ${this.game.scoreManager.score}`, this.game.width / 2, this.game.height / 2);
    ctx.fillText(`High Score: ${this.game.scoreManager.highScore}`, this.game.width / 2, this.game.height / 2 + 40);
    
    ctx.font = '20px Arial';
    ctx.fillText('Press SPACE or Click to Play Again', this.game.width / 2, this.game.height / 2 + 80);
  }
}
```

### 3. Game Context with State Management

```javascript
class FlappyKiroGame {
  constructor(canvasId) {
    this.canvas = document.getElementById(canvasId);
    this.ctx = this.canvas.getContext('2d');
    this.width = 320;
    this.height = 480;
    this.canvas.width = this.width;
    this.canvas.height = this.height;

    // Systems
    this.audioManager = new AudioManager();
    this.inputHandler = new InputHandler(this);
    this.scoreManager = new ScoreManager();
    this.wallSpawner = new WallSpawner(this.width, this.height);
    this.player = new Player(this);
    this.gameState = new GameStateManager();

    // Add states
    this.gameState.addState('start', new StartState());
    this.gameState.addState('playing', new PlayingState());
    this.gameState.addState('paused', new PausedState());
    this.gameState.addState('gameover', new GameOverState());

    this.gameState.transition('start');

    this.lastTime = 0;
    this.isRunning = false;
  }

  start() {
    this.isRunning = true;
    this.lastTime = performance.now();
    this.gameLoop();
  }

  stop() {
    this.isRunning = false;
  }

  gameOver(score) {
    this.gameState.gameOver(score);
  }

  update(deltaTime) {
    this.gameState.update(deltaTime);
  }

  render() {
    this.gameState.render(this.ctx);
  }

  gameLoop(timestamp) {
    if (!this.isRunning) return;

    const deltaTime = (timestamp - this.lastTime) / 1000;
    this.lastTime = timestamp;

    this.update(deltaTime);
    this.render();

    requestAnimationFrame(timestamp => this.gameLoop(timestamp));
  }

  reset() {
    this.scoreManager.score = 0;
    this.wallSpawner.reset();
    this.player.reset();
  }
}
```

---

## Score Persistence

### 1. Score Manager with Local Storage

```javascript
class ScoreManager {
  constructor() {
    this.score = 0;
    this.highScore = this.getHighScore();
    this.combo = 0;
    this.lastScoreTime = 0;
    this.comboWindow = 2.0; // seconds
  }

  addScore(points) {
    this.score += points;
    this.lastScoreTime = Date.now() / 1000;
    this.combo++;

    // Combo multiplier
    if (this.combo > 1 && this.combo % 5 === 0) {
      const multiplier = Math.min(1 + Math.floor(this.combo / 5), 3);
      this.score += points * multiplier;
    }

    // Save progress periodically
    this.saveProgress();
  }

  update(deltaTime) {
    // Check combo window
    if (Date.now() / 1000 - this.lastScoreTime > this.comboWindow) {
      this.combo = 0;
    }
  }

  getHighScore() {
    const saved = localStorage.getItem('flappyKiro_highScore');
    return saved ? parseInt(saved, 10) : 0;
  }

  updateHighScore() {
    if (this.score > this.highScore) {
      this.highScore = this.score;
      localStorage.setItem('flappyKiro_highScore', this.highScore);
    }
  }

  saveProgress() {
    const progress = {
      highScore: this.highScore,
      score: this.score,
      timestamp: Date.now()
    };
    localStorage.setItem('flappyKiro_progress', JSON.stringify(progress));
  }

  loadProgress() {
    const saved = localStorage.getItem('flappyKiro_progress');
    if (saved) {
      try {
        const progress = JSON.parse(saved);
        this.score = progress.score || 0;
        this.highScore = progress.highScore || 0;
        return true;
      } catch (e) {
        console.warn('Failed to load progress:', e);
        return false;
      }
    }
    return false;
  }

  clearProgress() {
    localStorage.removeItem('flappyKiro_progress');
    this.score = 0;
    this.combo = 0;
  }

  // Multiplayer score submission
  submitScore(playerName, callback) {
    const scoreData = {
      playerName,
      score: this.score,
      timestamp: Date.now(),
      device: navigator.userAgent
    };

    // In a real implementation, this would make an API call
    // fetch('/api/scores', {
    //   method: 'POST',
    //   headers: { 'Content-Type': 'application/json' },
    //   body: JSON.stringify(scoreData)
    // });

    callback && callback(scoreData);
  }

  // Score statistics
  getScoreStats() {
    return {
      score: this.score,
      highScore: this.highScore,
      combo: this.combo,
      comboMultiplier: Math.min(1 + Math.floor(this.combo / 5), 3)
    };
  }

  // Reset for new game
  reset() {
    this.score = 0;
    this.combo = 0;
    this.lastScoreTime = Date.now() / 1000;
  }
}
```

### 2. Persistent Data Storage

```javascript
class PersistentDataManager {
  constructor() {
    this.data = {};
    this.storage = localStorage;
    this.loadAll();
  }

  loadAll() {
    const keys = ['flappyKiro_highScore', 'flappyKiro_progress'];
    keys.forEach(key => {
      const value = this.storage.getItem(key);
      if (value) {
        try {
          this.data[key] = JSON.parse(value);
        } catch (e) {
          this.data[key] = value;
        }
      }
    });
  }

  get(key, defaultValue = null) {
    return this.data[key] !== undefined ? this.data[key] : defaultValue;
  }

  set(key, value) {
    this.data[key] = value;
    this.storage.setItem(key, JSON.stringify(value));
  }

  remove(key) {
    delete this.data[key];
    this.storage.removeItem(key);
  }

  // Player profile
  getPlayerProfile() {
    return {
      name: this.get('playerName', 'Guest'),
      highScore: this.get('flappyKiro_highScore', 0),
      gamesPlayed: this.get('gamesPlayed', 0),
      totalScore: this.get('totalScore', 0),
      levelUnlocks: this.get('levelUnlocks', [])
    };
  }

  updatePlayerProfile(updates) {
    const profile = this.getPlayerProfile();
    const newProfile = { ...profile, ...updates };
    
    this.set('playerName', newProfile.name);
    this.set('gamesPlayed', newProfile.gamesPlayed);
    this.set('totalScore', newProfile.totalScore);
    this.set('levelUnlocks', newProfile.levelUnlocks);

    return newProfile;
  }

  // Level progression
  unlockLevel(level) {
    const levels = this.get('levelUnlocks', []);
    if (!levels.includes(level)) {
      levels.push(level);
      this.set('levelUnlocks', levels);
      return true;
    }
    return false;
  }

  isLevelUnlocked(level) {
    const levels = this.get('levelUnlocks', []);
    return levels.includes(level);
  }

  // Clear all data
  clearAll() {
    this.data = {};
    this.storage.clear();
  }

  // Backup and restore
  createBackup() {
    const backup = JSON.stringify(this.data);
    const timestamp = Date.now();
    this.storage.setItem(`flappyKiro_backup_${timestamp}`, backup);
    return timestamp;
  }

  restoreFromBackup(timestamp) {
    const backupKey = `flappyKiro_backup_${timestamp}`;
    const backup = this.storage.getItem(backupKey);
    
    if (backup) {
      try {
        this.data = JSON.parse(backup);
        this.updateStorage();
        return true;
      } catch (e) {
        return false;
      }
    }
    return false;
  }

  updateStorage() {
    Object.entries(this.data).forEach(([key, value]) => {
      this.storage.setItem(key, JSON.stringify(value));
    });
  }
}
```

### 3. Cloud Save System

```javascript
class CloudSaveSystem {
  constructor() {
    this.backupManager = new PersistentDataManager();
    this.lastSync = null;
    this.isOnline = navigator.onLine;
    
    window.addEventListener('online', () => {
      this.isOnline = true;
      this.syncIfNeeded();
    });

    window.addEventListener('offline', () => {
      this.isOnline = false;
    });
  }

  async sync() {
    if (!this.isOnline) {
      console.warn('No internet connection. Changes will be synced when online.');
      return;
    }

    try {
      const data = {
        highScore: this.backupManager.get('flappyKiro_highScore', 0),
        progress: this.backupManager.get('flappyKiro_progress'),
        profile: this.backupManager.getPlayerProfile()
      };

      // In a real implementation, this would make an API call
      // const response = await fetch('/api/save', {
      //   method: 'POST',
      //   headers: { 'Content-Type': 'application/json' },
      //   body: JSON.stringify(data)
      // });

      // const result = await response.json();
      
      this.lastSync = Date.now();
      console.log('Save synced successfully');
    } catch (error) {
      console.error('Sync failed:', error);
      this.backupManager.set('needsSync', true);
    }
  }

  async load() {
    try {
      // In a real implementation, this would make an API call
      // const response = await fetch('/api/load');
      // const data = await response.json();

      // const loaded = this.backupManager.get('flappyKiro_highScore');
      // if (loaded && loaded > this.backupManager.get('flappyKiro_highScore')) {
      //   this.backupManager.set('flappyKiro_highScore', loaded);
      // }

      this.lastSync = Date.now();
      console.log('Load completed');
    } catch (error) {
      console.error('Load failed:', error);
    }
  }

  syncIfNeeded() {
    if (this.backupManager.get('needsSync')) {
      this.sync();
    }
  }

  // Auto-save
  scheduleAutoSave(interval = 60000) { // 1 minute
    this.autoSaveInterval = setInterval(() => {
      if (this.isOnline) {
        this.sync();
      }
    }, interval);
  }

  cancelAutoSave() {
    if (this.autoSaveInterval) {
      clearInterval(this.autoSaveInterval);
    }
  }
}
```

### 4. Leaderboard System

```javascript
class LeaderboardSystem {
  constructor(maxEntries = 10) {
    this.maxEntries = maxEntries;
    this.scores = this.loadLeaderboard();
  }

  loadLeaderboard() {
    const saved = localStorage.getItem('flappyKiro_leaderboard');
    return saved ? JSON.parse(saved) : [];
  }

  saveLeaderboard() {
    localStorage.setItem('flappyKiro_leaderboard', JSON.stringify(this.scores));
  }

  addScore(playerName, score) {
    const entry = {
      playerName,
      score,
      timestamp: Date.now()
    };

    this.scores.push(entry);
    this.scores.sort((a, b) => b.score - a.score);
    this.scores = this.scores.slice(0, this.maxEntries);
    this.saveLeaderboard();

    return this.getRank(score);
  }

  getRank(score) {
    return this.scores.findIndex(s => s.score <= score) + 1;
  }

  getTopScores() {
    return this.scores;
  }

  getTopScore() {
    return this.scores.length > 0 ? this.scores[0].score : 0;
  }

  isHighScore(score) {
    if (this.scores.length < this.maxEntries) {
      return true;
    }
    return score > this.scores[this.scores.length - 1].score;
  }

  clearLeaderboard() {
    this.scores = [];
    this.saveLeaderboard();
  }
}
```

---

## Difficulty Progression

### 1. Difficulty Manager

```javascript
class DifficultyManager {
  constructor() {
    this.level = 1;
    this.difficulty = 1.0;
    this.baseSpeed = 100;
    this.baseSpawnRate = 2000;
    this.baseGap = 120;
    this.scoreAtLastLevelUp = 0;
  }

  get currentSpeed() {
    return this.baseSpeed * this.difficulty;
  }

  get currentSpawnRate() {
    return Math.max(600, this.baseSpawnRate / this.difficulty);
  }

  get currentGap() {
    return Math.max(80, this.baseGap / this.difficulty);
  }

  updateScore(score) {
    // Level up every 5 points
    if (score > 0 && Math.floor(score / 5) > this.scoreAtLastLevelUp) {
      this.levelUp();
    }

    // Increase difficulty based on score
    this.difficulty = 1 + score * 0.05;
  }

  levelUp() {
    this.level++;
    this.scoreAtLastLevelUp = Math.floor(this.scoreAtLastLevelUp / 5) * 5;
    this.onLevelUp && this.onLevelUp(this.level);
  }

  // Difficulty progression types
  setProgression(type) {
    switch (type) {
      case 'linear':
        this.difficulty = 1 + this.level * 0.1;
        break;
      case 'exponential':
        this.difficulty = Math.pow(1.1, this.level);
        break;
      case 'logarithmic':
        this.difficulty = 1 + Math.log(this.level + 1);
        break;
      case 'custom':
        // Custom difficulty curve
        this.difficulty = 1 + Math.min(0.5, this.level * 0.05) * (1 + this.level / 10);
        break;
    }
  }

  // Difficulty levels
  getDifficultyInfo() {
    return {
      level: this.level,
      difficulty: this.difficulty,
      speed: this.currentSpeed,
      spawnRate: this.currentSpawnRate,
      gap: this.currentGap,
      progressionType: this.progressionType
    };
  }

  // Difficulty events
  onLevelUp(callback) {
    this.onLevelUp = callback;
  }
}
```

### 2. Progressive Challenge System

```javascript
class ProgressiveChallengeSystem {
  constructor(difficultyManager) {
    this.difficultyManager = difficultyManager;
    this.challenges = new Map();
    this.completedChallenges = new Set();
    this.progress = 0;
  }

  addChallenge(key, config) {
    this.challenges.set(key, config);
  }

  updateChallengeProgress(key, progress) {
    const challenge = this.challenges.get(key);
    if (challenge) {
      challenge.currentProgress += progress;
      this.checkChallengeComplete(key);
    }
  }

  checkChallengeComplete(key) {
    const challenge = this.challenges.get(key);
    if (challenge && challenge.currentProgress >= challenge.targetProgress) {
      this.completedChallenges.add(key);
      this.onChallengeComplete && this.onChallengeComplete(key, challenge);
    }
  }

  getChallengeProgress(key) {
    const challenge = this.challenges.get(key);
    if (!challenge) return 0;
    
    return challenge.currentProgress / challenge.targetProgress;
  }

  // Specific challenge types
  addScoreChallenge(targetScore) {
    this.addChallenge(`score_${targetScore}`, {
      type: 'score',
      targetProgress: targetScore,
      currentProgress: 0,
      reward: 100
    });
  }

  addComboChallenge(targetCombo) {
    this.addChallenge(`combo_${targetCombo}`, {
      type: 'combo',
      targetProgress: targetCombo,
      currentProgress: 0,
      reward: 50
    });
  }

  addSurvivalChallenge(duration) {
    this.addChallenge(`survival_${duration}`, {
      type: 'survival',
      targetProgress: duration,
      currentProgress: 0,
      reward: 200
    });
  }

  onChallengeComplete(callback) {
    this.onChallengeComplete = callback;
  }

  // Progress calculation
  getTotalProgress() {
    const totalTarget = Array.from(this.challenges.values())
      .reduce((sum, c) => sum + c.targetProgress, 0);
    
    const totalCurrent = Array.from(this.challenges.values())
      .reduce((sum, c) => sum + c.currentProgress, 0);

    return totalTarget > 0 ? totalCurrent / totalTarget : 0;
  }
}
```

### 3. Adaptive Difficulty

```javascript
class AdaptiveDifficulty {
  constructor(baseDifficultyManager) {
    this.baseDifficultyManager = baseDifficultyManager;
    this.playerPerformance = {
      jumps: 0,
      successfulPasses: 0,
      failures: 0,
      avgJumpAccuracy: 0
    };
    this.difficultyHistory = [];
    this.adaptationRate = 0.1;
  }

  recordJump(success) {
    this.playerPerformance.jumps++;
    
    if (success) {
      this.playerPerformance.successfulPasses++;
    } else {
      this.playerPerformance.failures++;
    }

    // Calculate jump accuracy
    const totalJumps = this.playerPerformance.successfulPasses + this.playerPerformance.failures;
    this.playerPerformance.avgJumpAccuracy = 
      this.playerPerformance.successfulPasses / totalJumps;
  }

  adaptDifficulty() {
    const accuracy = this.playerPerformance.avgJumpAccuracy || 0.5;
    
    // If player is doing well, increase difficulty
    if (accuracy > 0.7) {
      this.baseDifficultyManager.difficulty += this.adaptationRate;
    }
    // If player is struggling, decrease difficulty
    else if (accuracy < 0.3) {
      this.baseDifficultyManager.difficulty -= this.adaptationRate;
      this.baseDifficultyManager.difficulty = Math.max(1.0, this.baseDifficultyManager.difficulty);
    }
  }

  // Difficulty adjustment based on performance
  getAdjustedDifficulty() {
    this.adaptDifficulty();
    return this.baseDifficultyManager.difficulty;
  }

  // Reset performance tracking
  resetPerformance() {
    this.playerPerformance = {
      jumps: 0,
      successfulPasses: 0,
      failures: 0,
      avgJumpAccuracy: 0
    };
  }

  // Difficulty curve customization
  getDifficultyCurve() {
    const curve = this.difficultyHistory.map((d, i) => ({
      level: i + 1,
      difficulty: d
    }));
    return curve;
  }
}
```

### 4. Achievement System

```javascript
class AchievementSystem {
  constructor(scoreManager, leaderboard) {
    this.scoreManager = scoreManager;
    this.leaderboard = leaderboard;
    this.achievements = this.loadAchievements();
    this.completed = new Set(this.loadCompleted());
  }

  loadAchievements() {
    return [
      { id: 'first_flight', name: 'First Flight', description: 'Complete your first game', threshold: 1 },
      { id: 'high_scorer', name: 'High Scorer', description: 'Reach a score of 10', threshold: 10 },
      { id: 'master', name: 'Master', description: 'Reach a score of 20', threshold: 20 },
      { id: 'grand_master', name: 'Grand Master', description: 'Reach a score of 50', threshold: 50 },
      { id: 'perfect_game', name: 'Perfect Game', description: 'Get 50 points without missing', threshold: 50 },
      { id: 'combo_master', name: 'Combo Master', description: 'Get a 10-point combo', threshold: 10 },
      { id: 'streak', name: 'Streak', description: 'Play 5 games', threshold: 5 }
    ];
  }

  loadCompleted() {
    const saved = localStorage.getItem('flappyKiro_achievements');
    return saved ? JSON.parse(saved) : [];
  }

  checkAchievements(score, combo, gamesPlayed) {
    this.achievements.forEach(achievement => {
      if (this.completed.has(achievement.id)) return;

      if (this.checkAchievementCondition(achievement, score, combo, gamesPlayed)) {
        this.completeAchievement(achievement);
      }
    });
  }

  checkAchievementCondition(achievement, score, combo, gamesPlayed) {
    switch (achievement.id) {
      case 'first_flight':
        return gamesPlayed >= 1;
      case 'high_scorer':
        return score >= 10;
      case 'master':
        return score >= 20;
      case 'grand_master':
        return score >= 50;
      case 'perfect_game':
        // Would track misses separately
        return combo >= 50;
      case 'combo_master':
        return combo >= 10;
      case 'streak':
        return gamesPlayed >= 5;
      default:
        return false;
    }
  }

  completeAchievement(achievement) {
    this.completed.add(achievement.id);
    this.saveCompleted();
    
    this.onAchievementComplete && 
      this.onAchievementComplete(achievement);
  }

  saveCompleted() {
    localStorage.setItem('flappyKiro_achievements', JSON.stringify([...this.completed]));
  }

  onAchievementComplete(callback) {
    this.onAchievementComplete = callback;
  }

  getCompletedCount() {
    return this.completed.size;
  }

  getTotalAchievements() {
    return this.achievements.length;
  }
}
```
