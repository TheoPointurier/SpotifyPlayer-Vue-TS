<script lang="ts">
import { defineComponent, ref } from 'vue';

export default defineComponent({
  name: 'FullPlayerModal',
  props: {
    track: {
      type: Object as () => Spotify.PlaybackState['track_window']['current_track'] | null,
      required: true,
    },
    progress: {
      type: Number,
      required: true,
    },
    duration: {
      type: Number,
      required: true,
    },
    isPlaying: {
      type: Boolean,
      required: true,
    },
    isShuffling: {
      type: Boolean,
      required: true,
    },
    isRepeating: {
      type: String,
      required: true,
    },
    volume: {
      type: Number,
      required: true,
    },
  },
  emits: [
    'toggle-play-pause',
    'next-track',
    'previous-track',
    'toggle-shuffle',
    'toggle-repeat',
    'update-progress',
    'update-volume',
    'close',
  ],
  setup(props, { emit }) {
    const isDragging = ref<boolean>(false);

    const formatDuration = (durationMs: number): string => {
      const totalSeconds = Math.floor(durationMs / 1000);
      const minutes = Math.floor(totalSeconds / 60);
      const seconds = totalSeconds % 60;
      const formattedMinutes = String(minutes).padStart(2, '0');
      const formattedSeconds = String(seconds).padStart(2, '0');
      return `${formattedMinutes}:${formattedSeconds}`;
    };

    const handleProgressDragStart = () => {
      isDragging.value = true;
      emit('update-progress', { isDragging: true }); // Informer le parent que le drag commence
    };

    const handleProgressDrag = (event: Event) => {
      if (!isDragging.value || !props.duration) return;
      const input = event.target as HTMLInputElement;
      const newProgress = (Number.parseFloat(input.value) / 100) * props.duration;
      // Ne pas émettre ici, attendre la fin du drag
    };

    const handleProgressDragEnd = async (event: Event) => {
      if (!props.duration) return;
      const input = event.target as HTMLInputElement;
      const newProgress = (Number.parseFloat(input.value) / 100) * props.duration;
      emit('update-progress', { newProgress, isDragging: false }); // Émettre la nouvelle position et l'état du drag
      isDragging.value = false;
    };

    return {
      formatDuration,
      handleProgressDragStart,
      handleProgressDrag,
      handleProgressDragEnd,
    };
  },
});
</script>

<template>
  <div class="full-player-modal">
    <button class="close-btn" @click.stop="$emit('close')">▼</button>
    <div class="track-details">
      <img v-if="track" :src="track.album.images[0]?.url" alt="Album cover" class="large-album-art" />
      <h2>{{ track?.name || 'No track' }}</h2>
      <p>{{ track?.artists[0]?.name || '' }}</p>
    </div>
    <div class="progress-container">
      <span class="progress-time">{{ formatDuration(progress) }}</span>
      <div class="progress-bar">
        <input type="range" min="0" max="100" step="1" :value="duration ? (progress / duration) * 100 : 0"
          @mousedown="handleProgressDragStart" @input="handleProgressDrag" @mouseup="handleProgressDragEnd"
          @touchstart="handleProgressDragStart" @touchmove="handleProgressDrag" @touchend="handleProgressDragEnd"
          class="progress-slider" aria-label="Progression de la lecture"
          :aria-valuenow="duration ? (progress / duration) * 100 : 0" aria-valuemin="0" aria-valuemax="100"
          :aria-valuetext="`${formatDuration(progress)} de ${formatDuration(duration)}`" />
        <div class="progress-fill" :style="{ width: `${duration ? (progress / duration) * 100 : 0}%` }"></div>
      </div>
      <span class="progress-time">{{ track ? formatDuration(track.duration_ms) : '00:00' }}</span>
    </div>
    <div class="controls">
      <button class="control-button control-button-shuffle" @click="$emit('toggle-shuffle')"
        :class="{ active: isShuffling }" title="Shuffle">
        <i class="fas fa-shuffle"></i>
      </button>
      <button class="control-button control-button-previous" @click="$emit('previous-track')" title="Previous">
        <i class="fas fa-backward"></i>
      </button>
      <button class="control-button play-pause" @click="$emit('toggle-play-pause')"
        :title="isPlaying ? 'Pause' : 'Play'">
        <i :class="isPlaying ? 'fas fa-pause' : 'fas fa-play'"></i>
      </button>
      <button class="control-button control-button-next" @click="$emit('next-track')" title="Next">
        <i class="fas fa-forward"></i>
      </button>
      <button class="control-button control-button-repeat" @click="$emit('toggle-repeat')"
        :class="{ active: isRepeating !== 'off' }" :title="`Repeat: ${isRepeating}`">
        <i class="fas fa-redo"></i>
        <span v-if="isRepeating === 'track'" class="repeat-mode">1</span>
      </button>
    </div>
    <div class="other-controls">
      <button class="control-button" title="Volume">
        <i class="fas fa-volume-up"></i>
      </button>
      <input type="range" min="0" max="1" step="0.01" :value="volume" @input="$emit('update-volume')"
        aria-label="Volume" :aria-valuenow="volume * 100" aria-valuemin="0" aria-valuemax="100"
        :aria-valuetext="`${Math.round(volume * 100)}%`" />
    </div>
  </div>
</template>

<style scoped>
/* Styles inchangés */
.full-player-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #121212;
  color: var(--spotify-white);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px;
  z-index: 1000;
}

.close-btn {
  position: absolute;
  top: 20px;
  left: 20px;
  background: none;
  border: none;
  color: var(--spotify-white);
  font-size: 24px;
}

.large-album-art {
  width: 200px;
  height: 200px;
  border-radius: 8px;
  object-fit: cover;
  margin: 20px 0;
}

.track-details {
  text-align: center;
  margin-bottom: 20px;
}

.track-details h2 {
  font-size: 24px;
  margin: 10px 0;
}

.track-details p {
  font-size: 16px;
  color: var(--spotify-light-grey);
}

.progress-container {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 20px 0;
}

.progress-time {
  font-size: 12px;
  color: var(--spotify-light-grey);
}

.progress-bar {
  flex: 1;
  height: 4px;
  background-color: var(--spotify-light-grey);
  border-radius: 2px;
  overflow: hidden;
  position: relative;
}

.progress-bar:hover {
  height: 6px;
}

.progress-fill {
  height: 100%;
  background-color: var(--spotify-green);
  transition: width 0.1s linear;
}

.progress-slider {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
}

.progress-bar:hover .progress-fill {
  background-color: #1ed760;
}

.controls {
  display: flex;
  margin: 20px 0;
}

.control-button {
  background-color: transparent;
  border: none;
  color: var(--spotify-white);
  font-size: 20px;
  cursor: pointer;
  transition: color 0.2s ease;
}

.control-button:hover {
  color: var(--spotify-green);
}

.control-button.active {
  color: var(--spotify-green);
}

.control-button-repeat {
  position: relative;
}

.play-pause {
  background-color: var(--spotify-green);
  width: 48px;
  height: 48px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.play-pause:hover {
  background-color: #1ed760;
}

.repeat-mode {
  font-size: 10px;
  position: absolute;
  top: 0;
  left: 1;
}

.other-controls {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 8px;
  justify-content: center;
  margin-right: 1rem;
}

.other-controls input[type="range"] {
  width: 100px;
}
</style>