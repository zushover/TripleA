<script setup lang="ts">
import { ref, onMounted } from 'vue'

const paperUrl = ref('')
const paperLoading = ref(false)
const paperResult = ref<null | { title: string; problem: string; insight: string; concepts: string[]; review: string }>(null)
const news = ref<{ title: string; url: string; summary: string }[]>([])
const trending = ref<{ name: string; desc: string; stars: string; url: string; lang: string }[]>([])
const newsLoading = ref(false)

async function analyzePaper() {
  if (!paperUrl.value.trim() || paperLoading.value) return
  paperLoading.value = true
  paperResult.value = null
  try {
    const res = await fetch('http://127.0.0.1:8899/api/seestar/paper', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json; charset=utf-8' },
      body: JSON.stringify({ url: paperUrl.value }),
    })
    const data = await res.json()
    if (data.result) paperResult.value = data.result
  } catch (_) { /* ignore */ }
  paperLoading.value = false
}

async function loadNews(force = false) {
  newsLoading.value = true
  try {
    const url = 'http://127.0.0.1:8899/api/seestar/news' + (force ? '?refresh=1' : '')
    const res = await fetch(url)
    const data = await res.json()
    if (data.news) news.value = data.news
  } catch (_) { /* ignore */ }
  newsLoading.value = false
}

async function loadTrending(force = false) {
  try {
    const url = 'http://127.0.0.1:8899/api/seestar/trending' + (force ? '?refresh=1' : '')
    const res = await fetch(url)
    const data = await res.json()
    if (data.repos) trending.value = data.repos
  } catch (_) { /* ignore */ }
}

onMounted(() => { loadNews(); loadTrending() })
</script>

<template>
  <div>
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:20px;">
      <h1 style="font-size:1.3rem;font-weight:700;margin:0;">Seestar</h1>
      <button class="btn" @click="loadNews(true); loadTrending(true)" style="font-size:12px;">
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 11-2.12-9.36L23 10"/></svg>
        刷新
      </button>
    </div>

    <!-- ===== AI 新闻 RSS ===== -->
    <div style="margin-bottom:20px;">
      <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">
        AI 新闻 · aihot.virxact.com
      </div>
      <div v-if="news.length === 0 && newsLoading" style="color:var(--text-dim);font-size:12px;padding:8px;">加载中...</div>
      <div v-else style="display:grid;grid-template-columns:repeat(4,1fr);gap:8px;">
        <a v-for="n in news" :key="n.title" :href="n.url" target="_blank"
          class="glass-card" style="padding:12px 14px;text-decoration:none;display:block;transition:all 0.15s;"
          :style="{ cursor: n.url ? 'pointer' : 'default' }">
          <div style="font-weight:600;font-size:13px;color:var(--text);margin-bottom:4px;line-height:1.3;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden;">
            {{ n.title }}
          </div>
          <div style="font-size:11px;color:var(--text-dim);line-height:1.4;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden;">
            {{ n.summary }}
          </div>
        </a>
      </div>
    </div>

    <!-- ===== 双栏 ===== -->
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:14px;">

      <!-- 论文解析 -->
      <div>
        <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">论文解析</div>
        <div class="glass-card" style="padding:16px;">
          <div style="display:flex;gap:6px;margin-bottom:14px;">
            <input v-model="paperUrl" @keyup.enter="analyzePaper" placeholder="粘贴 arXiv URL..."
              style="flex:1;font-size:12px;" />
            <button class="btn btn-primary" @click="analyzePaper" :disabled="paperLoading" style="font-size:12px;">
              {{ paperLoading ? '解析中' : '解析' }}
            </button>
          </div>

          <!-- 太阳呼吸动画 — 解析中 -->
          <div v-if="paperLoading" style="display:flex;flex-direction:column;align-items:center;justify-content:center;padding:32px 20px;">
            <div class="sun-breathe">
              <svg width="48" height="48" viewBox="0 0 48 48" fill="none" stroke="var(--text)" stroke-width="1.2">
                <circle cx="24" cy="24" r="5" stroke-width="2"/>
                <path d="M24 2v5m0 34v5M8.5 8.5l3.5 3.5m24 24l3.5 3.5M2 24h5m34 0h5M8.5 39.5l3.5-3.5m24-24l3.5-3.5"/>
              </svg>
            </div>
            <div style="font-size:13px;font-weight:600;color:var(--text);margin-top:12px;">Triple A 正在为您解析论文</div>
            <div style="font-size:11px;color:var(--text-dim);margin-top:4px;">读取摘要 · 提取洞见 · 审稿视角</div>
          </div>

          <!-- 结果 -->
          <div v-else-if="paperResult" style="display:flex;flex-direction:column;gap:10px;">
            <div style="font-weight:700;font-size:14px;line-height:1.4;">{{ paperResult.title }}</div>
            <div style="font-size:10px;font-weight:600;color:var(--text-dim);">核心问题</div>
            <div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.problem }}</div>
            <div style="font-size:10px;font-weight:600;color:var(--text-dim);">关键概念</div>
            <div style="display:flex;flex-wrap:wrap;gap:4px;">
              <span v-for="c in paperResult.concepts" :key="c" style="font-size:11px;padding:2px 8px;background:var(--bg-hover);border-radius:4px;color:var(--text-secondary);">{{ c }}</span>
            </div>
            <div style="font-size:10px;font-weight:600;color:var(--text-dim);">核心洞见</div>
            <div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.insight }}</div>
            <div style="font-size:10px;font-weight:600;color:var(--text-dim);">审稿视角</div>
            <div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.review }}</div>
          </div>

          <!-- 空 -->
          <div v-else style="text-align:center;padding:20px;color:var(--text-dim);font-size:12px;">
            输入 arXiv 论文链接，AI 自动拆解核心问题、洞见和审稿视角
          </div>
        </div>
      </div>

      <!-- GitHub -->
      <div>
        <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">GitHub 热门</div>
        <div v-if="trending.length === 0" style="color:var(--text-dim);font-size:12px;padding:8px;">加载中...</div>
        <div v-else style="display:flex;flex-direction:column;gap:6px;">
          <a v-for="r in trending" :key="r.name" :href="r.url" target="_blank"
            class="glass-card" style="padding:10px 14px;text-decoration:none;display:block;transition:all 0.15s;">
            <div style="display:flex;justify-content:space-between;align-items:flex-start;gap:6px;">
              <div style="flex:1;min-width:0;">
                <div style="font-weight:600;font-size:13px;color:var(--text);">{{ r.name }}</div>
                <div style="font-size:11px;color:var(--text-dim);margin-top:2px;line-height:1.4;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden;">{{ r.desc }}</div>
              </div>
              <div style="font-size:10px;color:var(--text-dim);white-space:nowrap;text-align:right;">
                <div>{{ r.stars }}</div>
                <div v-if="r.lang" style="margin-top:2px;">{{ r.lang }}</div>
              </div>
            </div>
          </a>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
.sun-breathe { animation: breathe 1.4s ease-in-out infinite; }
@keyframes breathe {
  0%, 100% { transform: scale(1); opacity: 0.45; }
  50% { transform: scale(1.35); opacity: 1; }
}
.glass-card:hover { border-color: var(--text-dim); transform: translateY(-1px); }
</style>
