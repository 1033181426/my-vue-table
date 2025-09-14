<template>
  <div>
    <!-- 搜索 ID -->
    <el-input
      v-model="searchId"
      placeholder="Search by ID"
      clearable
      style="width: 300px; margin-bottom: 20px"
    />

    <!-- 数值过滤 -->
    <el-input-number
      v-model="filterValue"
      placeholder="Filter baseMean >="
      :min="0"
      style="margin: 0 10px"
    />

    <!-- 分页选择 -->
    <el-select v-model="pageSize" placeholder="Items per page" style="margin-bottom: 20px">
      <el-option label="10" :value="10" />
      <el-option label="20" :value="20" />
      <el-option label="50" :value="50" />
    </el-select>

    <!-- 表格 -->
    <el-table
      :data="paginatedData"
      border
      style="width: 100%"
      @sort-change="handleSort"
    >
      <el-table-column prop="ID" label="ID" sortable="custom" />

      <!-- 这里我们用 formatter 保证数据原样显示 -->
      <el-table-column
        prop="baseMean"
        label="baseMean"
        sortable="custom"
        :formatter="formatNumber"
      />
      <el-table-column
        prop="log2FoldChange"
        label="log2FoldChange"
        sortable="custom"
        :formatter="formatNumber"
      />
      <el-table-column
        prop="lfcSE"
        label="lfcSE"
        :formatter="formatNumber"
      />
      <el-table-column
        prop="stat"
        label="stat"
        :formatter="formatNumber"
      />
      <el-table-column
        prop="pvalue"
        label="pvalue"
        :formatter="formatNumber"
      />
      <el-table-column
        prop="padj"
        label="padj"
        :formatter="formatNumber"
      />
    </el-table>

    <!-- 分页器 -->
    <el-pagination
      background
      layout="prev, pager, next"
      :page-size="pageSize"
      :total="filteredData.length"
      v-model:current-page="currentPage"
      style="margin-top: 20px"
    />
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import * as XLSX from 'xlsx'

// 数据存储
const tableData = ref([])

const searchId = ref('')
const filterValue = ref(0)
const pageSize = ref(10)
const currentPage = ref(1)
const sortKey = ref('')
const sortOrder = ref('')

// 读取 Excel 文件
onMounted(async () => {
  const response = await fetch('/front-end-dynamic-table.xlsx')
  const arrayBuffer = await response.arrayBuffer()
  const workbook = XLSX.read(arrayBuffer, { type: 'array' })
  const worksheet = workbook.Sheets[workbook.SheetNames[0]]
  const jsonData = XLSX.utils.sheet_to_json(worksheet, { defval: null })
  tableData.value = jsonData
})

// 过滤
const filteredData = computed(() => {
  return tableData.value.filter(row => {
    const matchId = row.ID?.toString().toLowerCase().includes(searchId.value.toLowerCase())
    const matchValue = row.baseMean >= filterValue.value
    return matchId && matchValue
  })
})

// 排序
const sortedData = computed(() => {
  if (!sortKey.value) return filteredData.value
  return [...filteredData.value].sort((a, b) => {
    const valA = a[sortKey.value] ?? 0
    const valB = b[sortKey.value] ?? 0
    return sortOrder.value === 'ascending' ? valA - valB : valB - valA
  })
})

// 分页
const paginatedData = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  return sortedData.value.slice(start, start + pageSize.value)
})

// 格式化函数（显示完整数字，不截断）
function formatNumber(row, column, cellValue) {
  if (cellValue === null || cellValue === undefined || cellValue === 'NA') {
    return 'NA'
  }
  return cellValue.toString()  // 保证原样显示
}

function handleSort({ prop, order }) {
  sortKey.value = prop
  sortOrder.value = order
}
</script>
