<script setup>
import { onMounted, ref } from 'vue'
import { getJSON, postJSON } from '../api'
const stations = ref([])
const start = ref('A1')
const end = ref('B2')
const out = ref(null)
const savedId = ref(null)
const err = ref('')

const label = (s) => s.closed ? `${s.code} ${s.name}（封闭→${s.divert_to}）` : `${s.code} ${s.name}`

onMounted(async () => {
  // 下拉保留已封闭站原编码：用户仍可直接选择，试算时由后端改到邻站
  stations.value = (await getJSON('/api/stations?include_closed=1')).items
})

const sameAsTrial = () =>
  out.value && out.value._start === start.value && out.value._end === end.value

// 只读试算：persist=false，不落库
const run = async () => {
  err.value = ''
  savedId.value = null
  try {
    out.value = {
      _start: start.value,
      _end: end.value,
      ...(await postJSON('/api/quote', { start: start.value, end: end.value, persist: false })),
    }
  } catch (e) { err.value = e.message }
}

// 写入：按同样的原编码重新试算并落库，实际编码与途经以邻站版本快照保存
const save = async () => {
  err.value = ''
  try {
    const r = await postJSON('/api/quote', { start: start.value, end: end.value, persist: true })
    savedId.value = r.run_id
    out.value = { _start: start.value, _end: end.value, ...r }
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
      <button @click="save" :disabled="!out || !out.reachable || !sameAsTrial()">写入记录</button>
    </div>
    <div v-if="out" class="panel">
      <template v-if="out.reachable">
        <p>站数 {{ out.hops }} · 票价 <span class="hero-num">¥{{ out.fare }}</span></p>
        <p v-if="out.path" class="muted">途经 {{ out.path.join(' → ') }}</p>
        <p v-if="out.diverted" class="tag tag-divert">
          <template v-if="out.actual_start !== out.start">起点 {{ out.start }} 已封闭，实际从 {{ out.actual_start }} 站出发。</template>
          <template v-if="out.actual_end !== out.end">终点 {{ out.end }} 已封闭，实际到 {{ out.actual_end }} 站。</template>
        </p>
        <p v-if="savedId" class="ok">
          已写入 <router-link :to="`/history/${savedId}`">#{{ savedId }}</router-link>，打开可见实际编码与途经
        </p>
      </template>
      <p v-else class="muted">不可达</p>
    </div>
    <p v-if="err" class="err">{{ err }}</p>
  </div>
</template>
