<template>
  <div class="p-6 bg-gray-50 min-h-screen">
    <h1 class="text-2xl font-bold mb-6 text-gray-700">Dynamic Table Dashboard</h1>

    <!-- 搜索 + 过滤 + 按钮: 水平一行 -->
    <div class="filter-container">
      <el-input
        v-model="searchQuery"
        placeholder="Search by ID"
        clearable
        size="medium"
        class="filter-input"
      />

      <el-input
        v-model.number="filterValue"
        type="number"
        placeholder="Filter baseMean"
        size="medium"
        class="filter-input"
      />

      <el-select
        v-model="filterCondition"
        placeholder="Condition"
        size="medium"
        class="filter-input"
      >
        <el-option label=">" value="greater" />
        <el-option label="<" value="less" />
      </el-select>

      <el-button type="primary" @click="applyFilters" size="medium">搜索</el-button>
      <el-button type="warning" @click="resetFilters" size="medium">重置</el-button>

      <el-radio-group v-model="displayFormat" size="medium">
        <el-radio-button label="full">显示完整</el-radio-button>
        <el-radio-button label="2decimals">两位小数</el-radio-button>
      </el-radio-group>
    </div>

    <!-- 数据表格 -->
    <el-table
      :data="paginatedData"
      stripe
      border
      style="width: 100%"
      @sort-change="handleSort"
      header-row-class-name="table-header"
      row-class-name="table-row"
    >
      <el-table-column prop="ID" label="ID" sortable />

      <el-table-column
        v-for="col in numericColumns"
        :key="col"
        :prop="col"
        :label="col"
        sortable
      >
        <template #default="scope">
          {{ formatValue(scope.row[col]) }}
        </template>
      </el-table-column>
    </el-table>

    <!-- 分页 -->
    <div class="mt-6 flex justify-center">
      <el-pagination
        v-model:current-page="currentPage"
        v-model:page-size="pageSize"
        :page-sizes="[10, 20, 50]"
        layout="total, sizes, prev, pager, next, jumper"
        :total="filteredData.length"
        background
        class="custom-pagination"
      />
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import * as XLSX from "xlsx";

const tableData = ref([]);
const filteredData = ref([]);
const searchQuery = ref("");
const filterValue = ref(null);
const filterCondition = ref(null);
const currentPage = ref(1);
const pageSize = ref(10);
const displayFormat = ref("full");

const numericColumns = ["baseMean", "log2FoldChange", "lfcSE", "stat", "pvalue", "padj"];

// 多列排序状态
const sortOrders = ref([]); // [{ field: 'baseMean', order: 'ascending' }, ...]

// 读取 Excel
onMounted(async () => {
  const response = await fetch("/front-end-dynamic-table.xlsx");
  const arrayBuffer = await response.arrayBuffer();
  const workbook = XLSX.read(arrayBuffer, { type: "array" });
  const sheetName = workbook.SheetNames[0];
  const worksheet = workbook.Sheets[sheetName];
  const data = XLSX.utils.sheet_to_json(worksheet, { defval: null });
  tableData.value = data;
  filteredData.value = data; // 初始化显示全部
});

// 格式化显示值
function formatValue(val) {
  if (val === null || val === undefined || val === "NA") return "NA";
  if (typeof val === "number") {
    return displayFormat.value === "2decimals" ? val.toFixed(2) : val;
  }
  return val;
}

// 点击搜索按钮应用筛选
function applyFilters() {
  let data = [...tableData.value];

  if (searchQuery.value) {
    data = data.filter((row) =>
      row.ID?.toString().toLowerCase().includes(searchQuery.value.toLowerCase())
    );
  }

  if (filterValue.value !== null && filterCondition.value) {
    data = data.filter((row) => {
      const val = parseFloat(row.baseMean);
      if (isNaN(val)) return false;
      return filterCondition.value === "greater"
        ? val > filterValue.value
        : val < filterValue.value;
    });
  }

  filteredData.value = data;
  currentPage.value = 1; // 重置分页
}

// 重置按钮刷新页面
function resetFilters() {
  window.location.reload();
}

// 分页
const paginatedData = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value;
  return filteredData.value.slice(start, start + pageSize.value);
});

// 多列排序
function handleSort({ prop, order }) {
  // 移除该列的旧排序
  sortOrders.value = sortOrders.value.filter((s) => s.field !== prop);

  // 如果 order 不为空，就加入新的排序
  if (order) {
    sortOrders.value.push({ field: prop, order });
  }

  // 对 tableData 进行多列排序
  filteredData.value.sort((a, b) => {
    for (let s of sortOrders.value) {
      let aVal = a[s.field];
      let bVal = b[s.field];

      const isANull = aVal === null || aVal === undefined || aVal === "NA";
      const isBNull = bVal === null || bVal === undefined || bVal === "NA";

      if (isANull && isBNull) continue;
      if (isANull) return 1;
      if (isBNull) return -1;

      aVal = parseFloat(aVal);
      bVal = parseFloat(bVal);

      if (aVal < bVal) return s.order === "ascending" ? -1 : 1;
      if (aVal > bVal) return s.order === "ascending" ? 1 : -1;
    }
    return 0;
  });
}
</script>

<style>
/* 响应式过滤区 */
.filter-container {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
  margin-bottom: 16px;
}

.filter-input {
  min-width: 150px;
  border-radius: 6px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.el-button {
  border-radius: 6px;
}

@media (max-width: 768px) {
  .filter-input,
  .el-button,
  .el-radio-group {
    flex: 1 1 100%; /* 移动端每个控件换行 */
  }
}

/* 表头美化 */
.table-header {
  background-color: #f3f6f9;
  font-weight: bold;
  color: #3b4750;
}

/* 鼠标悬停整行高亮 */
.table-row:hover {
  background-color: #eef2f7 !important;
}

/* 分页器美化 */
.custom-pagination >>> .el-pager li.active {
  background-color: #409eff !important;
  color: white !important;
  border-radius: 4px;
}

.custom-pagination >>> .el-pagination__sizes,
.custom-pagination >>> .el-pagination__jump,
.custom-pagination >>> .el-pagination__prev,
.custom-pagination >>> .el-pagination__next {
  border-radius: 4px;
}
</style>
