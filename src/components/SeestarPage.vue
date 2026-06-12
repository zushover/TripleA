<script setup lang="ts">
import { ref, onMounted } from 'vue'

// ── State ──
const paperUrl = ref('')
const paperLoading = ref(false)
const paperResult = ref<null | { title: string; problem: string; insight: string; concepts: string[]; review: string }>(null)
const news = ref<{ title: string; summary: string; source: string }[]>([])
const trending = ref<{ name: string; desc: string; stars: string; url: string }[]>([])
const newsLoading = ref(false)
const activeTab = ref<'papers' | 'trending'>('papers')

// ── Paper Analysis ──
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
    else alert('解析失败: ' + (data.error || '未知错误'))
  } catch (e) {
    alert('请求失败')
  }
  paperLoading.value = false
}

// ── Load News ──
async function loadNews() {
  newsLoading.value = true
  try {
    const res = await fetch('http://127.0.0.1:8899/api/seestar/news')
    const data = await res.json()
    if (data.news) news.value = data.news
  } catch (_) { /* ignore */ }
  newsLoading.value = false
}

// ── Load Trending ──
async function loadTrending() {
  try {
    const res = await fetch('http://127.0.0.1:8899/api/seestar/trending')
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
      <button class="btn" @click="loadNews(); loadTrending()" style="font-size:12px;">
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 11-2.12-9.36L23 10"/></svg>
        刷新
      </button>
    </div>

    <!-- ===== AI 新闻横排卡片 ===== -->
    <div style="margin-bottom:20px;">
      <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">AI 速览</div>
      <div v-if="newsLoading" style="color:var(--text-dim);font-size:12px;padding:8px;">加载中...</div>
      <div v-else-if="news.length === 0" style="color:var(--text-dim);font-size:12px;padding:8px;">暂无新闻</div>
      <div v-else style="display:grid;grid-template-columns:repeat(auto-fill,minmax(240px,1fr));gap:8px;">
        <div v-for="n in news" :key="n.title" class="glass-card" style="padding:14px;">
          <div style="font-size:10px;color:var(--text-dim);margin-bottom:4px;">{{ n.source }}</div>
          <div style="font-weight:600;font-size:13px;margin-bottom:4px;line-height:1.3;">{{ n.title }}</div>
          <div style="font-size:11px;color:var(--text-secondary);line-height:1.5;">{{ n.summary }}</div>
        </div>
      </div>
    </div>

    <!-- ===== 双栏：论文 + GitHub ===== -->
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:14px;">

      <!-- 论文解析 -->
      <div>
        <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">论文解析</div>
        <div class="glass-card" style="padding:14px;">
          <div style="display:flex;gap:6px;margin-bottom:12px;">
            <input v-model="paperUrl" @keyup.enter="analyzePaper" placeholder="粘贴 arXiv URL 或论文链接..."
              style="flex:1;font-size:12px;" />
            <button class="btn btn-primary" @click="analyzePaper" :disabled="paperLoading" style="font-size:12px;">
              {{ paperLoading ? '解析中' : '解析' }}
            </button>
          </div>

          <!-- Loading -->
          <div v-if="paperLoading" style="text-align:center;padding:20px;color:var(--text-dim);font-size:12px;">
            <div class="pulse-dot"></div>
            正在调用 LLM 分析论文...
          </div>

          <!-- Result -->
          <div v-else-if="paperResult" style="display:flex;flex-direction:column;gap:10px;">
            <div style="font-weight:700;font-size:14px;">{{ paperResult.title }}</div>

            <div>
              <div style="font-size:10px;font-weight:600;color:var(--text-dim);margin-bottom:2px;">核心问题</div>
              <div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.problem }}</div>
            </div>

            <div>
              <div style="font-size:10px;font-weight:600;color:var(--text-dim);margin-bottom:2px;">关键概念</div>
              <div style="display:flex;flex-wrap:wrap;gap:4px;">
                <span v-for="c in paperResult.concepts" :key="c" style="font-size:11px;padding:2px 8px;background:var(--bg-hover);border-radius:4px;color:var(--text-secondary);">{{ c }}</span>
              </div>
            </div>

            <div>
              <div style="font-size:10px;font-weight:600;color:var(--text-dim);margin-bottom:2px;">核心洞见</div>
              <div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.insight }}</div>
            </div>

            <div>
              <div style="font-size:10px;font-weight:600;color:var(--text-dim);margin-bottom:2px;">审稿视角</div>
              <div style="font-size:12px;color:var(--text-secondary);line-height:1.5;">{{ paperResult.review }}</div>
            </div>
          </div>

          <!-- Empty -->
          <div v-else style="text-align:center;padding:20px;color:var(--text-dim);font-size:12px;">
            输入论文链接，AI 自动拆解核心问题、关键概念、洞见和审稿视角
          </div>
        </div>
      </div>

      <!-- GitHub 热门 -->
      <div>
        <div style="font-size:10px;font-weight:600;color:var(--text-dim);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:10px;">GitHub 热门</div>
        <div v-if="trending.length === 0" style="color:var(--text-dim);font-size:12px;padding:8px;">加载中...</div>
        <div v-else style="display:flex;flex-direction:column;gap:8px;">
          <div v-for="r in trending" :key="r.name" class="glass-card" style="padding:12px 14px;">
            <a :href="r.url" target="_blank" style="font-weight:600;font-size:13px;color:var(--text);text-decoration:none;">{{ r.name }}</a>
            <div style="font-size:11px;color:var(--text-secondary);margin-top:2px;line-height:1.4;">{{ r.desc }}</div>
            <div style="font-size:10px;color:var(--text-dim);margin-top:4px;">⭐ {{ r.stars }}</div>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
.pulse-dot {
  display: inline-block; width: 8px; height: 8px; background: var(--text); border-radius: 50%;
  margin-right: 6px; animation: pulse 1.4s ease-in-out infinite;
}
@keyframes pulse {
  0%, 100% { opacity: 0.3; }
  50% { opacity: 1; }
}
</style>
