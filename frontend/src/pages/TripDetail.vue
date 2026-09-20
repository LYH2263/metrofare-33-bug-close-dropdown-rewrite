<script setup>
import { onMounted, ref, watch } from 'vue'
import { useRoute } from 'vue-router'
import { getJSON } from '../api'
const route = useRoute()
const rec = ref(null)
const err = ref('')
const load = async () => {
  err.value = ''; rec.value = null
  try {
    const h = await getJSON(`/api/history/${route.params.id}`)
    let input = {}, result = {}
    try { input = JSON.parse(h.input_json) } catch { /* ignore */ }
    try { result = JSON.parse(h.result_json) } catch { /* ignore */ }
    rec.value = { ...h, input, result }
  } catch (e) { err.value = '记录不存在或已被删除' }
}
onMounted(load); watch(() => route.params.id, load)
</script>
<template>
  <div class="page">
    <h1>试算记录 #{{ route.params.id }}</h1>
    <p><router-link to="/history">← 返回记录列表</router-link></p>
    <div v-if="rec" class="panel">
      <p>
        请求编码：<strong>{{ rec.input.start }} → {{ rec.input.end }}</strong>
        <span v-if="rec.result.diverted" class="tag tag-divert">封站改道</span>
      </p>
      <p>实际使用编码：<strong>{{ rec.result.actual_start ?? rec.result.start }} → {{ rec.result.actual_end ?? rec.result.end }}</strong></p>
      <p v-if="rec.result.path">途经：{{ rec.result.path.join(' → ') }}</p>
      <p>站数 {{ rec.result.hops }} · 票价 <strong>¥{{ rec.result.fare }}</strong></p>
      <p v-if="rec.result.diverted && rec.result.actual_start !== rec.result.start" class="muted">起点 {{ rec.result.start }} 封闭，实际从 {{ rec.result.actual_start }} 出发</p>
      <p v-if="rec.result.diverted && rec.result.actual_end !== rec.result.end" class="muted">终点 {{ rec.result.end }} 封闭，实际改到 {{ rec.result.actual_end }}</p>
      <p class="muted">写入时间 {{ rec.created_at }}（记录为写入当时定稿，后续封站/解封不再改写）</p>
    </div>
    <p v-else-if="err" class="err">{{ err }}</p>
  </div>
</template>
