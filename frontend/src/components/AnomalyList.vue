<template>
  <div class="panel">
    <div class="head">
      <h4>⚠️ 近期告警</h4>
      <el-button size="small" type="primary" :disabled="exporting" @click="exportSelected">
        {{ exporting ? '导出中…' : '导出选中' }}
      </el-button>
    </div>
    <div class="filters">
      <el-date-picker
        v-model="range" type="datetimerange" size="small" class="range-picker"
        range-separator="至" start-placeholder="开始时间" end-placeholder="结束时间" />
      <div class="filter-row">
        <el-select v-model="rule" size="small" placeholder="触发条件" class="rule-select">
          <el-option label="全部条件" value="" />
          <el-option v-for="r in ruleOptions" :key="r" :label="r" :value="r" />
        </el-select>
        <el-button size="small" :loading="loading" @click="query">查询</el-button>
      </div>
    </div>

    <div v-if="!rows.length" class="empty">{{ emptyText }}</div>
    <template v-else>
      <div class="select-all">
        <el-checkbox v-model="allChecked">全选</el-checkbox>
        <span class="sel-count">已选 {{ selectedCount }}/{{ rows.length }}</span>
      </div>
      <div class="list">
        <div v-for="row in rows" :key="row.key" class="anomaly-row">
          <el-checkbox v-model="row.checked" class="row-check" />
          <span class="a-time">{{ ts(row.timestamp) }}</span>
          <span class="a-dev">#{{ row.triggers[0].device_id }} {{ row.device_type }}</span>
          <span v-for="t in row.triggers" :key="t.rule" class="a-tag">{{ t.rule }}: {{ t.value.toFixed(1) }}</span>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import type { Anomaly } from '../types'

interface Row extends Anomaly { key: string; checked: boolean }

const rows = ref<Row[]>([])
const ruleOptions = ref<string[]>([])
const rule = ref('')
const range = ref<[Date, Date]>([new Date(Date.now() - 3600e3), new Date()])
const loading = ref(false)
const exporting = ref(false)
const emptyText = ref('加载中…')

const selectedCount = computed(() => rows.value.filter(r => r.checked).length)
const allChecked = computed({
  get: () => rows.value.length > 0 && rows.value.every(r => r.checked),
  set: (v: boolean) => rows.value.forEach(r => { r.checked = v })
})

function ts(t: number) { return new Date(t * 1000).toLocaleTimeString() }

async function query() {
  loading.value = true
  try {
    const params = new URLSearchParams()
    if (range.value?.[0]) params.set('start', String(range.value[0].getTime() / 1000))
    if (range.value?.[1]) params.set('end', String(range.value[1].getTime() / 1000))
    if (rule.value) params.set('rule', rule.value)
    const res = await fetch(`/api/anomalies?${params}`)
    const data = await res.json()
    rows.value = (data.anomalies || []).map((a: Anomaly, i: number) => ({ ...a, key: `${a.timestamp}-${i}`, checked: false }))
    emptyText.value = rows.value.length ? '' : '当前筛选条件下没有命中记录'
  } catch {
    emptyText.value = '查询失败，请稍后重试'
  } finally {
    loading.value = false
  }
}

async function loadRules() {
  try {
    const res = await fetch('/api/anomaly-rules')
    ruleOptions.value = (await res.json()).rules || []
  } catch { /* 规则列表加载失败时仍可查询与导出 */ }
}

async function exportSelected() {
  if (exporting.value) return // 连续点击只保留第一次，避免产生多份重复文件
  if (!rows.value.length) {
    ElMessage.info('当前筛选条件下没有命中记录，未生成导出文件')
    return
  }
  const selected = rows.value.filter(r => r.checked)
  if (!selected.length) {
    ElMessage.warning('请先勾选要导出的记录')
    return
  }
  exporting.value = true
  try {
    await ElMessageBox.confirm(`将导出 ${selected.length} 条告警记录为 CSV 文件，是否继续？`, '导出确认', {
      confirmButtonText: '导出', cancelButtonText: '取消', type: 'info'
    })
    downloadCsv(buildCsv(selected))
    ElMessage.success(`已导出 ${selected.length} 条记录`)
  } catch {
    // 用户取消导出：不生成文件，finally 复位状态后可重新发起
  } finally {
    exporting.value = false
  }
}

function pad(n: number) { return String(n).padStart(2, '0') }
function formatTime(t: number) {
  const d = new Date(t * 1000)
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}`
}
function csvCell(v: unknown) {
  const s = String(v ?? '')
  return /[",\n]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s
}
// 每条记录的每个触发条件一行，与列表中展示的标签逐条对应
function buildCsv(selected: Row[]) {
  const lines = [['时间', '设备编号', '设备类型', '触发条件', '超出数值', '阈值'].join(',')]
  for (const r of selected) {
    for (const t of r.triggers) {
      lines.push([formatTime(r.timestamp), t.device_id, r.device_type, t.rule, t.value, t.threshold].map(csvCell).join(','))
    }
  }
  return lines.join('\r\n')
}
function downloadCsv(csv: string) {
  const blob = new Blob(['\uFEFF' + csv], { type: 'text/csv;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const d = new Date()
  const a = document.createElement('a')
  a.href = url
  a.download = `anomaly_records_${d.getFullYear()}${pad(d.getMonth() + 1)}${pad(d.getDate())}_${pad(d.getHours())}${pad(d.getMinutes())}${pad(d.getSeconds())}.csv`
  document.body.appendChild(a)
  a.click()
  a.remove()
  URL.revokeObjectURL(url)
}

onMounted(() => { loadRules(); query() })
</script>

<style scoped>
.panel{background:#0d1b2a;border-radius:8px;padding:12px;border:1px solid #1e3a5f;flex:1}
.head{display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
.panel h4{color:#f87171;font-size:13px}
.filters{display:flex;flex-direction:column;gap:6px;margin-bottom:8px}
.range-picker{width:100%}
.filter-row{display:flex;gap:6px}
.rule-select{flex:1}
.empty{color:#64748b;font-size:12px}
.select-all{display:flex;align-items:center;gap:8px;padding:2px 0 6px;border-bottom:1px solid #1e3a5f;margin-bottom:4px}
.sel-count{font-size:11px;color:#64748b}
.list{display:flex;flex-direction:column}
.anomaly-row{display:flex;gap:8px;align-items:center;padding:4px 0;font-size:11px;color:#fca5a5;flex-wrap:wrap}
.row-check{margin-right:-4px}
.a-time{color:#64748b;min-width:70px}
.a-dev{color:#93c5fd;min-width:110px}
.a-tag{background:#7f1d1d33;padding:1px 6px;border-radius:3px;border:1px solid #7f1d1d55}
</style>
