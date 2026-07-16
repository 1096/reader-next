<template>
  <div class="reader-source" :style="{ background: theme.popup, color: theme.fontColor }">
    <div class="source-header">
      <div class="header-left">
        <h3>切换书源</h3>
        <span class="source-count" v-if="preparedResults.length">{{ preparedResults.length }} 个结果</span>
      </div>
      <button class="close-btn" @click="store.closePanel()">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18 6 6 18M6 6l12 12" /></svg>
      </button>
    </div>
    
    <div class="source-list" ref="listRef">
      <!-- Current Source Header -->
      <div class="section-label" v-if="currentSource">当前书源</div>
      <div v-if="currentSource" class="source-item current">
        <div class="source-name">{{ currentSource.originName || currentSource.origin }}</div>
        <div class="source-url">{{ currentSource.origin }}</div>
      </div>

      <div v-if="store.book" class="book-brief">
        <div class="book-brief-cover">
          <img v-if="store.book.coverUrl" :src="store.book.coverUrl" :alt="store.book.name">
          <div v-else class="book-brief-placeholder">{{ store.book.name.slice(0, 1) }}</div>
        </div>
        <div class="book-brief-main">
          <div class="book-brief-title">{{ store.book.name }}</div>
          <div class="book-brief-meta">{{ store.book.author || '未知作者' }}</div>
          <div class="book-brief-meta" v-if="store.currentChapter?.title">当前章节：{{ store.currentChapter.title }}</div>
          <div class="book-brief-meta" v-if="store.book.latestChapterTitle">最新章节：{{ store.book.latestChapterTitle }}</div>
          <div class="book-brief-intro" v-if="store.book.intro">{{ store.book.intro }}</div>
        </div>
      </div>

      <div class="section-label">其他可用源</div>

      <div class="search-tools" v-if="store.book">
        <label class="tool-item compact">
          <span class="tool-label">并发</span>
          <input
            type="number"
            min="4"
            max="128"
            step="1"
            v-model.number="concurrentCount"
            :disabled="searching"
          >
        </label>
        <label class="tool-item compact">
          <span class="tool-label">扫描上限</span>
          <input
            type="number"
            min="20"
            max="1000"
            step="10"
            v-model.number="searchSize"
            :disabled="searching"
          >
        </label>
        <div class="tool-actions">
          <button class="refresh-btn" :disabled="searching" @click="startSearch(true)">重新搜索</button>
          <button v-if="searching || loadingMore" class="cancel-btn" type="button" @click="cancelSearch">取消搜索</button>
        </div>
      </div>

      <div v-if="store.book" class="search-state-banner" :class="{
        running: searching || loadingMore,
        idle: !searching && !loadingMore && hasMoreSources,
        done: !searching && !loadingMore && !hasMoreSources,
      }">
        <span class="search-state-dot"></span>
        <div class="search-state-text">
          <div class="search-state-title">
            {{ searching || loadingMore
              ? (isFollower ? '后台搜索中' : '正在后台搜索书源')
              : (hasMoreSources ? '搜索已暂停' : '搜索已完成') }}
          </div>
          <div class="search-state-subtitle">
            {{ searching || loadingMore
              ? '关闭书源面板不会中断搜索，返回后可继续查看结果。'
              : (hasMoreSources
                ? '当前结果可继续追加，点击加载更多或重新搜索。'
                : '已完成全部匹配，可直接切换结果书源。') }}
          </div>
        </div>
      </div>

      <div v-if="showProgress" class="progress-wrap">
        <div class="progress-head">
          <span>{{ isFollower ? '由其他页面搜索中' : (searching ? '正在搜索书源' : '搜索已完成') }}</span>
          <span>{{ progress.processed }}/{{ progress.total }}，命中 {{ progress.matched }}</span>
        </div>
        <div class="progress-track">
          <div class="progress-bar" :style="{ width: `${progressPercent}%` }"></div>
        </div>
      </div>
      
      <div v-if="searching && !preparedResults.length" class="loading">
        <div class="spinner"></div>
        正在全网搜索同名书籍...
      </div>
      
      <div v-else-if="!preparedResults.length" class="empty">未找到其他书源</div>
      
      <div
        v-else
        v-for="item in preparedResults"
        :key="item.book.bookUrl + item.book.origin"
        class="source-item"
        :class="{ selected: selectedCandidate?.book.bookUrl === item.book.bookUrl && selectedCandidate?.book.origin === item.book.origin }"
        @click="selectCandidate(item)"
      >
        <div class="source-main">
          <div class="source-name-row">
            <span class="source-name">{{ item.book.origin }}</span>
            <span class="source-tag" v-if="item.book.kind">{{ item.book.kind }}</span>
            <span v-if="item.sameLatest" class="compare-badge good">最新章节一致</span>
            <span v-else-if="item.book.lastChapter" class="compare-badge">章节不同</span>
          </div>
          <div class="source-book-name" v-if="item.book.name">{{ item.book.name }}</div>
          <div class="source-author">{{ item.book.author }}</div>
          <div class="source-intro" v-if="item.book.intro">{{ item.book.intro }}</div>
          <div class="source-chapter" v-if="item.book.lastChapter">最新: {{ item.book.lastChapter }}</div>
          <div class="source-update" v-if="item.book.updateTime">更新时间: {{ item.book.updateTime }}</div>
          <div class="source-compare-line">
            <span v-if="item.sameName" class="compare-text">书名匹配</span>
            <span v-if="item.sameAuthor" class="compare-text">作者匹配</span>
            <span v-if="item.chapterHint" class="compare-text strong">{{ item.chapterHint }}</span>
          </div>
        </div>
        <div class="source-action">
          <button class="switch-btn" @click.stop="handleSwitch(item.book)">切换</button>
        </div>
      </div>

      <div v-if="searching && preparedResults.length" class="inline-loading">
        <div class="spinner small"></div>
        继续搜索其他书源...
      </div>

      <div v-if="selectedCandidate" class="compare-panel">
        <div class="compare-header">
          <h4>书源对照</h4>
          <button class="switch-btn primary" :disabled="store.loading" @click="handleSwitch(selectedCandidate.book)">切换到此书源</button>
        </div>
        <div class="compare-grid">
          <div class="compare-card">
            <div class="compare-title">当前</div>
            <div class="compare-name">{{ store.book?.name }}</div>
            <div class="compare-meta">{{ store.book?.author || '未知作者' }}</div>
            <div class="compare-line">书源：{{ store.book?.originName || store.book?.origin }}</div>
            <div class="compare-line">当前章节：{{ store.currentChapter?.title || '未知' }}</div>
            <div class="compare-line">最新章节：{{ store.book?.latestChapterTitle || '未知' }}</div>
          </div>
          <div class="compare-card highlight">
            <div class="compare-title">目标</div>
            <div class="compare-name">{{ candidatePreview?.name || selectedCandidate.book.name }}</div>
            <div class="compare-meta">{{ candidatePreview?.author || selectedCandidate.book.author || '未知作者' }}</div>
            <div class="compare-line">书源：{{ candidatePreview?.originName || selectedCandidate.book.origin }}</div>
            <div class="compare-line">最新章节：{{ candidatePreview?.latestChapterTitle || selectedCandidate.book.lastChapter || '未知' }}</div>
            <div class="compare-line compare-intro" v-if="candidatePreview?.intro || selectedCandidate.book.intro">{{ candidatePreview?.intro || selectedCandidate.book.intro }}</div>
          </div>
        </div>
      </div>

      <div v-if="hasMoreSources && !searching" class="load-more-wrap">
        <button class="load-more-btn" :disabled="loadingMore" @click="loadMoreSources">
          {{ loadingMore ? '加载中...' : '加载更多' }}
        </button>
      </div>
    </div>

    <!-- Switching Overlay -->
    <Transition name="fade">
      <div v-if="store.loading" class="switch-overlay" :style="{ background: theme.popup }">
        <div class="spinner"></div>
        <p>正在切换书源，请稍候...</p>
      </div>
    </Transition>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { useReaderStore } from '../../stores/reader'
import { useAppStore } from '../../stores/app'
import { getAvailableBookSourceSSE } from '../../api/search'
import { getBookInfo } from '../../api/bookshelf'
import type { Book, SearchBook } from '../../types'

const SEARCH_PREF_KEY = 'reader-source-search-pref-v1'
const SEARCH_STATE_PREFIX = 'reader-source-search-state:'
const SEARCH_LOCK_PREFIX = 'reader-source-search-lock:'
const SEARCH_LOCK_TTL = 12000
const SEARCH_LOCK_HEARTBEAT = 4000
const SEARCH_FOLLOWER_CHECK_INTERVAL = 2000
const SEARCH_FOLLOWER_STALE_MS = SEARCH_LOCK_TTL + 3000
const SEARCH_STATE_MAX_AGE = 24 * 60 * 60 * 1000

type SearchProgress = {
  processed: number
  total: number
  matched: number
}

type PersistedSearchState = {
  results: SearchBook[]
  lastIndex: number
  hasMoreSources: boolean
  progress: SearchProgress
  concurrentCount: number
  searchSize: number
  updatedAt: number
}

type SearchChannelMessage =
  | { type: 'start'; tabId: string }
  | { type: 'snapshot'; tabId: string; payload: AvailableSourceSSEPayload }
  | { type: 'chunk'; tabId: string; payload: AvailableSourceSSEPayload }
  | { type: 'progress'; tabId: string; payload: AvailableSourceSSEPayload }
  | { type: 'end'; tabId: string; payload: AvailableSourceSSEPayload }
  | { type: 'cancel'; tabId: string }
  | { type: 'error'; tabId: string }
  | { type: 'needSnapshot'; tabId: string }
  | { type: 'request-load-more'; tabId: string }

type SearchLock = {
  owner: string
  expiresAt: number
}

type CandidateItem = {
  book: SearchBook
  sameName: boolean
  sameAuthor: boolean
  sameLatest: boolean
  chapterHint: string
  score: number
}

const store = useReaderStore()
const appStore = useAppStore()
const theme = computed(() => store.currentTheme)
const searching = ref(false)
const loadingMore = ref(false)
const results = ref<SearchBook[]>([])
const lastIndex = ref(-1)
const hasMoreSources = ref(true)
const selectedCandidate = ref<CandidateItem | null>(null)
const candidatePreview = ref<Book | null>(null)
const progress = ref<SearchProgress>({ processed: 0, total: 0, matched: 0 })
const tabId = `tab-${Math.random().toString(36).slice(2)}-${Date.now()}`
const isLeader = ref(false)
const isFollower = ref(false)
const concurrentCount = ref(24)
const searchSize = ref(200)
let heartbeatTimer: number | null = null
let followerMonitorTimer: number | null = null
let lastLeaderSignalAt = 0
let channel: BroadcastChannel | null = null
let availableSourceSSE: EventSource | null = null

const userKey = computed(() => appStore.userInfo?.username || 'guest')
const searchBookKey = computed(() => {
  if (!store.book) return ''
  return `${normalizeText(store.book.name)}::${normalizeAuthorText(store.book.author)}`
})
const stateStorageKey = computed(() => {
  if (!searchBookKey.value) return ''
  return `${SEARCH_STATE_PREFIX}${userKey.value}:${searchBookKey.value}`
})
const lockStorageKey = computed(() => {
  if (!searchBookKey.value) return ''
  return `${SEARCH_LOCK_PREFIX}${userKey.value}:${searchBookKey.value}`
})
const channelName = computed(() => {
  if (!searchBookKey.value) return ''
  return `reader-source-search:${userKey.value}:${searchBookKey.value}`
})

const progressPercent = computed(() => {
  if (!progress.value.total) return 0
  return Math.min(100, Math.max(0, Math.round((progress.value.processed / progress.value.total) * 100)))
})
const showProgress = computed(() => searching.value || progress.value.total > 0)

const currentSource = computed(() => {
  if (!store.book) return null
  return {
    origin: store.book.origin,
    originName: store.book.originName
  }
})

const isPanelActive = computed(() => store.activePanel === 'source')

function normalizeText(value?: string) {
  return (value || '')
    .replace(/\s+/g, '')
    .replace(/[：:,.，。！？!?\-—_()（）【】\[\]<>《》'"“”‘’]/g, '')
    .toLowerCase()
}

function normalizeAuthorText(value?: string) {
  return normalizeText(value).replace(/^作者/, '')
}

const preparedResults = computed<CandidateItem[]>(() => {
  if (!store.book) return []
  const currentName = normalizeText(store.book.name)
  const currentAuthor = normalizeAuthorText(store.book.author)
  const currentLatest = normalizeText(store.book.latestChapterTitle || store.currentChapter?.title)

  return results.value
    .map((book) => {
      const sameName = normalizeText(book.name) === currentName
      const sameAuthor = normalizeAuthorText(book.author) === currentAuthor
      const sameLatest = !!currentLatest && normalizeText(book.lastChapter) === currentLatest
      const chapterHint = sameLatest
        ? '可无缝续读'
        : (book.lastChapter ? `目标源最新：${book.lastChapter}` : '')
      const score = (sameName ? 3 : 0) + (sameAuthor ? 3 : 0) + (sameLatest ? 4 : 0)
      return { book, sameName, sameAuthor, sameLatest, chapterHint, score }
    })
    .sort((a, b) => b.score - a.score)
})

onMounted(() => {
  restoreSearchPref()
})

onUnmounted(() => {
  closeAvailableSourceSSE()
  stopHeartbeat()
  stopFollowerMonitor()
  releaseLockIfOwned()
  if (channel) {
    channel.close()
    channel = null
  }
})

watch([concurrentCount, searchSize], () => {
  normalizeSearchControls()
  persistSearchPref()
})

watch(
  () => store.activePanel,
  (panel) => {
    if (panel !== 'source') return
    initializeChannel()
    startFollowerMonitor()
    const restored = restorePersistedState()
    if (!restored && !searching.value && !results.value.length) {
      startSearch()
    }
  },
  { immediate: true },
)

watch(
  () => [store.book?.name, store.book?.author],
  () => {
    closeAvailableSourceSSE()
    stopHeartbeat()
    releaseLockIfOwned()
    stopFollowerMonitor()
    if (channel) {
      channel.close()
      channel = null
    }
    isLeader.value = false
    isFollower.value = false
    searching.value = false
    loadingMore.value = false

    if (isPanelActive.value) {
      initializeChannel()
      startFollowerMonitor()
      const restored = restorePersistedState()
      if (!restored) {
        startSearch()
      }
    } else {
      results.value = []
      lastIndex.value = -1
      hasMoreSources.value = true
      selectedCandidate.value = null
      candidatePreview.value = null
      progress.value = { processed: 0, total: 0, matched: 0 }
      clearPersistedState()
    }
  },
)

function startSearch(forceRefresh = false) {
  if (!store.book) return
  normalizeSearchControls()
  closeAvailableSourceSSE()
  persistSearchPref()

  if (forceRefresh) {
    clearPersistedState()
  }

  searching.value = true
  loadingMore.value = false
  if (forceRefresh || !results.value.length) {
    results.value = []
    lastIndex.value = -1
    hasMoreSources.value = true
    progress.value = { processed: 0, total: 0, matched: 0 }
  }
  selectedCandidate.value = null
  candidatePreview.value = null

  if (acquireSearchLock()) {
    isLeader.value = true
    isFollower.value = false
    broadcast({ type: 'start', tabId })
    openAvailableSourceSSE('initial')
  } else {
    isLeader.value = false
    isFollower.value = true
    noteLeaderSignal()
    broadcast({ type: 'needSnapshot', tabId })
  }
}

function cancelSearch() {
  if (!searching.value && !loadingMore.value && !availableSourceSSE) return

  closeAvailableSourceSSE()
  stopHeartbeat()
  stopFollowerMonitor()
  releaseLockIfOwned()

  if (channel) {
    broadcast({ type: 'cancel', tabId })
  }

  isLeader.value = false
  isFollower.value = false
  searching.value = false
  loadingMore.value = false
  persistSearchState()
}

function mergeCandidates(candidates: SearchBook[]) {
  if (!store.book || !candidates.length) return
  const currentBook = store.book
  const currentAuthor = normalizeAuthorText(currentBook.author)
  candidates.forEach((item) => {
    if (item.origin === currentBook.origin) return
    if (currentAuthor && item.author && normalizeAuthorText(item.author) !== currentAuthor) return
    const existed = results.value.some((candidate) =>
      candidate.origin === item.origin || (candidate.bookUrl === item.bookUrl && candidate.origin === item.origin),
    )
    if (!existed) {
      results.value.push(item)
    }
  })
}

type AvailableSourceMode = 'initial' | 'loadMore'
type AvailableSourceSSEPayload = {
  data?: SearchBook[]
  books?: SearchBook[]
  lastIndex?: number
  hasMore?: boolean
  processed?: number
  total?: number
  matched?: number
}

function closeAvailableSourceSSE() {
  if (!availableSourceSSE) return
  availableSourceSSE.close()
  availableSourceSSE = null
}

function parseAvailableSourcePayload(event: MessageEvent): AvailableSourceSSEPayload | null {
  try {
    return JSON.parse(event.data) as AvailableSourceSSEPayload
  } catch (error) {
    console.error('parse getAvailableBookSourceSSE payload failed', error)
    return null
  }
}

function applyAvailableSourcePayload(payload: AvailableSourceSSEPayload | null) {
  if (!payload) return
  const incoming = Array.isArray(payload.data)
    ? payload.data
    : (Array.isArray(payload.books) ? payload.books : [])

  if (typeof payload.lastIndex === 'number') {
    lastIndex.value = payload.lastIndex
  }
  if (typeof payload.hasMore === 'boolean') {
    hasMoreSources.value = payload.hasMore
  }
  if (typeof payload.processed === 'number') {
    progress.value.processed = payload.processed
  }
  if (typeof payload.total === 'number') {
    progress.value.total = payload.total
  }
  if (typeof payload.matched === 'number') {
    progress.value.matched = payload.matched
  }
  mergeCandidates(incoming)
  persistSearchState()

  if (!selectedCandidate.value && preparedResults.value.length) {
    void selectCandidate(preparedResults.value[0])
  }
}

function openAvailableSourceSSE(mode: AvailableSourceMode) {
  if (!store.book) return

  const beforeCount = results.value.length
  const startFromInitial = mode === 'initial' || lastIndex.value < 0
  const stream = getAvailableBookSourceSSE({
    url: store.book.bookUrl,
    name: store.book.name,
    author: store.book.author,
    origin: store.book.origin,
    lastIndex: startFromInitial ? -1 : lastIndex.value,
    concurrentCount: concurrentCount.value,
    searchSize: searchSize.value,
  })
  availableSourceSSE = stream

  stream.onmessage = (event) => {
    if (availableSourceSSE !== stream) return
    const payload = parseAvailableSourcePayload(event)
    applyAvailableSourcePayload(payload)
    broadcast({ type: 'chunk', tabId, payload: payload || {} })
  }

  stream.addEventListener('progress', (event) => {
    if (availableSourceSSE !== stream) return
    const payload = parseAvailableSourcePayload(event as MessageEvent)
    applyAvailableSourcePayload(payload)
    broadcast({ type: 'progress', tabId, payload: payload || {} })
  })

  stream.addEventListener('end', (event) => {
    if (availableSourceSSE !== stream) return
    const payload = parseAvailableSourcePayload(event as MessageEvent)
    applyAvailableSourcePayload(payload)
    broadcast({ type: 'end', tabId, payload: payload || {} })
    finishAvailableSourceSSE(stream, mode, beforeCount)
  })

  stream.onerror = (event) => {
    if (availableSourceSSE !== stream) return
    console.error('getAvailableBookSourceSSE failed', event)
    broadcast({ type: 'error', tabId })
    finishAvailableSourceSSE(stream, mode, beforeCount, true)
  }
}

function finishAvailableSourceSSE(
  stream: EventSource,
  mode: AvailableSourceMode,
  beforeCount: number,
  failed = false,
) {
  if (availableSourceSSE !== stream) return
  stream.close()
  availableSourceSSE = null

  searching.value = false
  loadingMore.value = false

  if (isLeader.value) {
    stopHeartbeat()
    releaseLockIfOwned()
    isLeader.value = false
  }

  if (!selectedCandidate.value && preparedResults.value.length) {
    void selectCandidate(preparedResults.value[0])
  }

  if (failed) {
    searching.value = false
    loadingMore.value = false
    persistSearchState()
    if (mode === 'loadMore') {
      appStore.showToast('加载更多书源失败', 'error')
    }
    return
  }

  persistSearchState()

  if (mode === 'loadMore') {
    const addedCount = results.value.length - beforeCount
    if (addedCount > 0) {
      appStore.showToast(`已新增 ${addedCount} 个书源`, 'success')
    } else if (!hasMoreSources.value) {
      appStore.showToast('没有更多书源了', 'warning')
    } else {
      appStore.showToast('本批次未找到更多匹配书源', 'warning')
    }
  }
}

async function selectCandidate(item: CandidateItem) {
  selectedCandidate.value = item
  candidatePreview.value = null
  try {
    candidatePreview.value = await getBookInfo(item.book.bookUrl, item.book.origin)
  } catch {
    candidatePreview.value = null
  }
}

function loadMoreSources() {
  if (!store.book || loadingMore.value || !hasMoreSources.value) return

  if (isLeader.value) {
    closeAvailableSourceSSE()
    loadingMore.value = true
    searching.value = true
    openAvailableSourceSSE('loadMore')
    return
  }

  loadingMore.value = true
  searching.value = true
  broadcast({ type: 'request-load-more', tabId })
}

async function handleSwitch(res: SearchBook) {
  if (store.loading) return
  try {
    const nextBook = await store.switchSource(res.bookUrl, res.origin)
    store.closePanel()
    appStore.showToast(`已切换到 ${nextBook?.originName || nextBook?.origin || res.origin}`, 'success')
  } catch (e: any) {
    appStore.showToast(`切换失败: ${e?.message || '未知错误'}`, 'error')
  }
}

function restoreSearchPref() {
  try {
    const raw = localStorage.getItem(SEARCH_PREF_KEY)
    if (!raw) return
    const parsed = JSON.parse(raw) as { concurrentCount?: number; searchSize?: number }
    if (typeof parsed.concurrentCount === 'number') {
      concurrentCount.value = clampNumber(parsed.concurrentCount, 4, 128)
    }
    if (typeof parsed.searchSize === 'number') {
      searchSize.value = clampNumber(parsed.searchSize, 20, 1000)
    }
    normalizeSearchControls()
  } catch {
    // ignore bad data
  }
}

function persistSearchPref() {
  localStorage.setItem(
    SEARCH_PREF_KEY,
    JSON.stringify({
      concurrentCount: clampNumber(concurrentCount.value, 4, 128),
      searchSize: clampNumber(searchSize.value, 20, 1000),
    }),
  )
}

function restorePersistedState() {
  if (!stateStorageKey.value) return false
  try {
    const raw = localStorage.getItem(stateStorageKey.value)
    if (!raw) return false
    const parsed = JSON.parse(raw) as PersistedSearchState
    if (!parsed || !Array.isArray(parsed.results)) return false
    if (Date.now() - (parsed.updatedAt || 0) > SEARCH_STATE_MAX_AGE) {
      clearPersistedState()
      return false
    }
    results.value = parsed.results
    lastIndex.value = typeof parsed.lastIndex === 'number' ? parsed.lastIndex : -1
    hasMoreSources.value = !!parsed.hasMoreSources
    progress.value = {
      processed: parsed.progress?.processed || 0,
      total: parsed.progress?.total || 0,
      matched: parsed.progress?.matched || parsed.results.length,
    }
    if (typeof parsed.concurrentCount === 'number') {
      concurrentCount.value = clampNumber(parsed.concurrentCount, 4, 128)
    }
    if (typeof parsed.searchSize === 'number') {
      searchSize.value = clampNumber(parsed.searchSize, 20, 1000)
    }
    searching.value = false
    loadingMore.value = false
    if (!selectedCandidate.value && preparedResults.value.length) {
      void selectCandidate(preparedResults.value[0])
    }
    return true
  } catch {
    return false
  }
}

function persistSearchState() {
  if (!stateStorageKey.value || !store.book) return
  const payload: PersistedSearchState = {
    results: results.value,
    lastIndex: lastIndex.value,
    hasMoreSources: hasMoreSources.value,
    progress: progress.value,
    concurrentCount: clampNumber(concurrentCount.value, 4, 128),
    searchSize: clampNumber(searchSize.value, 20, 1000),
    updatedAt: Date.now(),
  }
  localStorage.setItem(stateStorageKey.value, JSON.stringify(payload))
}

function clearPersistedState() {
  if (!stateStorageKey.value) return
  localStorage.removeItem(stateStorageKey.value)
}

function initializeChannel() {
  if (channel) {
    channel.close()
    channel = null
  }
  if (!channelName.value || typeof BroadcastChannel === 'undefined') return
  channel = new BroadcastChannel(channelName.value)
  channel.onmessage = (event: MessageEvent<SearchChannelMessage>) => {
    const msg = event.data
    if (!msg || msg.tabId === tabId) return
    noteLeaderSignal()

    if (msg.type === 'needSnapshot' && isLeader.value) {
      broadcast({
        type: 'snapshot',
        tabId,
        payload: {
          data: results.value,
          lastIndex: lastIndex.value,
          hasMore: hasMoreSources.value,
          processed: progress.value.processed,
          total: progress.value.total,
          matched: progress.value.matched,
        },
      })
      return
    }

    if (msg.type === 'request-load-more' && isLeader.value) {
      loadMoreSources()
      return
    }

    if (msg.type === 'start') {
      isFollower.value = true
      isLeader.value = false
      searching.value = true
      loadingMore.value = false
      return
    }

    if (msg.type === 'error') {
      searching.value = false
      loadingMore.value = false
      return
    }

    if (msg.type === 'cancel') {
      closeAvailableSourceSSE()
      stopHeartbeat()
      stopFollowerMonitor()
      releaseLockIfOwned()
      isLeader.value = false
      isFollower.value = false
      searching.value = false
      loadingMore.value = false
      persistSearchState()
      return
    }

    if (msg.type === 'snapshot' || msg.type === 'chunk' || msg.type === 'progress' || msg.type === 'end') {
      applyAvailableSourcePayload(msg.payload)
      if (msg.type === 'end') {
        searching.value = false
        loadingMore.value = false
      }
    }
  }
}

function broadcast(message: SearchChannelMessage) {
  if (!channel) return
  channel.postMessage(message)
}

function acquireSearchLock() {
  if (!lockStorageKey.value) return true
  const now = Date.now()
  try {
    const raw = localStorage.getItem(lockStorageKey.value)
    if (raw) {
      const lock = JSON.parse(raw) as SearchLock
      if (lock.owner !== tabId && lock.expiresAt > now) {
        return false
      }
    }
  } catch {
    // ignore broken lock
  }

  const next: SearchLock = { owner: tabId, expiresAt: now + SEARCH_LOCK_TTL }
  localStorage.setItem(lockStorageKey.value, JSON.stringify(next))
  startHeartbeat()
  return true
}

function startHeartbeat() {
  stopHeartbeat()
  heartbeatTimer = window.setInterval(() => {
    if (!isLeader.value || !lockStorageKey.value) return
    const next: SearchLock = { owner: tabId, expiresAt: Date.now() + SEARCH_LOCK_TTL }
    localStorage.setItem(lockStorageKey.value, JSON.stringify(next))
  }, SEARCH_LOCK_HEARTBEAT)
}

function startFollowerMonitor() {
  stopFollowerMonitor()
  followerMonitorTimer = window.setInterval(() => {
    tryTakeoverIfLeaderStale()
  }, SEARCH_FOLLOWER_CHECK_INTERVAL)
}

function stopFollowerMonitor() {
  if (followerMonitorTimer) {
    window.clearInterval(followerMonitorTimer)
    followerMonitorTimer = null
  }
}

function noteLeaderSignal() {
  lastLeaderSignalAt = Date.now()
}

function tryTakeoverIfLeaderStale() {
  if (!searching.value || !isFollower.value || isLeader.value) return
  if (!hasMoreSources.value) return
  if (Date.now() - lastLeaderSignalAt < SEARCH_FOLLOWER_STALE_MS) return
  if (!acquireSearchLock()) return

  isLeader.value = true
  isFollower.value = false
  loadingMore.value = false
  broadcast({ type: 'start', tabId })
  openAvailableSourceSSE(lastIndex.value >= 0 ? 'loadMore' : 'initial')
  appStore.showToast('检测到其他页面搜索中断，已自动接管继续搜索', 'warning')
}

function stopHeartbeat() {
  if (heartbeatTimer) {
    window.clearInterval(heartbeatTimer)
    heartbeatTimer = null
  }
}

function releaseLockIfOwned() {
  if (!lockStorageKey.value) return
  try {
    const raw = localStorage.getItem(lockStorageKey.value)
    if (!raw) return
    const lock = JSON.parse(raw) as SearchLock
    if (lock.owner === tabId) {
      localStorage.removeItem(lockStorageKey.value)
    }
  } catch {
    // ignore bad lock
  }
}

function clampNumber(value: number, min: number, max: number) {
  return Math.min(max, Math.max(min, Number.isFinite(value) ? value : min))
}

function normalizeSearchControls() {
  concurrentCount.value = Math.round(clampNumber(concurrentCount.value, 4, 128))
  searchSize.value = Math.round(clampNumber(searchSize.value, 20, 1000))
}
</script>

<style scoped>
.reader-source {
  display: flex;
  flex-direction: column;
  height: 100%;
  position: relative;
}

.source-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  margin: 12px 12px 8px;
  border: 1px solid rgba(0,0,0,0.08);
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.22);
  backdrop-filter: blur(12px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.04);
  flex-shrink: 0;
}

.header-left { display: flex; align-items: baseline; gap: 8px; }
.source-header h3 { font-size: 16px; margin: 0; }
.source-count { font-size: 11px; opacity: 0.5; }

.close-btn {
  width: 32px; height: 32px;
  display: flex; align-items: center; justify-content: center;
  border-radius: 8px; color: inherit; opacity: 0.6;
  background: transparent; border: none; cursor: pointer;
}

.source-list {
  flex: 1;
  overflow-y: auto;
  padding: 4px 0 12px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.search-tools {
  margin: 0 12px;
  padding: 10px 12px;
  border-radius: 16px;
  border: 1px solid rgba(0,0,0,0.08);
  background: rgba(255, 255, 255, 0.22);
  backdrop-filter: blur(12px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.04);
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  align-items: center;
}

.tool-item {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 8px;
  font-size: 12px;
  opacity: 0.78;
}

.tool-label {
  font-size: 11px;
  line-height: 1;
  opacity: 0.72;
}

.tool-control {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
}

.tool-control-range input[type='range'] {
  flex: 1;
  min-width: 0;
}

.tool-item.compact input {
  width: 100%;
  box-sizing: border-box;
  border: 1px solid rgba(0,0,0,0.15);
  background: transparent;
  border-radius: 8px;
  height: 28px;
  padding: 0 8px;
  color: inherit;
}

.tool-value {
  min-width: 34px;
  text-align: right;
  font-variant-numeric: tabular-nums;
  opacity: 0.72;
}

.refresh-btn {
  height: 100%;
  min-height: 28px;
  border-radius: 999px;
  border: 1px solid var(--color-primary, #c97f3a);
  background: rgba(201, 127, 58, 0.08);
  color: var(--color-primary, #c97f3a);
  cursor: pointer;
  padding: 0 12px;
  font-size: 12px;
  white-space: nowrap;
}

.refresh-btn:disabled {
  opacity: 0.55;
  cursor: wait;
}

.tool-actions {
  grid-column: 1 / -1;
  display: flex;
  align-items: stretch;
  gap: 8px;
  justify-content: flex-end;
  flex-wrap: wrap;
}

.cancel-btn {
  min-height: 28px;
  padding: 0 12px;
  border-radius: 999px;
  border: 1px solid rgba(0,0,0,0.12);
  background: rgba(0,0,0,0.03);
  color: inherit;
  cursor: pointer;
  font-size: 12px;
  white-space: nowrap;
}

.cancel-btn:hover:not(:disabled) {
  border-color: rgba(0,0,0,0.2);
  background: rgba(0,0,0,0.06);
}

.cancel-btn:disabled {
  opacity: 0.5;
  cursor: wait;
}

.progress-wrap {
  margin: 0 12px;
  padding: 10px 12px;
  border-radius: 16px;
  border: 1px solid rgba(201, 127, 58, 0.18);
  background: rgba(201, 127, 58, 0.1);
  backdrop-filter: blur(12px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.04);
}

.progress-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  font-size: 12px;
  opacity: 0.8;
  margin-bottom: 8px;
}

.progress-track {
  width: 100%;
  height: 8px;
  border-radius: 999px;
  overflow: hidden;
  background: rgba(0, 0, 0, 0.08);
}

.progress-bar {
  height: 100%;
  background: linear-gradient(90deg, var(--color-primary, #c97f3a) 0%, #e4a560 100%);
  transition: width 0.2s ease;
}

.book-brief {
  display: flex;
  gap: 12px;
  margin: 0 12px;
  padding: 12px;
  border-radius: 16px;
  background: rgba(201, 127, 58, 0.08);
  border: 1px solid rgba(201, 127, 58, 0.14);
  backdrop-filter: blur(12px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.04);
}

.book-brief-cover {
  width: 56px;
  height: 76px;
  flex-shrink: 0;
  border-radius: 10px;
  overflow: hidden;
  background: rgba(0, 0, 0, 0.06);
}

.book-brief-cover img,
.book-brief-placeholder {
  width: 100%;
  height: 100%;
}

.book-brief-cover img {
  object-fit: cover;
}

.book-brief-placeholder {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  font-weight: 700;
}

.book-brief-main {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.book-brief-title {
  font-size: 15px;
  font-weight: 700;
}

.book-brief-meta {
  font-size: 12px;
  opacity: 0.68;
  line-height: 1.4;
}

.book-brief-intro {
  font-size: 12px;
  line-height: 1.45;
  opacity: 0.76;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.section-label {
  margin: 2px 12px 0;
  padding: 9px 12px;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  opacity: 0.48;
  font-weight: 700;
  border-radius: 12px;
  border: 1px solid rgba(0,0,0,0.06);
  background: rgba(0, 0, 0, 0.025);
}

.source-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin: 0 12px;
  padding: 12px 14px;
  border-radius: 16px;
  border: 1px solid rgba(0,0,0,0.07);
  background: rgba(255, 255, 255, 0.22);
  backdrop-filter: blur(12px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.03);
  cursor: pointer;
  transition: background 0.2s, transform 0.2s, box-shadow 0.2s, border-color 0.2s;
}

.source-item:hover {
  background: rgba(255, 255, 255, 0.34);
  transform: translateY(-1px);
  box-shadow: 0 10px 28px rgba(0, 0, 0, 0.05);
}
.source-item.current { background: rgba(201, 127, 58, 0.06); cursor: default; }
.source-item.selected {
  background: rgba(201, 127, 58, 0.08);
  border-color: rgba(201, 127, 58, 0.22);
  box-shadow: inset 3px 0 0 var(--color-primary, #c97f3a);
}

.source-main { flex: 1; min-width: 0; }
.source-name-row { display: flex; align-items: center; gap: 8px; margin-bottom: 2px; }
.source-name { font-weight: 600; font-size: 14px; }
.source-tag { font-size: 10px; opacity: 0.5; border: 1px solid currentColor; padding: 0 3px; border-radius: 3px; }
.compare-badge {
  font-size: 10px;
  padding: 1px 6px;
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.06);
  opacity: 0.72;
}

.compare-badge.good {
  background: rgba(82, 196, 26, 0.14);
  color: #3f8f16;
  opacity: 1;
}

.source-book-name {
  font-size: 13px;
  margin-bottom: 2px;
  opacity: 0.86;
}

.source-author { font-size: 11px; opacity: 0.5; margin-bottom: 4px; }
.source-chapter { font-size: 11px; opacity: 0.7; color: var(--color-primary, #c97f3a); }
.source-update { font-size: 11px; opacity: 0.48; margin-top: 2px; }
.source-compare-line {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: 6px;
}

.compare-text {
  font-size: 11px;
  opacity: 0.62;
}

.compare-text.strong {
  color: var(--color-primary, #c97f3a);
  opacity: 0.92;
}

.source-intro {
  font-size: 12px;
  opacity: 0.68;
  line-height: 1.45;
  margin-bottom: 4px;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.source-url { font-size: 11px; opacity: 0.3; }

.switch-btn {
  padding: 4px 12px;
  font-size: 11px;
  border-radius: 12px;
  border: 1px solid var(--color-border);
  background: transparent;
  color: inherit;
  cursor: pointer;
  opacity: 0.7;
}

.source-item:hover .switch-btn {
  background: var(--color-primary, #c97f3a);
  color: white;
  border-color: var(--color-primary, #c97f3a);
  opacity: 1;
}

.compare-panel {
  margin: 0 12px;
  padding: 14px;
  border-radius: 16px;
  background: rgba(201, 127, 58, 0.08);
  border: 1px solid rgba(201, 127, 58, 0.14);
  backdrop-filter: blur(12px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.04);
}

.compare-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 12px;
}

.compare-header h4 {
  margin: 0;
  font-size: 14px;
  font-weight: 700;
}

.switch-btn.primary {
  background: var(--color-primary, #c97f3a);
  border-color: var(--color-primary, #c97f3a);
  color: #fff;
  opacity: 1;
}

.switch-btn:disabled {
  cursor: not-allowed;
  opacity: 0.45;
}

.compare-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.compare-card {
  padding: 12px;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.6);
  border: 1px solid rgba(0, 0, 0, 0.06);
}

.compare-card.highlight {
  border-color: rgba(201, 127, 58, 0.28);
  background: rgba(201, 127, 58, 0.12);
}

.compare-title {
  margin-bottom: 8px;
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  opacity: 0.58;
}

.compare-name {
  font-size: 14px;
  font-weight: 700;
  line-height: 1.4;
  margin-bottom: 4px;
}

.compare-meta {
  font-size: 12px;
  opacity: 0.68;
  margin-bottom: 10px;
}

.compare-line {
  font-size: 12px;
  line-height: 1.5;
  opacity: 0.78;
  margin-top: 4px;
}

.compare-intro {
  display: -webkit-box;
  -webkit-line-clamp: 4;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.loading, .empty {
  padding: 40px 20px;
  text-align: center;
  opacity: 0.5;
  font-size: 14px;
}

.inline-loading {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 12px 20px;
  font-size: 12px;
  opacity: 0.55;
}

.load-more-wrap {
  padding: 4px 12px 8px;
  display: flex;
  justify-content: center;
}

.load-more-btn {
  min-width: 120px;
  padding: 10px 18px;
  border-radius: 999px;
  border: 1px solid rgba(0,0,0,0.08);
  background: transparent;
  color: inherit;
  cursor: pointer;
}

.load-more-btn:hover:not(:disabled) {
  background: rgba(201, 127, 58, 0.08);
  border-color: var(--color-primary, #c97f3a);
  color: var(--color-primary, #c97f3a);
}

.load-more-btn:disabled {
  opacity: 0.5;
  cursor: wait;
}

.spinner {
  width: 24px;
  height: 24px;
  border: 2px solid rgba(0,0,0,0.1);
  border-top-color: var(--color-primary, #c97f3a);
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 12px;
}

.inline-loading .spinner,
.spinner.small {
  width: 14px;
  height: 14px;
  margin: 0;
  border-width: 2px;
}

@keyframes spin { to { transform: rotate(360deg); } }

.switch-overlay {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  z-index: 20;
}

.switch-overlay p { margin-top: 16px; font-size: 14px; opacity: 0.8; }

@media (max-width: 640px) {
  .source-header {
    margin: 10px 10px 6px;
    padding: 14px 16px;
  }

  .search-tools {
    margin: 0 10px;
    grid-template-columns: 1fr;
  }

  .tool-item {
    gap: 6px;
  }

  .tool-control {
    gap: 10px;
  }

  .refresh-btn {
    width: 100%;
  }

  .tool-actions {
    justify-content: stretch;
  }

  .tool-actions > button {
    flex: 1 1 0;
  }

  .search-state-banner {
    margin: 0 12px;
    padding: 10px 12px;
    border-radius: 16px;
    border: 1px solid rgba(0,0,0,0.08);
    background: rgba(0, 0, 0, 0.025);
    display: flex;
    align-items: flex-start;
    gap: 10px;
  }

  .search-state-banner.running {
    border-color: rgba(201, 127, 58, 0.2);
    background: rgba(201, 127, 58, 0.08);
  }

  .search-state-banner.done {
    border-color: rgba(82, 196, 26, 0.18);
    background: rgba(82, 196, 26, 0.08);
  }

  .search-state-dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    margin-top: 5px;
    flex-shrink: 0;
    background: rgba(0,0,0,0.2);
  }

  .search-state-banner.running .search-state-dot {
    background: var(--color-primary, #c97f3a);
    box-shadow: 0 0 0 4px rgba(201, 127, 58, 0.12);
  }

  .search-state-banner.done .search-state-dot {
    background: #3f8f16;
    box-shadow: 0 0 0 4px rgba(82, 196, 26, 0.12);
  }

  .search-state-text {
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  .search-state-title {
    font-size: 12px;
    font-weight: 700;
  }

  .search-state-subtitle {
    font-size: 11px;
    line-height: 1.45;
    opacity: 0.72;
  }

  .search-state-banner {
    margin: 0 10px;
  }

  .progress-wrap {
    margin: 0 10px;
  }

  .progress-head {
    flex-direction: column;
    align-items: flex-start;
  }

  .source-item {
    padding: 12px 16px;
    align-items: flex-start;
  }

  .source-action {
    padding-top: 2px;
  }

  .compare-panel {
    margin: 0 10px;
    padding: 12px;
  }

  .compare-header {
    flex-direction: column;
    align-items: stretch;
  }

  .compare-grid {
    grid-template-columns: 1fr;
  }
}
</style>
