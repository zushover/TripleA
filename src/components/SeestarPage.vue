<script setup lang="ts">
import { ref } from 'vue'

defineProps<{
  news: { title: string; url: string; summary: string }[]
  trending: { name: string; desc: string; stars: string; url: string; lang: string }[]
  newsLoading: boolean
}>()

defineEmits(['refreshNews', 'refreshTrending'])

const paperUrl = ref('')
const paperLoading = ref(false)
const paperResult = ref<null | { title: string; problem: string; insight: string; concepts: string[]; review: string }>(null)

async function analyzePaper() {
  if (!paperUrl.value.trim() || paperLoading.value) return
  paperLoading.value = true; paperResult.value = null
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
</script>

<template>
  <div>
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:20px;">
      <h1 style="font-size:1.3rem;font-weight:700;margin:0;">Seestar</h1>
      <button class="btn" @click="$emit('refreshNews'); $emit('refreshTrending')" style="font-size:12px;">
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 11-2.12-9.36L23 10"/></svg>
        刷新
      </button>
    </div>

    <!-- ===== AI 新闻 RSS ===== -->
    <div style="margin-bottom:20px;">
      <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">AI 新闻 · aihot.virxact.com</div>
      <div v-if="newsLoading" style="color:var(--text-dim);font-size:12px;padding:8px;">加载中...</div>
      <div v-else style="display:grid;grid-template-columns:repeat(4,1fr);gap:8px;">
        <a v-for="n in news" :key="n.title" :href="n.url" target="_blank"
          class="news-card glass-card" style="padding:12px 14px;text-decoration:none;display:block;">
          <div class="news-title">{{ n.title }}</div>
          <div class="news-summary">{{ n.summary }}</div>
        </a>
      </div>
    </div>

    <!-- ===== 双栏 ===== -->
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:14px;">

      <!-- 论文 -->
      <div>
        <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">论文解析</div>
        <div class="glass-card" style="padding:16px;">
          <div style="display:flex;gap:6px;margin-bottom:14px;">
            <input v-model="paperUrl" @keyup.enter="analyzePaper" placeholder="粘贴 arXiv URL..." style="flex:1;font-size:12px;" />
            <button class="btn btn-primary" @click="analyzePaper" :disabled="paperLoading" style="font-size:12px;">{{ paperLoading ? '解析中' : '解析' }}</button>
          </div>

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

          <div v-else-if="paperResult" style="display:flex;flex-direction:column;gap:10px;">
            <div style="font-weight:700;font-size:14px;">{{ paperResult.title }}</div>
            <div><div style="font-size:10px;font-weight:600;color:var(--text-dim);margin-bottom:2px;">核心问题</div><div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.problem }}</div></div>
            <div><div style="font-size:10px;font-weight:600;color:var(--text-dim);margin-bottom:2px;">关键概念</div><div style="display:flex;flex-wrap:wrap;gap:4px;"><span v-for="c in paperResult.concepts" :key="c" style="font-size:11px;padding:2px 8px;background:var(--bg-hover);border-radius:4px;color:var(--text-secondary);">{{ c }}</span></div></div>
            <div><div style="font-size:10px;font-weight:600;color:var(--text-dim);margin-bottom:2px;">核心洞见</div><div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.insight }}</div></div>
            <div><div style="font-size:10px;font-weight:600;color:var(--text-dim);margin-bottom:2px;">审稿视角</div><div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.review }}</div></div>
          </div>

          <div v-else style="text-align:center;padding:20px;color:var(--text-dim);font-size:12px;">输入 arXiv 链接，AI 拆解核心问题、洞见和审稿视角</div>
        </div>
      </div>

      <!-- GitHub -->
      <div>
        <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">GitHub 热门</div>
        <div v-if="trending.length === 0" style="color:var(--text-dim);font-size:12px;padding:8px;">加载中...</div>
        <div v-else style="display:flex;flex-direction:column;gap:6px;">
          <a v-for="r in trending" :key="r.name" :href="r.url" target="_blank" class="gh-card glass-card" style="padding:10px 14px;text-decoration:none;display:block;">
            <div style="display:flex;justify-content:space-between;align-items:flex-start;gap:6px;">
              <div style="flex:1;min-width:0;">
                <div style="font-weight:600;font-size:13px;color:var(--text);">{{ r.name }}</div>
                <div style="font-size:11px;color:var(--text-dim);margin-top:2px;line-height:1.4;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden;">{{ r.desc }}</div>
              </div>
              <div style="font-size:10px;color:var(--text-dim);white-space:nowrap;text-align:right;">
                <div>{{ r.stars }}</div><div v-if="r.lang" style="margin-top:2px;">{{ r.lang }}</div>
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
@keyframes breathe { 0%,100%{transform:scale(1);opacity:.45} 50%{transform:scale(1.35);opacity:1} }
.news-card:hover, .gh-card:hover { border-color: var(--text-dim); transform: translateY(-1px); }
.news-title { font-weight:600;font-size:13px;color:var(--text);margin-bottom:4px;line-height:1.3;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden; }
.news-summary { font-size:11px;color:var(--text-dim);line-height:1.4;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden; }
</style>
