# Implementation Plan: Flappy Kiro

## Overview

This implementation plan breaks down the Flappy Kiro game development into 15+ major tasks covering all 15 requirements. The plan follows a component-based approach with clear separation of concerns. Property-based tests are included where applicable to validate correctness properties defined in the design document.

## Tasks

- [ ] 1. Set up project structure and core interfaces
  - Create directory structure (src/, src/components/, src/types/, src/utils/, tests/)
  - Define TypeScript interfaces for all data models (Ghost, Pipe, Cloud, GameSession, DifficultySettings)
  - Set up main game loop structure with update and render phases
  - Create core utility functions for random number generation and collision detection
  - _Requirements: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15_

- [ ] 1.5 Create Configuration System
  - Create `src/config/constants.ts` with all numerical values for game parameters
  - Define TypeScript interfaces for configuration (GameConfig, DifficultySettings, etc.)
  - Organize values into logical groups: physics, entities, pipes, clouds, scoring, difficulty, performance, layout, colors
  - Add comments documenting each parameter's purpose
  - Create `src/config/settings.ts` with tunable parameters (for game balance)
  - Create `src/config/profiles.ts` with difficulty presets (easy, normal, hard)
  - Implement simple config loader that can merge default config with overrides
  - _Requirements: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15 (all requirements via configuration)_

- [ ] 2. Implement Game Manager and State Management
  - [ ] 2.1 Create GameState enum and GameSession interface
    - Define Start, Playing, Game Over states
    - Implement state transition logic
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_
  
  - [ ] 2.2 Implement Game Manager component
    - Handle state transitions based on game events
    - Coordinate component updates during Playing state
    - Pause physics calculations during Game Over state
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5_

- [ ] 3. Implement Input Handler
  - [ ] 3.1 Create Input Handler component
    - Register keyboard (spacebar), mouse (click), and touch events
    - Prioritize most recent input when multiple occur simultaneously
    - _Requirements: 8.1, 8.2, 8.3, 8.4_
  
  - [ ] 3.2 Implement input-to-action mapping
    - Map valid inputs to flap action
    - Ensure inputs only processed in Playing state
    - Log ignored inputs from other states
    - _Requirements: 1.1, 1.2, 4, 8.1, 8.2, 8.3, 8.4_

- [ ] 4. Implement Physics Engine
  - [ ] 4.1 Create Physics Engine component
    - Calculate ghost position using delta time: `y_new = y_old + velocity * delta_time`
    - Apply constant gravity (500 px/s² downward)
    - Apply flap velocity (200 px/s upward on input)
    - _Requirements: 1.3, 1.4, 12.3, 12.4_
  
  - [ ] 4.2 Implement delta time capping
    - Cap delta time at 0.1 seconds to prevent physics anomalies
    - Test with simulated lag spikes
    - _Requirements: 12.4_
  
  - [ ]* 4.3 Write property test for frame-rate independent movement
    - **Property 1: Frame-rate independent movement**
    - **Validates: Requirements 12.3, 12.4**
    - Test that position updates are consistent across varying delta times (0.001 to 0.1 seconds)
    - Verify same result for scaled velocities

- [ ] 5. Implement Pipe Spawner System
  - [ ] 5.1 Create Pipe Spawner component
    - Generate pipes at regular intervals based on game speed
    - Position pipes on right side when previous pipe exits left side
    - _Requirements: 2.1_
  
  - [ ] 5.2 Implement gap randomization
    - Randomly position gap between 100-250 pixels tall
    - Ensure gap stays within bounds (50px minimum from top and bottom edges)
    - _Requirements: 2.2, 2.3, 2.5_
  
  - [ ] 5.3 Implement pipe movement
    - Move pipes leftward at configurable speed (starting at 150 px/s)
    - Update positions each frame using delta time
    - _Requirements: 2.4_
  
  - [ ]* 5.4 Write property test for pipe spacing consistency
    - **Property 2: Pipe spacing consistency**
    - **Validates: Requirements 2.1**
    - Test that distance between consecutive pipes equals interval * speed
    - Generate varying speeds and verify spacing remains consistent

- [ ] 6. Implement Cloud Manager for Perspective Effects
  - [ ] 6.1 Create Cloud Manager component
    - Generate at least 3 cloud elements at random vertical positions on load
    - _Requirements: 15.5_
  
  - [ ] 6.2 Implement cloud movement
    - Move near clouds at 100 px/s, far clouds at 50 px/s
    - Update positions each frame using delta time
    - _Requirements: 15.3_
  
  - [ ] 6.3 Implement cloud recycling
    - Reposition clouds that exit left side to right side
    - Maintain random vertical positions
    - _Requirements: 15.4_
  
  - [ ] 6.4 Set cloud transparency
    - Assign transparency values between 30% and 70%
    - Store opacity value in Cloud data model
    - _Requirements: 15.2_

- [ ] 7. Implement Collision Detection System
  - [ ] 7.1 Create Collision Detector component
    - Detect ghost-pipe intersections using AABB collision detection
    - _Requirements: 3.1_
  
  - [ ] 7.2 Implement boundary detection
    - Detect ghost-bottom intersection (game over)
    - Detect ghost-top intersection (game over)
    - _Requirements: 3.2, 3.3_
  
  - [ ] 7.3 Implement collision time recording
    - Record collision timestamp for visual effects
    - Store in GameSession object
    - _Requirements: 3.4_
  
  - [ ] 7.4 Implement failsafe mechanism
    - Log collision detection failures
    - Ensure Game Over state triggers even if primary detection fails
    - _Requirements: 3.5_
  
  - [ ]* 7.5 Write property test for collision detection completeness
    - **Property 6: Collision detection completeness**
    - **Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**
    - Test all ghost-pipe and ghost-boundary intersections trigger Game Over
    - Generate edge cases (ghost barely touching pipes)

- [ ] 8. Implement Score Manager
  - [ ] 8.1 Create Score Manager component
    - Track current score (increment on pipe passage)
    - _Requirements: 4.1_
  
  - [ ] 8.2 Implement pipe passage detection
    - Detect when ghost successfully passes through pipe gap
    - Mark pipe as passed to prevent double counting
    - _Requirements: 4.1_
  
  - [ ] 8.3 Implement score preservation
    - Preserve final score on Game Over
    - _Requirements: 4.2_
  
  - [ ] 8.4 Implement score reset
    - Reset score to 0 on game start/restart
    - _Requirements: 4.3_
  
  - [ ] 8.5 Implement score display locking
    - Display frozen score during Game Over state
    - _Requirements: 4.4_
  
  - [ ]* 8.6 Write property test for score accuracy
    - **Property 3: Score accuracy**
    - **Validates: Requirements 4.1, 4.2**
    - Test that score equals number of unique pipes passed
    - Test that passing same pipe multiple times doesn't increase score

- [ ] 9. Implement Visual Rendering System
  - [ ] 9.1 Create Renderer component
    - Draw ghost at current position each frame
    - Draw all active pipes at their current positions
    - Clear and redraw affected areas on state changes
    - _Requirements: 6.1, 6.2, 6.5_
  
  - [ ] 9.2 Implement start screen rendering
    - Display game title and instructions
    - Render only during Start state
    - _Requirements: 6.3_
  
  - [ ] 9.3 Implement game over screen rendering
    - Display final score and restart option
    - Render only during Game Over state
    - _Requirements: 6.4_
  
  - [ ] 9.4 Implement UI Manager
    - Render UI elements only during designated states
    - Handle score display, high score display, instructions
    - _Requirements: 6.6_
  
  - [ ] 9.5 Implement cloud rendering
    - Draw cloud elements with correct opacity behind game elements
    - _Requirements: 15.1_

- [ ] 10. Implement Audio System
  - [ ] 10.1 Create Audio Manager component
    - Initialize audio context with proper error handling
    - _Requirements: 7.5_
  
  - [ ] 10.2 Implement jump sound
    - Play jump sound when player triggers flap
    - _Requirements: 7.1_
  
  - [ ] 10.3 Implement game over sound
    - Play game over sound when collision occurs
    - _Requirements: 7.2_
  
  - [ ] 10.4 Implement score sound
    - Play score sound when ghost passes through pipe
    - _Requirements: 7.3_
  
  - [ ] 10.5 Implement background music
    - Play background music loop when game starts
    - _Requirements: 7.4_
  
  - [ ] 10.6 Implement graceful degradation
    - Continue game operation without audio on initialization failures
    - Log audio errors but don't block game execution
    - _Requirements: 7.5_

- [ ] 11. Implement Persistence System
  - [ ] 11.1 Create Persistence Manager component
    - Retrieve high score on game load
    - _Requirements: 9.3_
  
  - [ ] 11.2 Implement high score comparison
    - Compare final score to high score on game over
    - _Requirements: 9.1_
  
  - [ ] 11.3 Implement high score saving
    - Save new high score when final score exceeds stored value
    - _Requirements: 9.2_
  
  - [ ] 11.4 Implement high score display
    - Display high score alongside final score during Game Over
    - _Requirements: 9.4_
  
  - [ ] 11.5 Implement placeholder values
    - Display 0 for score and N/A for high score when unavailable
    - _Requirements: 9.5_
  
  - [ ]* 11.6 Write property test for high score preservation
    - **Property 4: High score preservation**
    - **Validates: Requirements 9.1, 9.2, 9.3**
    - Test that high score updates when final score exceeds it
    - Test persistence across reload cycles

- [ ] 12. Implement Responsive Layout System
  - [ ] 12.1 Create Layout Manager component
    - Handle window resize events
    - Adjust game area dimensions proportionally
    - _Requirements: 10.1, 10.2_
  
  - [ ] 12.2 Implement aspect ratio maintenance
    - Maintain 4:3 aspect ratio on load
    - Allow deviation on limited screen real estate
    - _Requirements: 10.3, 10.4_
  
  - [ ] 12.3 Implement maximum dimension constraints
    - Prevent game area from exceeding 1024x768 pixels
    - _Requirements: 10.5_
  
  - [ ] 12.4 Implement element repositioning
    - Reposition all game elements proportionally on layout change
    - Ensure ghost, pipes, and clouds stay within bounds

- [ ] 13. Implement Difficulty Progression System
  - [ ] 13.1 Create Difficulty Manager component
    - Track score milestones for difficulty increases
    - _Requirements: 11.1, 11.2, 11.3_
  
  - [ ] 13.2 Implement speed progression
    - Increase pipe speed by 10% every 5 points
    - _Requirements: 11.1_
  
  - [ ] 13.3 Implement gap difficulty
    - Decrease gap size by 15% every 10 points
    - _Requirements: 11.2_
  
  - [ ] 13.4 Implement gravity progression
    - Increase gravity by 5% every 15 points
    - _Requirements: 11.3_
  
  - [ ] 13.5 Implement speed cap
    - Cap maximum pipe speed at 300 pixels per second
    - _Requirements: 11.4_
  
  - [ ]* 13.6 Write property test for difficulty progression monotonicity
    - **Property 7: Difficulty progression monotonicity**
    - **Validates: Requirements 11.1, 11.2, 11.3, 11.4**
    - Test that difficulty parameters increase monotonically with score
    - Verify maximum speed cap is never exceeded

- [ ] 14. Implement Reset Functionality
  - [ ] 14.1 Create Reset Manager component
    - Move ghost to starting position
    - Remove all active pipes
    - Reset score to 0
    - Reset difficulty to initial values
    - _Requirements: 13.1, 13.2, 13.3, 13.4_
  
  - [ ] 14.2 Implement cloud reset
    - Reposition all clouds to initial state
    - Generate new random positions
    - _Requirements: 13.5_
  
  - [ ] 14.3 Implement state transition after reset
    - Transition from Game Over to Start state
    - Clear collision time and other state-specific data
    - Prepare for new game session
  
  - [ ]* 14.4 Write property test for reset state consistency
    - **Property 8: Reset state consistency**
    - **Validates: Requirements 13.1, 13.2, 13.3, 13.4, 13.5**
    - Test that reset returns to initial conditions
    - Generate random game states and verify full reset

- [ ] 15. Implement Performance Monitoring System
  - [ ] 15.1 Create Performance Monitor component
    - Track frame rate (target: 60 FPS)
    - Monitor memory usage (alert at 100MB)
    - Process input events before physics calculations each frame
    - _Requirements: 14.1, 14.5_
  
  - [ ] 15.2 Implement visual effects reduction
    - Reduce visual effects when frame rate drops below 30 FPS
    - _Requirements: 14.2_
  
  - [ ] 15.3 Implement memory cleanup
    - Trigger cleanup when memory exceeds 100MB
    - Execute escalating fallback measures (visual effects first)
    - _Requirements: 14.3, 14.4_
  
  - [ ] 15.4 Implement input event ordering
    - Process all input events before physics calculations
    - Ensure deterministic behavior for input sequences

- [ ] 16. Integration and Testing
  - [ ] 16.1 Wire all components together
    - Connect Input Handler to Physics Engine and Game Manager
    - Connect Collision Detector to Game Manager for Game Over state
    - Connect Score Manager to Game Manager for score updates
    - _Requirements: All requirements via component integration_
  
  - [ ]* 16.2 Write integration tests for full game sessions
    - Test Start → Playing → Game Over sequence
    - Verify all components coordinate correctly
    - Test score progression and difficulty changes
  
  - [ ]* 16.3 Write integration tests for state transitions
    - Test Start → Playing → Game Over → Start sequence
    - Verify resource cleanup between transitions
    - Test multiple consecutive transitions
  
  - [ ]* 16.4 Write integration tests for window resize
    - Test resize events during gameplay
    - Verify element repositioning
    - Test maximum dimension enforcement

- [ ] 17. Final Checkpoint - Ensure all tests pass
  - Run all unit tests, property-based tests, and integration tests
  - Verify 60 FPS performance target
  - Confirm memory usage stays under 100MB
  - Ensure all 15 requirements are fully implemented and tested
  - Ask the user if questions arise.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["1.5", "2", "3", "4", "5", "6"] },
    { "id": 2, "tasks": ["7", "8", "9", "10", "11", "12"] },
    { "id": 3, "tasks": ["13", "14", "15"] },
    { "id": 4, "tasks": ["16"] },
    { "id": 5, "tasks": ["17"] }
  ]
}
```

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Core implementation tasks should not be marked optional
- Property-based tests validate universal correctness properties
- Integration tests verify end-to-end flows
- Each component is tested individually before integration
- Performance tests run after implementation to verify targets

## Property-Based Testing Summary

The following properties are validated through property-based tests:

| Property | Description | Requirements Validated |
|----------|-------------|----------------------|
| 1 | Frame-rate independent movement | 12.3, 12.4 |
| 2 | Pipe spacing consistency | 2.1 |
| 3 | Score accuracy | 4.1, 4.2 |
| 4 | High score preservation | 9.1, 9.2, 9.3 |
| 5 | State transition validity | 5.1, 5.2, 5.3, 5.4 |
| 6 | Collision detection completeness | 3.1, 3.2, 3.3, 3.4, 3.5 |
| 7 | Difficulty progression monotonicity | 11.1, 11.2, 11.3, 11.4 |
| 8 | Reset state consistency | 13.1, 13.2, 13.3, 13.4, 13.5 |

## Test Files Structure

```
tests/
├── physics/
│   ├── position_update.test.ts
│   └── delta_time_capping.test.ts
├── pipes/
│   ├── spawner.test.ts
│   └── movement.test.ts
├── clouds/
│   ├── manager.test.ts
│   └── recycling.test.ts
├── collision/
│   ├── pipe_detection.test.ts
│   ├── boundary_detection.test.ts
│   └── failsafe.test.ts
├── scoring/
│   ├── increment.test.ts
│   ├── preservation.test.ts
│   └── reset.test.ts
├── rendering/
│   ├── ghost.test.ts
│   ├── pipes.test.ts
│   ├── clouds.test.ts
│   └── ui.test.ts
├── audio/
│   ├── initialization.test.ts
│   ├── effects.test.ts
│   └── degradation.test.ts
├── persistence/
│   ├── storage.test.ts
│   ├── retrieval.test.ts
│   └── fallback.test.ts
├── layout/
│   ├── resize.test.ts
│   └── dimensions.test.ts
├── difficulty/
│   ├── progression.test.ts
│   └── cap.test.ts
├── reset/
│   ├── state_reset.test.ts
│   └── resource_cleanup.test.ts
├── performance/
│   ├── frame_rate.test.ts
│   ├── memory.test.ts
│   └── input_ordering.test.ts
├── integration/
│   ├── full_game.test.ts
│   ├── state_transitions.test.ts
│   └── window_resize.test.ts
└── property/
    ├── frame_rate_independence.test.ts
    ├── pipe_spacing.test.ts
    ├── score_accuracy.test.ts
    ├── high_score_preservation.test.ts
    ├── collision_completeness.test.ts
    ├── difficulty_monotonicity.test.ts
    └── reset_consistency.test.ts
```