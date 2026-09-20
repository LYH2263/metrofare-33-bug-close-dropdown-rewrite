<script setup>
import { onMounted, ref, watch } from 'vue'
import { useRoute } from 'vue-router'
import { getJSON } from '../api'
const route = useRoute()
const row = ref(null)
const input = ref({})
const result = ref({})
const notFound = ref(false)

const load = async () => {
  notFound.value = false
  try {
    row.value = await getJSON(`/api/history/${route.params.id}`)
    input.value = JSON.parse(row.value.input_json || '{}')
    result.value = JSON.parse(row.value.result_json || '{}')
  } catch { notFound.value = true }
}
onMounted(load); watch(() => route.params.id, load)
</script>
<template>
  <div class="page">
    <h1>试算记录 #{{ route.params.id }}</h1>
    <p v-if="notFound" class="err">记录不存在</p>
    <div v-else-if="row" class="panel">
      <p>请求编码：{{ input.start }} → {{ input.end }}</p>
      <p>
        实际使用：<strong>{{ result.actual_start ?? result.start }} → {{ result.actual_end ?? result.end }}</strong>
        <span v-if="result.diverted" class="tag tag-divert">封站改道</span>
      </p>
      <p v-if="result.actual_start && result.actual_start !== result.start" class="muted">
        起点 {{ result.start }} 封闭，实际从 {{ result.actual_start }} 出发
      </p>
      <p v-if="result.actual_end && result.actual_end !== result.end" class="muted">
        终点 {{ result.end }} 封闭，实际到 {{ result.actual_end }}
      </p>
      <p v-if="result.path">途经：{{ result.path.join(' → ') }}</p>
      <p>站数 {{ result.hops }} · 票价 <span class="hero-num">¥{{ result.fare }}</span></p>
      <p class="muted">{{ row.created_at }}</p>
      <p><router-link to="/history">返回列表</router-link></p>
    </div>
  </div>
</template>
