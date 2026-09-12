# Ghosty Character Specifications

## Sprite Dimensions
- **Width**: 32 pixels
- **Height**: 32 pixels
- **Format**: PNG with transparency support
- **Animation Frame Rate**: 12 frames per second

## Animation States

### Idle State
- **Duration**: Continuous loop
- **Frame Count**: 4 frames
- **Description**: Gentle floating motion with slight rotation

### Fluttering State
- **Duration**: Continuous loop during gameplay
- **Frame Count**: 8 frames
- **Description**: Active wing movement with body control

### Death State
- **Duration**: 500ms total animation
- **Frame Count**: 12 frames
- **Description**: Rotation and fade-out effect

## Collision Specification
- **Shape**: Circular collision
- **Radius**: 12 pixels
- **Center Point**: (x + 16, y + 16) relative to sprite position
