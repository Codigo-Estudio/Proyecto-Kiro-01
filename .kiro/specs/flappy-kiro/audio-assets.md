# Audio Assets Specification

## Sound Design Overview
All audio assets are optimized for browser playback with graceful degradation support.

## Sound Effects

### Flapping Sound
- **Filename**: jump.wav
- **Duration**: 0.1 seconds
- **Format**: WAV (lossless)
- **Sample Rate**: 44.1 kHz
- **Channels**: Stereo
- **Description**: Short whistle tone
- **Frequency Profile**: Start: 800 Hz, End: 1200 Hz
- **Volume**: -12 dB
- **Pitch Envelope**: Upward slide

### Score Sound
- **Filename**: score.wav
- **Duration**: 0.2 seconds
- **Format**: WAV (lossless)
- **Sample Rate**: 44.1 kHz
- **Channels**: Stereo
- **Description**: Pleasant jingle
- **Frequency Profile**: First note: 1200 Hz (0.05s), Second note: 1800 Hz (0.15s)
- **Volume**: -6 dB
- **Note**: Two-note ascending arpeggio

### Collision Sound
- **Filename**: gameover.wav
- **Duration**: 0.3 seconds
- **Format**: WAV (lossless)
- **Sample Rate**: 44.1 kHz
- **Channels**: Stereo
- **Description**: Thud sound
- **Frequency Profile**: Start: 100 Hz, End: 20 Hz
- **Volume**: -3 dB
- **Envelope**: Attack (0.01s), Decay (0.29s)
