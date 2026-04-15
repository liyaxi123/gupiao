<template>
  <div class="stock-filter">
    <el-card class="filter-card">
      <template #header>
        <div class="card-header">
          <span>MA60 股票筛选器</span>
        </div>
      </template>

      <div class="filter-controls">
        <el-form :model="filterForm" label-position="top">
          <el-row :gutter="20">
            <el-col :xs="24" :sm="8">
              <el-form-item label="时间范围">
                <el-select
                  v-model="filterForm.months"
                  placeholder="请选择时间范围"
                  style="width: 100%">
                  <el-option label="15天" :value="0.5" />
                  <el-option label="一个月" :value="1" />
                  <el-option label="两个月" :value="2" />
                </el-select>
              </el-form-item>
            </el-col>
            <el-col :xs="24" :sm="8">
              <el-form-item label="Macd波动范围">
                <el-input-number
                  v-model="filterForm.volatility"
                  :precision="2"
                  :step="0.01"
                  :min="0"
                  style="width: 100%" />
              </el-form-item>
            </el-col>
            <el-col :xs="24" :sm="8" class="search-btn-col">
              <el-form-item label="&nbsp;">
                <el-button
                  type="primary"
                  :loading="loading"
                  @click="handleSearch"
                  style="width: 100%">
                  开始筛选
                </el-button>
              </el-form-item>
            </el-col>
          </el-row>
        </el-form>
      </div>
    </el-card>

    <el-card v-if="hasSearched" class="result-card">
      <template #header>
        <div class="card-header result-header">
          <div class="header-left">
            <span>筛选结果 (共 {{ filteredResults.length }} 只股票)</span>
          </div>
          <div class="header-right">
            <el-form :inline="true" size="small" class="local-filter-form">
              <el-form-item label="结果内二次过滤 (波动 <)">
                <el-input-number
                  v-model="localVolatility"
                  :precision="3"
                  :step="0.005"
                  :min="0"
                  class="local-vol-input" />
              </el-form-item>
            </el-form>
          </div>
        </div>
      </template>

      <el-table
        :data="filteredResults"
        stripe
        style="width: 100%"
        v-loading="loading">
        <el-table-column prop="stock_code" label="股票代码" width="180" />
        <el-table-column prop="stock_name" label="股票名称" width="180" />

        <el-table-column prop="ma60_start" label="初始 MA60" sortable>
          <template #default="scope">
            <span>
              {{
                typeof scope.row.ma60_start === 'number'
                  ? scope.row.ma60_start.toFixed(2)
                  : '--'
              }}
            </span>
          </template>
        </el-table-column>

        <el-table-column prop="ma60_end" label="结束 MA60" sortable>
          <template #default="scope">
            <span>
              {{
                typeof scope.row.ma60_end === 'number'
                  ? scope.row.ma60_end.toFixed(2)
                  : '--'
              }}
            </span>
          </template>
        </el-table-column>

        <el-table-column prop="ma60_diff" label="MA60 差值" sortable>
          <template #default="scope">
            <span
              :class="{
                highlight:
                  typeof scope.row.ma60_diff === 'number' &&
                  Math.abs(scope.row.ma60_diff) < localVolatility
              }">
              {{
                typeof scope.row.ma60_diff === 'number'
                  ? scope.row.ma60_diff.toFixed(4)
                  : '--'
              }}
            </span>
          </template>
        </el-table-column>

        <el-table-column prop="latest_price" label="最新价" sortable>
          <template #default="scope">
            <span>
              {{
                typeof scope.row.latest_price === 'number'
                  ? scope.row.latest_price.toFixed(2)
                  : '--'
              }}
            </span>
          </template>
        </el-table-column>

        <el-table-column prop="update_time" label="更新时间" width="200" />
      </el-table>
    </el-card>

    <el-empty
      v-if="hasSearched && results.length === 0 && !loading"
      description="暂无符合条件的股票" />
  </div>
</template>

<script setup>
import { ref, reactive, computed } from 'vue'
import axios from 'axios'
import { ElMessage } from 'element-plus'

const loading = ref(false)
const hasSearched = ref(false)
const results = ref([])
const localVolatility = ref(0.07) // 前端本地二次过滤的波动限制

const filterForm = reactive({
  months: 0.5,
  volatility: 0.07
})

// 前端二次过滤逻辑
const filteredResults = computed(() => {
  return results.value.filter((item) => {
    if (typeof item.ma60_diff !== 'number') return false
    return Math.abs(item.ma60_diff) < localVolatility.value
  })
})

const handleSearch = async () => {
  loading.value = true
  hasSearched.value = true
  try {
    const res = await axios.get('/api/stocks', {
      params: filterForm
    })
    results.value = res.data || []
    ElMessage.success(`筛选完成，共 ${results.value.length} 只股票`)
  } catch (err) {
    console.error(err)
    ElMessage.error('请求失败')
    results.value = []
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.stock-filter {
  max-width: 1000px;
  margin: 0 auto;
  padding: 10px;
}
.filter-card,
.result-card {
  margin-bottom: 20px;
}
.card-header {
  font-weight: bold;
  font-size: 1.1rem;
}
.result-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 15px;
}
.header-right :deep(.el-form-item) {
  margin-bottom: 0;
}
.highlight {
  color: #67c23a;
  font-weight: bold;
}

.local-vol-input {
  width: 130px;
}
/* 响应式调整 */
@media screen and (max-width: 768px) {
  .stock-filter {
    padding: 5px;
  }
  .card-header {
    font-size: 1rem;
  }
  .result-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }
  .header-right {
    width: 100%;
  }
  .header-right :deep(.el-form-item) {
    width: 100%;
    margin-right: 0;
  }
  .header-right :deep(.el-form-item__content) {
    width: 100%;
  }
  .local-vol-input {
    width: 100%;
  }
  .local-filter-form {
    width: 100%;
  }
  .search-btn-col {
    margin-top: -10px;
  }
  /* 表格内边距微调 */
  :deep(.el-card__body) {
    padding: 10px;
  }
  :deep(.el-card__header) {
    padding: 10px 15px;
  }
}
</style>
