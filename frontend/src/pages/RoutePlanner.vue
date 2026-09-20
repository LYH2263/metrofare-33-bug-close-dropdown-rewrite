<script setup>
import { onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import { getJSON, postJSON } from '../api'
const router = useRouter()
const stations = ref([])
const start = ref('A1')
const end = ref('B2')
const out = ref(null)
const err = ref('')
// 下拉仍可直接选出已封闭站的原编码
onMounted(async () => { stations.value = (await getJSON('/api/stations?include_closed=1')).items })
const label = (s) => s.closed ? `${s.name}（封闭·改到 ${s.divert_to}）` : s.name
// 只读试算：persist=false，不落库
const run = async () => {
  err.value = ''
  try { out.value = await postJSON('/api/quote', { start: start.value, end: end.value, persist: false }) }
  catch (e) { err.value = e.message }
}
// 写入：按所选编码落库定稿，实际编码/途经为改道后的版本，然后按编号打开
const save = async () => {
  err.value = ''
  try {
    const r = await postJSON('/api/quote', { start: start.value, end: end.value, persist: true })
    out.value = r
    if (r.run_id != null) router.push(`/history/${r.run_id}`)
  } catch (e) { err.value = e.message }
}
</script>
<template>
  <div class="page"><h1>最短站数票价</h1>
    <div class="panel">
      <select v-model="start"><option v-for="s in stations" :key="s.code" :value="s.code">{{ label(s) }}</option></select>
      →
      <select v-model="end"><option v-for="s in stations" :key="s.code" :value="s.code">{{ label(s) }}</option></select>
      <button @click="run">试算</button>
      <button @click="save" style="margin-left:.5rem">写入并打开</button>
    </div>
    <div v-if="out" class="panel">
      <template v-if="out.reachable">
        <p>站数 {{ out.hops }} · 票价 <span class="hero-num">¥{{ out.fare }}</span></p>
        <p v-if="out.path" class="muted">途经 {{ out.path.join(' → ') }}</p>
        <p v-if="out.diverted" class="tag tag-divert">
          <template v-if="out.actual_start !== out.start">起点 {{ out.start }} 已封闭，实际从 {{ out.actual_start }} 站出发。</template>
          <template v-if="out.actual_end !== out.end">终点 {{ out.end }} 已封闭，实际改到 {{ out.actual_end }} 站。</template>
        </p>
      </template>
      <p v-else class="muted">不可达，未写入记录</p>
    </div>
    <p v-if="err" class="err">{{ err }}</p>
  </div>
</template>
