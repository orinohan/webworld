<template>
  <div class="balance-sheet-container">
    <div class="header">
      <h1>资产负债表</h1>
      <div class="action-buttons">
        <el-button type="primary" @click="saveData">保存</el-button>
        <el-button type="success" @click="exportData">导出</el-button>
      </div>
    </div>

    <div class="company-info">
      <el-input v-model="formData.companyName" placeholder="公司名称" />
      <el-date-picker
        v-model="formData.reportDate"
        type="date"
        placeholder="报告日期"
      />
      <div :class="['validation-message', validationClass]">
        {{ validationMessage }}
      </div>
    </div>

    <el-table :data="tableData" border style="width: 100%">
      <el-table-column prop="assetName" label="资产 Assets" />
      <el-table-column prop="lineNo" label="行次 Line No." width="80" />
      <el-table-column label="年初余额 Beginning Balance">
        <template #default="{ row }">
          <el-input-number
            v-model="row.beginningBalance"
            :controls="false"
            :precision="2"
            @change="calculateTotals"
            :disabled="row.isReadOnly"
          />
        </template>
      </el-table-column>
      <el-table-column label="期末余额 Ending Balance">
        <template #default="{ row }">
          <el-input-number
            v-model="row.endingBalance"
            :controls="false"
            :precision="2"
            @change="calculateTotals"
            :disabled="row.isReadOnly"
          />
        </template>
      </el-table-column>
      <el-table-column prop="liabilityName" label="负债和所有者权益 Liabilities and Owner's Equity" />
      <el-table-column prop="liabilityLineNo" label="行次 Line No." width="80" />
      <el-table-column label="年初余额 Beginning Balance">
        <template #default="{ row }">
          <el-input-number
            v-model="row.liabilityBeginningBalance"
            :controls="false"
            :precision="2"
            @change="calculateTotals"
            :disabled="row.isReadOnly"
          />
        </template>
      </el-table-column>
      <el-table-column label="期末余额 Ending Balance">
        <template #default="{ row }">
          <el-input-number
            v-model="row.liabilityEndingBalance"
            :controls="false"
            :precision="2"
            @change="calculateTotals"
            :disabled="row.isReadOnly"
          />
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { ElMessage } from 'element-plus'

interface BalanceSheetItem {
  assetName: string
  lineNo: number
  beginningBalance: number
  endingBalance: number
  liabilityName: string
  liabilityLineNo: number
  liabilityBeginningBalance: number
  liabilityEndingBalance: number
  isReadOnly: boolean
  isHighlighted: boolean
}

const formData = ref({
  companyName: '',
  reportDate: ''
})

const tableData = ref<BalanceSheetItem[]>([])
const validationMessage = ref('')
const validationClass = ref('')

// 初始化表格数据
const initTableData = () => {
  // 这里添加所有资产负债表项目
  tableData.value = [
    {
      assetName: '流动资产 Current assets：',
      lineNo: 1,
      beginningBalance: 0,
      endingBalance: 0,
      liabilityName: '流动负债 Current liabilities:',
      liabilityLineNo: 36,
      liabilityBeginningBalance: 0,
      liabilityEndingBalance: 0,
      isReadOnly: true,
      isHighlighted: false
    },
    // ... 添加其他行
  ]
}

// 计算总额
const calculateTotals = () => {
  // 计算应收账款净额
  const arRow = tableData.value.find(row => row.lineNo === 5)
  const badDebtRow = tableData.value.find(row => row.lineNo === 6)
  const netArRow = tableData.value.find(row => row.lineNo === 7)
  
  if (arRow && badDebtRow && netArRow) {
    netArRow.beginningBalance = arRow.beginningBalance - badDebtRow.beginningBalance
    netArRow.endingBalance = arRow.endingBalance - badDebtRow.endingBalance
  }

  // 计算流动资产合计
  const currentAssetsRows = tableData.value.filter(row => 
    [2,3,4,7,8,9,10,12,13,14,15,16,17].includes(row.lineNo)
  )
  const currentAssetsTotalRow = tableData.value.find(row => row.lineNo === 18)
  
  if (currentAssetsTotalRow) {
    currentAssetsTotalRow.beginningBalance = currentAssetsRows.reduce(
      (sum, row) => sum + (row.beginningBalance || 0), 0
    )
    currentAssetsTotalRow.endingBalance = currentAssetsRows.reduce(
      (sum, row) => sum + (row.endingBalance || 0), 0
    )
  }

  // 计算固定资产净值
  const fixedAssetsCostRow = tableData.value.find(row => row.lineNo === 22)
  const accumulatedDepreciationRow = tableData.value.find(row => row.lineNo === 23)
  const fixedAssetsNetRow = tableData.value.find(row => row.lineNo === 24)
  
  if (fixedAssetsCostRow && accumulatedDepreciationRow && fixedAssetsNetRow) {
    fixedAssetsNetRow.beginningBalance = fixedAssetsCostRow.beginningBalance - accumulatedDepreciationRow.beginningBalance
    fixedAssetsNetRow.endingBalance = fixedAssetsCostRow.endingBalance - accumulatedDepreciationRow.endingBalance
  }

  // 计算资产总计和负债及所有者权益合计的差额
  const assetsTotalRow = tableData.value.find(row => row.lineNo === 35)
  const liabilitiesEquityTotalRow = tableData.value.find(row => row.liabilityLineNo === 70)
  
  if (assetsTotalRow && liabilitiesEquityTotalRow) {
    const beginningDifference = Math.abs(assetsTotalRow.beginningBalance - liabilitiesEquityTotalRow.liabilityBeginningBalance)
    const endingDifference = Math.abs(assetsTotalRow.endingBalance - liabilitiesEquityTotalRow.liabilityEndingBalance)
    
    if (beginningDifference === 0 && endingDifference === 0) {
      validationMessage.value = '恭喜！报表平衡！'
      validationClass.value = 'validation-success'
    } else {
      let message = '报表不平衡：'
      if (beginningDifference > 0) {
        message += `年初差额：${beginningDifference.toLocaleString()}；`
      }
      if (endingDifference > 0) {
        message += `年末差额：${endingDifference.toLocaleString()}`
      }
      validationMessage.value = message
      validationClass.value = 'validation-error'
    }
  }
}

// 保存数据
const saveData = () => {
  const data = {
    companyName: formData.value.companyName,
    reportDate: formData.value.reportDate,
    items: tableData.value
  }
  localStorage.setItem('balanceSheetData', JSON.stringify(data))
  ElMessage.success('数据已保存！')
}

// 导出数据
const exportData = () => {
  const data = {
    companyName: formData.value.companyName,
    reportDate: formData.value.reportDate,
    items: tableData.value
  }
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'balance_sheet.json'
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

// 加载保存的数据
const loadSavedData = () => {
  const savedData = localStorage.getItem('balanceSheetData')
  if (savedData) {
    const data = JSON.parse(savedData)
    formData.value.companyName = data.companyName
    formData.value.reportDate = data.reportDate
    tableData.value = data.items
    calculateTotals()
  }
}

onMounted(() => {
  initTableData()
  loadSavedData()
})
</script>

<style lang="scss" scoped>
.balance-sheet-container {
  padding: 20px;
  background-color: #f8fafc;
  
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    padding-bottom: 10px;
    border-bottom: 2px solid #e2e8f0;
    
    h1 {
      color: #1e3a8a;
    }
  }
  
  .company-info {
    display: flex;
    align-items: center;
    gap: 20px;
    margin-bottom: 20px;
  }
  
  .validation-message {
    padding: 8px;
    border-radius: 4px;
    font-weight: bold;
    min-width: 200px;
    text-align: center;
    margin-left: 20px;
    
    &.validation-success {
      background-color: rgba(16, 185, 129, 0.2);
      color: #065f46;
      border: 1px solid #065f46;
    }
    
    &.validation-error {
      background-color: rgba(239, 68, 68, 0.2);
      color: #991b1b;
      border: 1px solid #991b1b;
    }
  }
  
  :deep(.el-input-number) {
    width: 100%;
  }
  
  :deep(.highlight-cell) {
    font-weight: bold;
    background-color: rgba(144, 238, 144, 0.3);
  }
}
</style> 