<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { request, setToken, clearToken } from './api'

const user = ref(null)
const mode = ref('signin') // 'signin' | 'signup'
const busy = ref(false)
const loading = ref(true)

const name = ref('')
const email = ref('')
const password = ref('')
const confirmation = ref('')

const posts = ref([])
const title = ref('')
const content = ref('')
const editingId = ref(null)

const message = ref('')
const success = ref('')

const searchQuery = ref('')
const filterMyPosts = ref(false)
const sortOrder = ref('newest') // 'newest' | 'oldest'
const copiedPostId = ref(null)
const currentTime = ref('')

let timer = null

function updateClock() {
  const now = new Date()
  currentTime.value = now.toLocaleDateString('en-US', {
    weekday: 'short',
    month: 'short',
    day: 'numeric',
    year: 'numeric'
  }).toUpperCase() + ' // ' + now.toLocaleTimeString('en-US', { hour12: false })
}

function resetMessages() {
  message.value = ''
  success.value = ''
}

function resetForm() {
  title.value = ''
  content.value = ''
  editingId.value = null
}

function changeMode() {
  mode.value = mode.value === 'signin' ? 'signup' : 'signin'
  password.value = ''
  confirmation.value = ''
  resetMessages()
}

function signOut() {
  clearToken()
  user.value = null
  password.value = ''
  confirmation.value = ''
  filterMyPosts.value = false
  resetForm()
  resetMessages()
  success.value = 'OPERATOR SIGNED OUT. SESSION TERMINATED.'
}

function sessionExpired() {
  signOut()
  message.value = 'YOUR SESSION EXPIRED. RE-AUTHENTICATE TO ACCESS SECURE DISPATCHES.'
}

onMounted(() => {
  updateClock()
  timer = setInterval(updateClock, 1000)
  loadPosts()
  window.addEventListener('session-expired', sessionExpired)
})

onUnmounted(() => {
  if (timer) clearInterval(timer)
  window.removeEventListener('session-expired', sessionExpired)
})

async function loadPosts() {
  loading.value = true
  try {
    posts.value = await request('/posts')
  } catch (error) {
    message.value = error.message
  } finally {
    loading.value = false
  }
}

async function authenticate() {
  if (busy.value) return
  resetMessages()
  if (mode.value === 'signup' && password.value !== confirmation.value) {
    message.value = 'PASSKEY MISMATCH. Confirmation does not match password.'
    return
  }
  busy.value = true
  try {
    if (mode.value === 'signup') {
      await request('/auth/signup', {
        method: 'POST',
        body: { name: name.value.trim(), email: email.value.trim(), password: password.value }
      })
      mode.value = 'signin'
      password.value = ''
      confirmation.value = ''
      success.value = 'CREDENTIALS REGISTERED. Sign in with your passkey.'
    } else {
      const result = await request('/auth/login', {
        method: 'POST',
        body: { email: email.value.trim(), password: password.value }
      })
      setToken(result.accessToken)
      user.value = await request('/me', { protectedCall: true })
      password.value = ''
      resetForm()
      success.value = `ACCESS GRANTED. Welcome back, ${user.value.name}.`
    }
  } catch (error) {
    message.value = error.message
  } finally {
    busy.value = false
  }
}

function startEdit(post) {
  resetMessages()
  editingId.value = post.id
  title.value = post.title
  content.value = post.content
  // Scroll compose box into view on mobile
  const composeEl = document.getElementById('compose-section')
  if (composeEl) {
    composeEl.scrollIntoView({ behavior: 'smooth', block: 'start' })
  }
}

async function savePost() {
  if (busy.value) return
  busy.value = true
  resetMessages()
  try {
    const editing = editingId.value !== null
    await request(editing ? `/posts/${editingId.value}` : '/posts', {
      method: editing ? 'PUT' : 'POST',
      protectedCall: true,
      body: { title: title.value.trim(), content: content.value.trim() },
    })
    resetForm()
    success.value = editing ? 'DISPATCH UPDATED ON THE WIRE.' : 'DISPATCH TRANSMITTED & PUBLISHED.'
    await loadPosts()
  } catch (error) {
    message.value = error.message
  } finally {
    busy.value = false
  }
}

async function removePost(post) {
  if (busy.value) return
  const confirmed = window.confirm(`CONFIRM EXPUNGE:\nPermanently remove dispatch “${post.title}”? This cannot be undone.`)
  if (!confirmed) return
  busy.value = true
  resetMessages()
  try {
    await request(`/posts/${post.id}`, { method: 'DELETE', protectedCall: true })
    if (editingId.value === post.id) resetForm()
    success.value = 'DISPATCH EXPUNGED FROM THE ARCHIVE.'
    await loadPosts()
  } catch (error) {
    message.value = error.message
  } finally {
    busy.value = false
  }
}

function insertTag(tag) {
  if (!title.value.startsWith(`[${tag}] `)) {
    title.value = `[${tag}] ${title.value.replace(/^\[.*?\]\s*/, '')}`
  }
}

async function copyPost(post) {
  try {
    const text = `${post.title}\n\n${post.content}\n\n— Via Campus Board (${dateLabel(post.createdAt)})`
    await navigator.clipboard.writeText(text)
    copiedPostId.value = post.id
    setTimeout(() => {
      if (copiedPostId.value === post.id) copiedPostId.value = null
    }, 2000)
  } catch (e) {
    // Fallback if clipboard API restricted
    copiedPostId.value = post.id
    setTimeout(() => { copiedPostId.value = null }, 2000)
  }
}

const dateLabel = (value) => {
  if (!value) return ''
  const d = new Date(value)
  return d.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'short',
    day: '2-digit'
  }) + ' · ' + d.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', hour12: false })
}

const timeAgo = (value) => {
  if (!value) return ''
  const diffSec = Math.floor((new Date() - new Date(value)) / 1000)
  if (diffSec < 60) return 'JUST NOW'
  if (diffSec < 3600) return `${Math.floor(diffSec / 60)}M AGO`
  if (diffSec < 86400) return `${Math.floor(diffSec / 3600)}H AGO`
  return `${Math.floor(diffSec / 86400)}D AGO`
}

const readingTime = (text) => {
  if (!text) return '1 MIN READ'
  const words = text.trim().split(/\s+/).length
  const minutes = Math.ceil(words / 180)
  return `${minutes} MIN READ (${words} WORDS)`
}

const myPostsCount = computed(() => {
  if (!user.value) return 0
  return posts.value.filter(p => p.ownerId === user.value.id).length
})

const filteredPosts = computed(() => {
  let list = [...posts.value]
  if (filterMyPosts.value && user.value) {
    list = list.filter(p => p.ownerId === user.value.id)
  }
  const q = searchQuery.value.trim().toLowerCase()
  if (q) {
    list = list.filter(p =>
      (p.title && p.title.toLowerCase().includes(q)) ||
      (p.content && p.content.toLowerCase().includes(q))
    )
  }
  if (sortOrder.value === 'oldest') {
    list.sort((a, b) => new Date(a.createdAt) - new Date(b.createdAt))
  } else {
    list.sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt))
  }
  return list
})
</script>

<template>
  <div class="editorial-shell">
    <!-- Top News Ticker Marquee -->
    <aside class="ticker-bar" aria-label="Campus Wire Live Status">
      <div class="ticker-content">
        <span class="ticker-badge">LIVE WIRE</span>
        <span class="ticker-item">● CAMPUS BOARD EDITORIAL SYSTEM</span>
        <span class="ticker-item">● DATABASE: NEON POSTGRESQL</span>
        <span class="ticker-item">● SECURITY: JWT RFC-7519 COMPLIANT</span>
        <span class="ticker-item">● DEPLOYMENT: RENDER CLUSTER</span>
        <span class="ticker-item">● EDITION: GLOBAL ACADEMIC FORUM</span>
        <span class="ticker-item">● ALL POSTS PERSISTENT & AUDITED</span>
      </div>
      <div class="ticker-meta">
        <span class="live-indicator"><span class="pulse-dot"></span>SYS_OK</span>
        <span class="time-stamp">{{ currentTime }}</span>
      </div>
    </aside>

    <!-- Broadsheet Masthead -->
    <header class="masthead">
      <div class="masthead-top">
        <div class="masthead-issue-data">
          <span>VOL. XXIV // NO. 108</span>
          <span class="bullet">◆</span>
          <span>CAMPUS DISPATCHES & INTEL</span>
          <span class="bullet">◆</span>
          <span>EST. 2026</span>
        </div>
        <div class="masthead-actions">
          <a class="nav-chip" href="/swagger-ui.html" target="_blank" rel="noopener" title="Inspect OpenAPI Specification">
            <span class="icon">⚡</span> OPENAPI SPEC ↗
          </a>
          <a class="nav-chip" href="/actuator/health" target="_blank" rel="noopener" title="Inspect Actuator Health Check">
            <span class="icon">❤</span> HEALTH_API
          </a>
        </div>
      </div>

      <div class="masthead-title-wrap">
        <a href="/" class="masthead-title-link">
          <h1 class="masthead-title">THE CAMPUS BOARD</h1>
        </a>
        <div class="masthead-stamp">
          <span>OFFICIAL WIRE</span>
          <span class="stamp-code">SEC-07</span>
        </div>
      </div>

      <div class="masthead-bottom-bar">
        <p class="masthead-motto">
          “A strictly unfiltered broadsheet for academic ideas, peer inquiry, and dispatch intelligence.”
        </p>
        <div class="masthead-stats">
          <span class="stat-pill"><strong>{{ posts.length }}</strong> DISPATCHES LOGGED</span>
          <span v-if="user" class="stat-pill user-active"><strong>{{ user.name }}</strong> CONNECTED</span>
        </div>
      </div>
    </header>

    <!-- Main Editorial Stage -->
    <main class="editorial-main">
      <!-- Status Notice Area -->
      <section v-if="message || success" class="wire-alerts" aria-live="polite">
        <div v-if="message" class="alert-box alert-error" role="alert">
          <div class="alert-tag">FLASH ERRATA // NOTICE</div>
          <p class="alert-text">{{ message }}</p>
          <button class="alert-close" aria-label="Dismiss error notice" @click="message = ''">✕</button>
        </div>
        <div v-if="success" class="alert-box alert-success" role="status">
          <div class="alert-tag">BULLETIN // VERIFIED</div>
          <p class="alert-text">{{ success }}</p>
          <button class="alert-close" aria-label="Dismiss success notice" @click="success = ''">✕</button>
        </div>
      </section>

      <!-- Two-Column Editorial Layout -->
      <div class="editorial-grid">
        <!-- Left Column: The Operator Desk (Account & Compose) -->
        <aside class="operator-column">
          <!-- Auth Section (When Logged Out) -->
          <section v-if="!user" id="auth-desk" class="brutalist-card auth-card">
            <div class="card-header-bar">
              <span class="card-tag">DESK // CREDENTIALS</span>
              <span class="card-status-dot"></span>
            </div>

            <div class="auth-tabs">
              <button
                id="btn-tab-signin"
                type="button"
                :class="['auth-tab-btn', { active: mode === 'signin' }]"
                :disabled="busy"
                @click="mode = 'signin'; resetMessages()"
              >
                SIGN IN
              </button>
              <button
                id="btn-tab-signup"
                type="button"
                :class="['auth-tab-btn', { active: mode === 'signup' }]"
                :disabled="busy"
                @click="mode = 'signup'; resetMessages()"
              >
                REGISTER
              </button>
            </div>

            <div class="card-inner-padded">
              <h2 class="section-headline">
                {{ mode === 'signup' ? 'ENROLL OPERATOR' : 'OPERATOR LOGIN' }}
              </h2>
              <p class="section-subtext">
                {{ mode === 'signup'
                  ? 'Obtain authorized credentials to transmit dispatches to the campus board.'
                  : 'Enter your verified keypair to access the publication desk.'
                }}
              </p>

              <form id="auth-form" @submit.prevent="authenticate">
                <div v-if="mode === 'signup'" class="field-group">
                  <label for="input-name">
                    <span class="label-number">[01]</span> FULL LEGAL / CALLSIGN
                  </label>
                  <input
                    id="input-name"
                    v-model="name"
                    required
                    maxlength="60"
                    autocomplete="name"
                    :disabled="busy"
                    placeholder="e.g. Marie Curie"
                  />
                </div>

                <div class="field-group">
                  <label for="input-email">
                    <span class="label-number">{{ mode === 'signup' ? '[02]' : '[01]' }}</span> UNIVERSITY EMAIL
                  </label>
                  <input
                    id="input-email"
                    v-model="email"
                    required
                    type="email"
                    maxlength="254"
                    autocomplete="username"
                    :disabled="busy"
                    placeholder="student@university.edu"
                  />
                </div>

                <div class="field-group">
                  <div class="label-split">
                    <label for="input-password">
                      <span class="label-number">{{ mode === 'signup' ? '[03]' : '[02]' }}</span> PASSKEY
                    </label>
                    <span v-if="mode === 'signup'" class="length-badge" :class="{ ok: password.length >= 12 }">
                      {{ password.length }}/12+ CHARS
                    </span>
                  </div>
                  <input
                    id="input-password"
                    v-model="password"
                    required
                    type="password"
                    :minlength="mode === 'signup' ? 12 : 1"
                    maxlength="72"
                    :autocomplete="mode === 'signup' ? 'new-password' : 'current-password'"
                    :disabled="busy"
                    placeholder="••••••••••••"
                  />
                </div>

                <div v-if="mode === 'signup'" class="field-group">
                  <label for="input-confirm">
                    <span class="label-number">[04]</span> REPEAT PASSKEY
                  </label>
                  <input
                    id="input-confirm"
                    v-model="confirmation"
                    required
                    type="password"
                    autocomplete="new-password"
                    :disabled="busy"
                    placeholder="••••••••••••"
                  />
                </div>

                <div class="form-guidance">
                  <p v-if="mode === 'signup'">
                    ▲ Passkey must be <strong>12+ characters</strong>. Encrypted in-database using BCrypt work-factor 12.
                  </p>
                  <p v-else>
                    ▲ Tokens are held in volatile memory. Reloading requires fresh authentication.
                  </p>
                </div>

                <button id="btn-submit-auth" type="submit" class="brutalist-button primary-action" :disabled="busy">
                  <span class="btn-text">
                    {{ busy ? 'PROCESSING PROTOCOL…' : (mode === 'signup' ? 'COMMENCE REGISTRATION →' : 'AUTHENTICATE & ENTER →') }}
                  </span>
                </button>
              </form>

              <div class="auth-switch-box">
                <button
                  type="button"
                  class="text-link-button"
                  :disabled="busy"
                  @click="changeMode"
                >
                  {{ mode === 'signup' ? 'Already possess credentials? Sign in' : 'First time on the wire? Register dossier' }}
                </button>
              </div>
            </div>
          </section>

          <!-- User Dossier Section (When Logged In) -->
          <section v-else class="brutalist-card user-dossier-card">
            <div class="card-header-bar invert">
              <span class="card-tag">ACCREDITED CORRESPONDENT</span>
              <span class="badge-active">ACTIVE_SESSION</span>
            </div>

            <div class="card-inner-padded">
              <div class="dossier-id-block">
                <div class="dossier-avatar">
                  {{ user.name ? user.name.charAt(0).toUpperCase() : '?' }}
                </div>
                <div class="dossier-details">
                  <h2 class="dossier-name">{{ user.name }}</h2>
                  <p class="dossier-email">{{ user.email }}</p>
                  <div class="dossier-tags">
                    <span class="mini-tag">KEY: HS256</span>
                    <span class="mini-tag">UUID: #{{ user.id ? user.id.slice(0, 8) : '0000' }}</span>
                  </div>
                </div>
              </div>

              <div class="dossier-actions">
                <button
                  id="btn-sign-out"
                  type="button"
                  class="brutalist-button secondary-action"
                  :disabled="busy"
                  @click="signOut"
                >
                  TERMINATE SESSION [SIGN OUT]
                </button>
              </div>

              <!-- Post Composition Desk -->
              <div id="compose-section" class="compose-block">
                <div class="compose-header">
                  <h3 class="compose-title">
                    {{ editingId ? 'AMEND EXISTING DISPATCH' : 'TRANSMIT NEW DISPATCH' }}
                  </h3>
                  <span v-if="editingId" class="edit-pill">REVISION IN PROGRESS</span>
                </div>

                <div class="quick-tags">
                  <span class="tag-prompt">QUICK PREFIX:</span>
                  <button type="button" class="tag-btn" @click="insertTag('RESEARCH')">+RESEARCH</button>
                  <button type="button" class="tag-btn" @click="insertTag('ANNOUNCE')">+ANNOUNCE</button>
                  <button type="button" class="tag-btn" @click="insertTag('QUERY')">+QUERY</button>
                  <button type="button" class="tag-btn" @click="insertTag('OPINION')">+OPINION</button>
                </div>

                <form id="compose-form" @submit.prevent="savePost">
                  <div class="field-group">
                    <div class="label-split">
                      <label for="input-title">HEADLINE // SUBJECT</label>
                      <span class="char-count" :class="{ warn: title.length > 100 }">
                        {{ title.length }}/120
                      </span>
                    </div>
                    <input
                      id="input-title"
                      v-model="title"
                      required
                      maxlength="120"
                      :disabled="busy"
                      placeholder="Concise, arresting headline…"
                    />
                  </div>

                  <div class="field-group">
                    <div class="label-split">
                      <label for="input-content">BODY // DISPATCH DETAILS</label>
                      <span class="char-count" :class="{ warn: content.length > 3800 }">
                        {{ content.length }}/4000
                      </span>
                    </div>
                    <textarea
                      id="input-content"
                      v-model="content"
                      required
                      maxlength="4000"
                      rows="7"
                      :disabled="busy"
                      placeholder="Write your observation, announcement, or critique in clear prose…"
                    ></textarea>
                  </div>

                  <div class="compose-actions">
                    <button
                      id="btn-save-post"
                      type="submit"
                      class="brutalist-button primary-action"
                      :disabled="busy || !title.trim() || !content.trim()"
                    >
                      <span class="btn-text">
                        {{ busy ? 'COMMITTING BYTES…' : (editingId ? 'CONFIRM AMENDMENT ↗' : 'BROADCAST TO WIRE ↗') }}
                      </span>
                    </button>
                    <button
                      v-if="editingId"
                      id="btn-cancel-edit"
                      type="button"
                      class="brutalist-button cancel-action"
                      :disabled="busy"
                      @click="resetForm"
                    >
                      CANCEL EDIT
                    </button>
                  </div>
                </form>
              </div>
            </div>
          </section>

          <!-- Editorial Manifesto Card -->
          <div class="manifesto-card">
            <h4 class="manifesto-heading">BROADSHEET CODE</h4>
            <ul class="manifesto-list">
              <li>01. Authentic ideas only. Keep arguments sharp and concise.</li>
              <li>02. Public reading is permitted to anyone, anywhere.</li>
              <li>03. Only authors may alter or expunge their own dispatches.</li>
              <li>04. Signatures verified by serverless HS256 tokens.</li>
            </ul>
          </div>
        </aside>

        <!-- Right Column: The Broadsheet Wire (Feed) -->
        <section class="wire-column">
          <!-- Wire Control Bar -->
          <div class="wire-control-bar">
            <div class="wire-title-group">
              <h2 class="wire-headline">THE FRONT PAGE DISPATCHES</h2>
              <span class="wire-count-tag">
                SHOWING {{ filteredPosts.length }} OF {{ posts.length }}
              </span>
            </div>

            <div class="wire-tools">
              <div class="search-input-wrap">
                <span class="search-icon">🔍</span>
                <input
                  id="filter-search"
                  v-model="searchQuery"
                  type="text"
                  placeholder="Filter by keyword…"
                  aria-label="Filter dispatches by keyword"
                />
                <button
                  v-if="searchQuery"
                  class="clear-search-btn"
                  title="Clear search"
                  @click="searchQuery = ''"
                >
                  ✕
                </button>
              </div>

              <div class="filter-actions">
                <button
                  v-if="user"
                  type="button"
                  :class="['filter-btn', { active: filterMyPosts }]"
                  @click="filterMyPosts = !filterMyPosts"
                >
                  {{ filterMyPosts ? '★ MY POSTS' : 'ALL POSTS' }}
                  <span class="pill-number">{{ filterMyPosts ? myPostsCount : posts.length }}</span>
                </button>

                <button
                  type="button"
                  class="filter-btn sort-btn"
                  title="Toggle chronological order"
                  @click="sortOrder = sortOrder === 'newest' ? 'oldest' : 'newest'"
                >
                  {{ sortOrder === 'newest' ? 'NEWEST FIRST ▾' : 'OLDEST FIRST ▴' }}
                </button>

                <button
                  id="btn-refresh-feed"
                  type="button"
                  class="refresh-icon-btn"
                  :disabled="loading || busy"
                  title="Refresh wire feed"
                  @click="loadPosts"
                >
                  <span :class="['refresh-arrow', { spinning: loading }]">↻</span>
                  <span class="refresh-label">REFRESH</span>
                </button>
              </div>
            </div>
          </div>

          <!-- Loading State Skeleton -->
          <div v-if="loading" class="loading-state" role="status">
            <div class="skeleton-card brutalist-card">
              <div class="skeleton-line full"></div>
              <div class="skeleton-line title"></div>
              <div class="skeleton-line half"></div>
              <div class="skeleton-line"></div>
              <p class="skeleton-status">DECODING INCOMING TRANSMISSIONS FROM NEON POSTGRES…</p>
            </div>
          </div>

          <!-- Empty State -->
          <div v-else-if="!filteredPosts.length" class="empty-state brutalist-card">
            <div class="empty-stamp">TABULA RASA</div>
            <h3 class="empty-title">
              {{ searchQuery ? 'NO DISPATCHES MATCH YOUR QUERY.' : 'THE WIRE IS SILENT.' }}
            </h3>
            <p class="empty-desc">
              {{ searchQuery
                ? `No dispatches containing “${searchQuery}” were discovered. Try a broader search term.`
                : 'No dispatches currently filed on this frequency. Authenticate on the left to publish the first story.'
              }}
            </p>
            <button
              v-if="searchQuery"
              class="brutalist-button secondary-action inline"
              @click="searchQuery = ''; filterMyPosts = false"
            >
              RESET ALL FILTERS
            </button>
          </div>

          <!-- Posts Feed -->
          <div v-else class="post-feed">
            <article
              v-for="(post, index) in filteredPosts"
              :key="post.id"
              :class="['dispatch-card', 'brutalist-card', { 'is-owner': user?.id === post.ownerId }]"
            >
              <!-- Card Top Header Stamp -->
              <div class="dispatch-header">
                <div class="dispatch-index">
                  <span class="index-num">DISPATCH // № {{ String(index + 1).padStart(3, '0') }}</span>
                  <span class="time-ago">{{ timeAgo(post.createdAt) }}</span>
                </div>
                <div class="dispatch-stamp-group">
                  <span v-if="user?.id === post.ownerId" class="owner-badge">
                    ★ AUTHORIZED AUTHOR
                  </span>
                  <time class="dispatch-time" :datetime="post.createdAt">
                    {{ dateLabel(post.createdAt) }}
                  </time>
                </div>
              </div>

              <!-- Article Content -->
              <div class="dispatch-body">
                <h3 class="dispatch-title">
                  {{ post.title }}
                </h3>
                <p class="dispatch-text">{{ post.content }}</p>
              </div>

              <!-- Card Footer Strip -->
              <div class="dispatch-footer">
                <div class="dispatch-meta-info">
                  <span class="meta-tag">{{ readingTime(post.content) }}</span>
                  <span class="meta-uuid">REF: {{ post.id.slice(0, 8) }}…</span>
                </div>

                <div class="dispatch-actions">
                  <button
                    type="button"
                    class="action-pill copy-pill"
                    title="Copy dispatch text"
                    @click="copyPost(post)"
                  >
                    {{ copiedPostId === post.id ? '✓ COPIED' : '⎘ COPY' }}
                  </button>

                  <template v-if="user?.id === post.ownerId">
                    <button
                      type="button"
                      class="action-pill edit-pill-btn"
                      :disabled="busy"
                      title="Edit this dispatch"
                      @click="startEdit(post)"
                    >
                      ✎ EDIT
                    </button>
                    <button
                      type="button"
                      class="action-pill delete-pill-btn"
                      :disabled="busy"
                      title="Delete this dispatch"
                      @click="removePost(post)"
                    >
                      ✕ EXPUNGE
                    </button>
                  </template>
                </div>
              </div>

              <!-- Barcode Graphic Element -->
              <div class="card-barcode" aria-hidden="true">
                <span class="bar b1"></span><span class="bar b3"></span><span class="bar b2"></span>
                <span class="bar b1"></span><span class="bar b4"></span><span class="bar b2"></span>
                <span class="bar b1"></span><span class="bar b3"></span><span class="bar b2"></span>
                <span class="bar b4"></span><span class="bar b1"></span><span class="bar b2"></span>
                <span class="barcode-text">AUTHENTICATED DISPATCH // {{ post.id.slice(-6).toUpperCase() }}</span>
              </div>
            </article>

            <!-- Bottom Feed Summary -->
            <div class="feed-summary-stamp">
              <span class="rule-line"></span>
              <span class="summary-text">
                END OF WIRE // {{ filteredPosts.length }} DISPATCHES ARCHIVED // NEON LAKEBASE POSTGRES
              </span>
              <span class="rule-line"></span>
            </div>
          </div>
        </section>
      </div>
    </main>

    <!-- Newspaper Colophon Footer -->
    <footer class="broadsheet-footer">
      <div class="colophon-grid">
        <div class="colophon-col">
          <h4 class="colophon-title">CAMPUS BOARD CHRONICLE</h4>
          <p class="colophon-body">
            Independent editorial communication portal. Engineered for reliable peer transmissions, verified academic discourse, and distributed system education.
          </p>
          <div class="colophon-stamp">PRINTED DIGITALLY · WEEK 07</div>
        </div>

        <div class="colophon-col">
          <h4 class="colophon-title">SYSTEM SPECIFICATION</h4>
          <ul class="colophon-links">
            <li>SERVER: <span>SPRING BOOT 3.5.16 (JAVA 21)</span></li>
            <li>DATABASE: <span>NEON LAKEBASE POSTGRESQL</span></li>
            <li>CLIENT: <span>VUE 3.5 + VITE 7 + VANILLA CSS</span></li>
            <li>ORCHESTRATION: <span>RENDER CLOUD CONTAINER</span></li>
          </ul>
        </div>

        <div class="colophon-col">
          <h4 class="colophon-title">WIRE INDEX & REPOSITORIES</h4>
          <ul class="colophon-nav">
            <li><a href="/swagger-ui.html" target="_blank" rel="noopener">OpenAPI 3.0 Interactive Explorer ↗</a></li>
            <li><a href="/v3/api-docs" target="_blank" rel="noopener">JSON Raw OpenAPI Specification ↗</a></li>
            <li><a href="/actuator/health" target="_blank" rel="noopener">Actuator System Liveness Check ↗</a></li>
            <li><a href="https://github.com/SaatvikSS/campus-board" target="_blank" rel="noopener">GitHub Source Repository ↗</a></li>
          </ul>
        </div>

        <div class="colophon-col colophon-badge-col">
          <div class="press-seal">
            <div class="seal-inner">
              <span>MACSLAB</span>
              <strong>VERIFIED</strong>
              <span>2026</span>
            </div>
          </div>
          <p class="seal-caption">OFFICIAL BROADCAST // ENGLISH SECTION</p>
        </div>
      </div>

      <div class="colophon-bottom-strip">
        <span>© 2026 CAMPUS BOARD · ALL INTELLECTUAL WIRE RIGHTS RESERVED</span>
        <span>BRUTALIST EDITORIAL EDITION v2.4</span>
      </div>
    </footer>
  </div>
</template>
