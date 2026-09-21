<template>
  <div class="panel">
    <div class="head">
      <h4>⚠️ 近期告警</h4>
      <button class="btn" :disabled="loading" @click="load">{{ loading ? '查询中…' : '刷新' }}</button>
    </div>

    <div class="filters">
      <label>从<input v-model="startStr" type="datetime-local" step="1"></label>
      <label>至<input v-model="endStr" type="datetime-local" step="1"></label>
      <label>触发条件
        <select v-model="rule">
          <option value="">全部</option>
          <option v-for="r in ruleOptions" :key="r" :value="r">{{ r }}</option>
        </select>
      </label>
      <button class="btn primary" :disabled="loading" @click="load">筛选</button>
      <button class="btn" :disabled="loading" @click="resetFilters">重置</button>
    </div>

    <div v-if="!records.length" class="empty">{{ loading ? '加载中…' : '当前筛选条件下没有命中记录' }}</div>
    <div v-else class="list">
      <label v-for="(a, i) in records" :key="a.timestamp + '-' + i" class="anomaly-row">
        <input type="checkbox" :checked="selected.has(i)" @change="toggle(i)">
        <span class="a-time">{{ fmtTime(a.timestamp) }}</span>
        <span class="a-dev">#{{ deviceIdsOf(a) }}</span>
        <span v-for="t in a.triggers" :key="t.device_id + t.rule" class="a-tag">
          {{ t.rule }}: {{ fmtVal(t.value) }}<em class="th">(阈值 {{ fmtThreshold(t.threshold) }})</em>
        </span>
      </label>
    </div>

    <div class="footer">
      <label class="sel-all"><input type="checkbox" :checked="allSelected" :disabled="!records.length" @change="toggleAll"> 全选</label>
      <span class="sel-count">已选 {{ selected.size }}/{{ records.length }} 条</span>
      <button class="btn export" :disabled="exporting || loading" @click="exportCsv">
        {{ exporting ? '导出中…' : '导出选中' }}
      </button>
    </div>
    <div v-if="notice" class="notice">{{ notice }}</div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import type { Anomaly } from '../types'

const records = ref<Anomaly[]>([])
const ruleOptions = ref<string[]>([])
const startStr = ref('')
const endStr = ref('')
const rule = ref('')
const loading = ref(false)
const exporting = ref(false)
const notice = ref('')
const selected = ref<Set<number>>(new Set())
let releaseTimer: number | undefined

const allSelected = computed(() => records.value.length > 0 && selected.value.size === records.value.length)

// 列表显示与导出共用的格式化函数，保证文件内容与列表逐条对得上
function fmtTime(t: number) { return new Date(t * 1000).toLocaleString() }
function fmtVal(v: number) { return String(v) }
function fmtThreshold(th: number | string) { return String(th) }
function deviceIdsOf(a: Anomaly) { return [...new Set(a.triggers.map(t => t.device_id))].join(',') }

async function load() {
  loading.value = true
  notice.value = ''
  try {
    const params = new URLSearchParams()
    if (startStr.value) params.set('start', String(new Date(startStr.value).getTime() / 1000))
    if (endStr.value) params.set('end', String(new Date(endStr.value).getTime() / 1000))
    if (rule.value) params.set('rule', rule.value)
    const res = await fetch(`/api/anomalies?${params}`)
    const data = await res.json()
    records.value = data.anomalies || []
    selected.value = new Set()
    if (!records.value.length) notice.value = '当前筛选条件下没有命中记录'
    else if (data.total > records.value.length) notice.value = `命中 ${data.total} 条，仅显示最近 ${records.value.length} 条，导出以当前显示为准`
  } catch {
    notice.value = '查询失败，请稍后重试'
  } finally {
    loading.value = false
  }
}

function resetFilters() {
  startStr.value = ''
  endStr.value = ''
  rule.value = ''
  load()
}

function toggle(i: number) {
  const s = new Set(selected.value)
  if (s.has(i)) s.delete(i); else s.add(i)
  selected.value = s
}

function toggleAll() {
  selected.value = allSelected.value ? new Set() : new Set(records.value.map((_, i) => i))
}

function csvCell(s: string) {
  return /[",\n]/.test(s) ? '"' + s.replace(/"/g, '""') + '"' : s
}

function exportCsv() {
  notice.value = ''
  if (exporting.value) return // 连续点击直接拦截，避免产生多份重复文件
  if (!records.value.length) { notice.value = '当前筛选条件下没有命中记录，未生成文件'; return }
  const picked = records.value.filter((_, i) => selected.value.has(i))
  if (!picked.length) { notice.value = '请先勾选要导出的记录'; return }
  exporting.value = true
  try {
    const lines = [['时间', '设备编号', '设备类型', '触发条件', '超出的数值', '阈值'].join(',')]
    for (const a of picked) { // 顺序与列表当前显示保持一致
      for (const t of a.triggers) {
        lines.push([fmtTime(a.timestamp), String(t.device_id), a.device_type, t.rule, fmtVal(t.value), fmtThreshold(t.threshold)].map(csvCell).join(','))
      }
    }
    const blob = new Blob(['\ufeff' + lines.join('\n')], { type: 'text/csv;charset=utf-8' })
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    const now = new Date()
    const pad = (n: number) => String(n).padStart(2, '0')
    link.href = url
    link.download = `anomalies_${now.getFullYear()}${pad(now.getMonth() + 1)}${pad(now.getDate())}_${pad(now.getHours())}${pad(now.getMinutes())}${pad(now.getSeconds())}.csv`
    document.body.appendChild(link)
    link.click()
    link.remove()
    URL.revokeObjectURL(url)
    notice.value = `已导出 ${picked.length} 条记录`
  } finally {
    // 稍后释放导出锁：拦截快速连点，同时保证取消/完成后可以重新发起
    releaseTimer = window.setTimeout(() => { exporting.value = false }, 800)
  }
}

onMounted(async () => {
  try {
    const res = await fetch('/api/anomaly-rules')
    const data = await res.json()
    ruleOptions.value = data.rules || []
  } catch { /* 下拉为空时仍可导出全部 */ }
  load()
})
onUnmounted(() => { if (releaseTimer) clearTimeout(releaseTimer) })
</script>

<style scoped>
.panel{background:#0d1b2a;border-radius:8px;padding:12px;border:1px solid #1e3a5f;flex:1}
.head{display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
.panel h4{color:#f87171;font-size:13px;margin:0}
.filters{display:flex;flex-wrap:wrap;gap:6px;align-items:center;margin-bottom:8px;font-size:11px;color:#94a3b8}
.filters label{display:flex;align-items:center;gap:4px}
.filters input,.filters select{background:#112233;border:1px solid #1e3a5f;border-radius:4px;color:#e0e6ed;font-size:11px;padding:2px 4px}
.btn{background:#112233;border:1px solid #1e3a5f;border-radius:4px;color:#94a3b8;font-size:11px;padding:3px 10px;cursor:pointer}
.btn:hover:not(:disabled){border-color:#64b5f6;color:#e0e6ed}
.btn:disabled{opacity:.5;cursor:not-allowed}
.btn.primary{color:#64b5f6}
.btn.export{color:#fbbf24;border-color:#78510a}
.empty{color:#64748b;font-size:12px;padding:8px 0}
.list{display:flex;flex-direction:column;max-height:220px;overflow-y:auto}
.anomaly-row{display:flex;gap:8px;padding:4px 0;font-size:11px;color:#fca5a5;flex-wrap:wrap;align-items:center;cursor:pointer}
.a-time{color:#64748b;min-width:130px}
.a-dev{color:#94a3b8;min-width:24px}
.a-tag{background:#7f1d1d33;padding:1px 6px;border-radius:3px;border:1px solid #7f1d1d55}
.th{font-style:normal;color:#94a3b8;margin-left:3px}
.footer{display:flex;align-items:center;gap:10px;margin-top:8px;font-size:11px;color:#94a3b8}
.sel-all{display:flex;align-items:center;gap:4px;cursor:pointer}
.sel-count{flex:1}
.notice{margin-top:6px;font-size:11px;color:#fbbf24}
</style>
