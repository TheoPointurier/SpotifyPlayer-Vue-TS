<!-- src/components/HomePage.vue -->
<template>
  <div class="home">
    <div class="intro">
      <h2>La super playlist</h2>
      <p>Voici ma playlist confectionnée avec soin, à écouter à tout moment de la journée pour une ambiance toujours
        "Fresh"</p>
    </div>
    <div class="hero" v-if="playlist">
      <img class="cover" :src="playlist.images?.[0]?.url" alt="" />
      <div class="actions">
        <button class="btn" :class="{ active: isShuffling }" @click="onToggleShuffle">
          🔀 {{ isShuffling ? 'Shuffle activé' : 'Activer shuffle' }}
        </button>
      </div>
      <div class="meta">
        <span class="badge">PLAYLIST</span>
        <h1 class="title">{{ playlist.name }}</h1>
        <p class="desc" v-html="playlist.description || ''"></p>
        <div class="byline">
          <span class="owner">{{ playlist.owner?.display_name }}</span>
          <span class="dot">•</span>
          <span class="count">{{ playlist.tracks?.total }} titres</span>
        </div>
      </div>
    </div>

    <div class="panel">
      <div class="panel-header">
        <span>#</span>
        <span>Titre</span>
        <span>Album</span>
        <span>Durée</span>
      </div>

      <div class="tracks" v-if="tracks.length">
        <button v-for="(item, i) in tracks" :key="item.track?.id || i" class="row"
          :class="{ active: selectedTrackId === item.track?.id }" @click="onSelect(item.track)">
          <span class="index">{{ i + 1 }}</span>
          <span class="title-artist">
            <span class="t">{{ item.track?.name }}</span>
            <span class="a">{{ artists(item.track) }}</span>
          </span>
          <span class="album">{{ item.track?.album?.name }}</span>
          <span class="duration">{{ mmss(item.track?.duration_ms) }}</span>
        </button>
      </div>

      <div class="empty" v-else-if="!loading && !error">
        Aucune piste trouvée.
      </div>

      <div class="state" v-if="loading">Chargement…</div>
      <div class="state error" v-if="error">{{ error }}</div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, computed } from 'vue'
import { getTheOnePlaylist } from '../services/spotifyService'
import { playTrack } from '../services/spotifyService'
import { toggleShuffle } from '../services/spotifyPlayerService'

// 👉 Id playlist : mets celui que tu veux, ou récupère-le via route params
const PLAYLIST_ID = '35x9ZgrBLJI6ZQAkO17fFh'

type SpotifyTrack = {
  id: string
  uri: string
  name: string
  duration_ms: number
  album?: { name?: string }
  artists?: { name: string }[]
}

type PlaylistItem = { track: SpotifyTrack }
type Playlist = {
  name: string
  description?: string
  images?: { url: string }[]
  owner?: { display_name?: string }
  tracks: { items: PlaylistItem[]; total: number }
}

const isShuffling = ref(false)
const playlist = ref<Playlist | null>(null)
const loading = ref(true)
const error = ref<string | null>(null)
const selectedTrackId = ref<string | null>(null)

const tracks = computed(() => playlist.value?.tracks?.items ?? [])

const queue = computed<string[]>(() =>
  tracks.value
    .map(it => it.track?.uri)
    .filter((u): u is string => Boolean(u))
)

function artists(track?: SpotifyTrack) {
  return track?.artists?.map(a => a.name).join(', ') || ''
}

function mmss(ms?: number) {
  if (!ms && ms !== 0) return ''
  const total = Math.floor(ms / 1000)
  const m = Math.floor(total / 60)
  const s = total % 60
  return `${m}:${s.toString().padStart(2, '0')}`
}

async function onSelect(track?: SpotifyTrack) {
  if (!track?.id || !track?.uri) return
  selectedTrackId.value = track.id
  try {
    await playTrack(track.uri, queue.value)
    console.log('Selected track:', track)
  } catch (e) {
    console.error(e)
    error.value = (e as Error).message
  }
}

async function fetchShuffleState() {
  const res = await fetch('/api/spotify/proxy', {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      method: 'GET',
      path: '/v1/me/player',
    }),
  })
  // 204 possible si pas de device actif
  if (res.status === 204) return
  if (!res.ok) return
  const data = await res.json()
  if (typeof data?.shuffle_state === 'boolean') {
    isShuffling.value = data.shuffle_state
  }
}

async function onToggleShuffle() {
  const previous = isShuffling.value
  try {
    // optimiste
    isShuffling.value = !previous
    // appel ta fonction : elle renvoie le nouvel état effectif
    isShuffling.value = await toggleShuffle(previous)
  } catch (e) {
    // rollback visuel en cas d’erreur
    isShuffling.value = previous
    console.error(e)
  }
}

onMounted(async () => {
  try {
    loading.value = true
    error.value = null
    const data = await getTheOnePlaylist(PLAYLIST_ID)
    playlist.value = data
    await fetchShuffleState()
  } catch (error: unknown) {
    return error instanceof Error ? error.message : String(error)
  } finally {
    loading.value = false
  }
})
</script>

<style scoped>
.home {
  display: grid;
  gap: 24px;
  padding: 24px;
  color: var(--spotify-white, #fff);
  background: linear-gradient(180deg, rgba(0, 0, 0, 0.6), rgba(0, 0, 0, 0.2));
  max-height: 100%;
  overflow: hidden;
}

.intro {
  display: flex;
  flex-direction: column;
  justify-content: center;
  max-width: 500px;
  height: 20rem;
  margin: 0 auto;
  line-height: 1.4;
  opacity: 0.9;
}

.intro h2 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
}

/* Hero */
.hero {
  display: grid;
  grid-template-columns: 180px 1fr;
  gap: 20px;
  align-items: center;
}

.actions {
  margin-top: 10px;
  display: flex;
  gap: 10px;
}

.btn {
  padding: 8px 12px;
  border-radius: 999px;
  cursor: pointer;
  background: rgba(255, 255, 255, .08);
  border: 1px solid rgba(255, 255, 255, .12);
  color: #fff;
}

.btn:hover {
  background: rgba(255, 255, 255, .14);
}

.btn.active {
  background: rgba(30, 215, 96, .22);
  border-color: rgba(30, 215, 96, .5);
}


.cover {
  width: 180px;
  height: 180px;
  object-fit: cover;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, .35);
}

.meta {
  display: grid;
  gap: 8px;
}

.badge {
  font-size: 12px;
  letter-spacing: .12em;
  opacity: .8;
}

.title {
  font-size: 42px;
  line-height: 1.1;
  margin: 0;
}

.desc :deep(a) {
  color: inherit;
  text-decoration: underline;
}

.desc {
  opacity: .9;
}

.byline {
  display: flex;
  gap: 8px;
  align-items: center;
  opacity: .8;
}

.dot {
  opacity: .6;
}

/* Panel liste */
.panel {
  background: rgba(255, 255, 255, .04);
  border-radius: 14px;
  overflow: hidden;
  backdrop-filter: blur(8px);
  border: 1px solid rgba(255, 255, 255, .06);
}

.panel-header {
  display: grid;
  grid-template-columns: 40px 1fr 1fr 70px;
  padding: 12px 16px;
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: .08em;
  color: rgba(255, 255, 255, .7);
  border-bottom: 1px solid rgba(255, 255, 255, .08);
}

/* Scrollable tracklist */
.tracks {
  max-height: 52vh;
  overflow: auto;
}

.row {
  display: grid;
  grid-template-columns: 40px 1fr 1fr 70px;
  width: 100%;
  text-align: left;
  gap: 8px;
  padding: 10px 16px;
  background: transparent;
  color: inherit;
  border: 0;
  cursor: pointer;
}

.row:hover {
  background: rgba(255, 255, 255, .06);
}

.row.active {
  background: rgba(30, 215, 96, .18);
}

.index {
  opacity: .7;
}

.title-artist {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.title-artist .t {
  font-weight: 600;
}

.title-artist .a {
  opacity: .8;
  font-size: 0.92rem;
}

.album {
  opacity: .85;
}

.duration {
  opacity: .75;
  text-align: right;
}

.state,
.empty,
.error {
  padding: 18px 16px;
}

.error {
  color: #ffb4b4;
}
</style>
