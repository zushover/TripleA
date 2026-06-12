<script setup lang="ts">
import { ref } from 'vue'

defineProps<{
  news: { title: string; url: string; summary: string; date: string }[]
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
      method: 'POST', headers: { 'Content-Type': 'application/json; charset=utf-8' },
      body: JSON.stringify({ url: paperUrl.value }),
    })
    const data = await res.json()
    if (data.result) paperResult.value = data.result
  } catch (_) { /* ignore */ }
  paperLoading.value = false
}

function fmtDate(d: string) {
  if (!d) return ''
  try { return new Date(d).toLocaleDateString('zh-CN', { month:'short', day:'numeric', hour:'2-digit', minute:'2-digit' }) }
  catch { return d.slice(0,16) }
}
</script>

<template>
  <div style="display:flex;gap:16px;height:calc(100vh - 80px);">

    <!-- ====== 左侧：AI 新闻竖排滚动 ====== -->
    <div style="flex:1;min-width:0;display:flex;flex-direction:column;">
      <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:12px;flex-shrink:0;">
        <div style="display:flex;align-items:center;gap:8px;">
          <h1 style="font-size:1.2rem;font-weight:700;margin:0;">Seestar</h1>
          <span style="font-size:10px;color:var(--text-dim);">AI 新闻</span>
        </div>
        <button class="btn" @click="$emit('refreshNews'); $emit('refreshTrending')" style="font-size:11px;">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 11-2.12-9.36L23 10"/></svg>
        </button>
      </div>

      <!-- 新闻列表 -->
      <div style="flex:1;overflow-y:auto;display:flex;flex-direction:column;gap:6px;padding-right:4px;">
        <div v-if="newsLoading" style="color:var(--text-dim);font-size:12px;padding:20px;text-align:center;">加载中...</div>
        <a v-for="n in news" :key="n.title" :href="n.url" target="_blank"
          class="news-item glass-card"
          style="padding:12px 14px;text-decoration:none;display:flex;gap:10px;cursor:pointer;">
          <!-- 时间戳 -->
          <div style="flex-shrink:0;width:52px;text-align:right;">
            <div style="font-size:10px;color:var(--text-dim);line-height:1.3;">{{ fmtDate(n.date) }}</div>
          </div>
          <!-- 内容 -->
          <div style="flex:1;min-width:0;">
            <div class="news-title">{{ n.title }}</div>
            <div class="news-desc">{{ n.summary }}</div>
          </div>
        </a>
      </div>
    </div>

    <!-- ====== 右侧：论文 + GitHub ====== -->
    <div style="width:340px;flex-shrink:0;display:flex;flex-direction:column;gap:14px;">
      <!-- 论文解析 -->
      <div class="glass-card" style="padding:16px;">
        <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">论文解析</div>
        <div style="display:flex;gap:4px;margin-bottom:12px;">
          <input v-model="paperUrl" @keyup.enter="analyzePaper" placeholder="arXiv URL..." style="flex:1;font-size:11px;" />
          <button class="btn btn-primary" @click="analyzePaper" :disabled="paperLoading" style="font-size:11px;padding:5px 10px;">{{ paperLoading ? '...' : '解析' }}</button>
        </div>

        <div v-if="paperLoading" style="display:flex;flex-direction:column;align-items:center;padding:24px 12px;">
          <div class="sun-breathe">
            <svg width="40" height="40" viewBox="0 0 48 48" fill="none" stroke="var(--text)" stroke-width="1.2">
              <circle cx="24" cy="24" r="5" stroke-width="2"/>
              <path d="M24 2v5m0 34v5M8.5 8.5l3.5 3.5m24 24l3.5 3.5M2 24h5m34 0h5M8.5 39.5l3.5-3.5m24-24l3.5-3.5"/>
            </svg>
          </div>
          <div style="font-size:12px;font-weight:600;margin-top:10px;">Triple A 解析中</div>
        </div>

        <div v-else-if="paperResult" style="display:flex;flex-direction:column;gap:8px;max-height:300px;overflow-y:auto;">
          <div style="font-weight:700;font-size:13px;line-height:1.4;">{{ paperResult.title }}</div>
          <div><div style="font-size:10px;color:var(--text-dim);">核心问题</div><div style="font-size:11px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.problem }}</div></div>
          <div><div style="font-size:10px;color:var(--text-dim);">洞见</div><div style="font-size:11px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.insight }}</div></div>
          <div><div style="font-size:10px;color:var(--text-dim);">概念</div><div style="display:flex;flex-wrap:wrap;gap:3px;"><span v-for="c in paperResult.concepts" :key="c" style="font-size:10px;padding:1px 6px;background:var(--bg-hover);border-radius:3px;color:var(--text-secondary);">{{ c }}</span></div></div>
          <div><div style="font-size:10px;color:var(--text-dim);">审稿</div><div style="font-size:11px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.review }}</div></div>
        </div>

        <div v-else style="text-align:center;padding:14px;color:var(--text-dim);font-size:11px;">输入 arXiv 链接，AI 拆解论文</div>
      </div>

      <!-- GitHub -->
      <div class="glass-card" style="padding:14px;flex:1;display:flex;flex-direction:column;min-height:0;">
        <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:8px;">GitHub 热门</div>
        <div style="flex:1;overflow-y:auto;display:flex;flex-direction:column;gap:5px;">
          <a v-for="r in trending" :key="r.name" :href="r.url" target="_blank"
            class="gh-item" style="text-decoration:none;padding:8px 10px;border-radius:6px;display:block;transition:all 0.12s;">
            <div style="font-weight:600;font-size:12px;color:var(--text);">{{ r.name }}</div>
            <div style="font-size:10px;color:var(--text-dim);margin-top:1px;">{{ r.stars }} · {{ r.lang }}</div>
          </a>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
.sun-breathe { animation: breathe 1.4s ease-in-out infinite; }
@keyframes breathe { 0%,100%{transform:scale(1);opacity:.45} 50%{transform:scale(1.35);opacity:1} }

.news-item { transition: all 0.12s; }
.news-item:hover { border-color: var(--text-dim); background: var(--bg-hover); }

.news-title {
  font-weight: 600; font-size: 13px; color: var(--text); line-height: 1.35;
  display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden;
  margin-bottom: 3px;
}
.news-desc {
  font-size: 11px; color: var(--text-dim); line-height: 1.4;
  display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden;
}

.gh-item { cursor: pointer; }
.gh-item:hover { background: var(--bg-hover); }
</style>
