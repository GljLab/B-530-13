<template>
  <div class="booking-list-container">
    <el-row :gutter="16" class="stats-row">
      <el-col :span="6">
        <el-card shadow="hover" class="stat-card stat-new">
          <div class="stat-content">
            <div class="stat-value">{{ todayStats.newBookings || 0 }}</div>
            <div class="stat-label">今日新增预订</div>
          </div>
          <el-icon :size="40" class="stat-icon"><Calendar /></el-icon>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card shadow="hover" class="stat-card stat-pending">
          <div class="stat-content">
            <div class="stat-value">{{ todayStats.pendingCheckin || 0 }}</div>
            <div class="stat-label">今日待入住</div>
          </div>
          <el-icon :size="40" class="stat-icon"><Clock /></el-icon>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card shadow="hover" class="stat-card stat-checkedin">
          <div class="stat-content">
            <div class="stat-value">{{ todayStats.checkedIn || 0 }}</div>
            <div class="stat-label">今日已入住</div>
          </div>
          <el-icon :size="40" class="stat-icon"><User /></el-icon>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card shadow="hover" class="stat-card stat-cancelled">
          <div class="stat-content">
            <div class="stat-value">{{ todayStats.cancelled || 0 }}</div>
            <div class="stat-label">今日取消</div>
          </div>
          <el-icon :size="40" class="stat-icon"><CircleClose /></el-icon>
        </el-card>
      </el-col>
    </el-row>

    <el-card shadow="never" class="filter-card">
      <div class="filter-row">
        <el-input v-model="queryParams.keyword" placeholder="预订单号/客户姓名/手机号" clearable style="width: 220px" @keyup.enter="handleSearch">
          <template #prefix><el-icon><Search /></el-icon></template>
        </el-input>
        <el-select v-model="queryParams.roomTypeId" placeholder="房型" clearable style="width: 140px" @change="handleSearch">
          <el-option v-for="rt in roomTypes" :key="rt.id" :label="rt.typeName" :value="rt.id" />
        </el-select>
        <el-select v-model="queryParams.status" placeholder="预订状态" clearable style="width: 140px" @change="handleSearch">
          <el-option v-for="s in statusOptions" :key="s.value" :label="s.label" :value="s.value" />
        </el-select>
        <el-select v-model="queryParams.source" placeholder="预订来源" clearable style="width: 140px" @change="handleSearch">
          <el-option v-for="s in sourceOptions" :key="s.value" :label="s.label" :value="s.value" />
        </el-select>
        <el-date-picker
          v-model="queryParams.checkinDateRange"
          type="daterange"
          range-separator="至"
          start-placeholder="入住开始日期"
          end-placeholder="入住结束日期"
          value-format="YYYY-MM-DD"
          style="width: 280px"
          @change="handleSearch"
        />
        <el-date-picker
          v-model="queryParams.bookingTimeRange"
          type="datetimerange"
          range-separator="至"
          start-placeholder="预订开始时间"
          end-placeholder="预订结束时间"
          value-format="YYYY-MM-DD HH:mm:ss"
          style="width: 320px"
          @change="handleSearch"
        />
        <el-select v-model="queryParams.sortBy" placeholder="排序" clearable style="width: 160px" @change="handleSearch">
          <el-option label="按预订时间降序" value="bookingTime_desc" />
          <el-option label="按预订时间升序" value="bookingTime_asc" />
          <el-option label="按入住日期降序" value="checkinDate_desc" />
          <el-option label="按入住日期升序" value="checkinDate_asc" />
          <el-option label="按金额降序" value="amount_desc" />
          <el-option label="按金额升序" value="amount_asc" />
        </el-select>
        <el-button type="primary" @click="handleSearch">
          <el-icon><Search /></el-icon>搜索
        </el-button>
        <el-button @click="handleReset">
          <el-icon><Refresh /></el-icon>重置
        </el-button>
      </div>
    </el-card>

    <el-card shadow="never" class="table-card">
      <div class="table-toolbar">
        <div class="table-toolbar-left">
          <el-button v-if="hasPermission('booking:add')" type="primary" @click="handleAddBooking">
            <el-icon><Plus /></el-icon>新增预订
          </el-button>
          <el-button
            v-if="hasPermission('booking:batch:confirm')"
            :disabled="selectedRows.length === 0 || !canBatchConfirm"
            @click="handleBatchConfirm"
          >
            <el-icon><CircleCheck /></el-icon>批量确认
          </el-button>
          <el-button v-if="hasPermission('booking:export')" @click="openExportDialog">
            <el-icon><Download /></el-icon>导出
          </el-button>
        </div>
        <div v-if="selectedRows.length > 0" class="batch-toolbar">
          <el-tag type="info" class="batch-count-tag">
            已选择 <strong>{{ selectedRows.length }}</strong> 条预订单
          </el-tag>
          <el-button size="small" @click="clearSelection">取消选择</el-button>
        </div>
      </div>

      <el-table
        :data="tableData"
        stripe
        border
        v-loading="tableLoading"
        style="width: 100%"
        @selection-change="handleSelectionChange"
      >
        <el-table-column type="selection" width="50" align="center" />
        <el-table-column prop="bookingNo" label="预订单号" width="160" align="center" />
        <el-table-column prop="customerName" label="客户姓名" width="110" align="center" />
        <el-table-column prop="phone" label="手机号" width="130" align="center" />
        <el-table-column prop="roomTypeName" label="房型" min-width="140" />
        <el-table-column prop="roomNumber" label="房号" width="90" align="center">
          <template #default="{ row }">{{ row.roomNumber || '-' }}</template>
        </el-table-column>
        <el-table-column prop="checkinDate" label="入住日期" width="120" align="center" />
        <el-table-column prop="checkoutDate" label="退房日期" width="120" align="center" />
        <el-table-column prop="days" label="天数" width="80" align="center" />
        <el-table-column prop="totalAmount" label="应付金额" width="110" align="center">
          <template #default="{ row }">¥{{ row.totalAmount?.toFixed(2) || '0.00' }}</template>
        </el-table-column>
        <el-table-column label="状态" width="100" align="center">
          <template #default="{ row }">
            <el-tag :type="statusTagType(row.status)" size="small">{{ statusLabel(row.status) }}</el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="bookingTime" label="预订时间" width="170" align="center" />
        <el-table-column prop="sourceName" label="预订来源" width="110" align="center" />
        <el-table-column label="操作" width="280" align="center" fixed="right">
          <template #default="{ row }">
            <template v-if="row.status === 1">
              <el-button v-if="hasPermission('booking:confirm')" type="primary" link size="small" @click="handleConfirm(row)">确认</el-button>
              <el-button v-if="hasPermission('booking:edit')" type="primary" link size="small" @click="handleEdit(row)">修改</el-button>
              <el-button v-if="hasPermission('booking:cancel')" type="danger" link size="small" @click="handleCancel(row)">取消</el-button>
            </template>
            <template v-else-if="row.status === 2">
              <el-button v-if="hasPermission('booking:checkin')" type="success" link size="small" @click="handleCheckin(row)">入住</el-button>
              <el-button v-if="hasPermission('booking:edit')" type="primary" link size="small" @click="handleEdit(row)">修改</el-button>
              <el-button v-if="hasPermission('booking:cancel')" type="danger" link size="small" @click="handleCancel(row)">取消</el-button>
            </template>
            <template v-else-if="row.status === 3">
              <el-button v-if="hasPermission('booking:checkin')" type="success" link size="small" @click="handleCheckin(row)">入住</el-button>
              <el-button v-if="hasPermission('booking:refund')" type="warning" link size="small" @click="handleRefund(row)">退款</el-button>
            </template>
            <template v-else-if="row.status === 4">
              <el-button type="primary" link size="small" @click="handleViewCheckin(row)">查看入住</el-button>
            </template>
            <template v-else-if="row.status === 6">
              <el-button type="danger" link size="small" @click="handleViewCancel(row)">查看取消</el-button>
            </template>
          </template>
        </el-table-column>
      </el-table>

      <div class="pagination-wrap">
        <el-pagination
          v-model:current-page="queryParams.pageNum"
          v-model:page-size="queryParams.pageSize"
          :page-sizes="[20, 50, 100]"
          :total="total"
          layout="total, sizes, prev, pager, next, jumper"
          @size-change="loadBookings"
          @current-change="loadBookings"
        />
      </div>
    </el-card>

    <el-dialog v-model="cancelDialogVisible" title="取消预订" width="480px" destroy-on-close>
      <el-form ref="cancelFormRef" :model="cancelForm" :rules="cancelRules" label-width="100px">
        <el-form-item label="预订单号">
          <span>{{ cancelForm.bookingNo }}</span>
        </el-form-item>
        <el-form-item label="取消原因" prop="reason">
          <el-select v-model="cancelForm.reason" placeholder="请选择取消原因" style="width: 100%">
            <el-option label="客户取消" value="客户取消" />
            <el-option label="房源不可用" value="房源不可用" />
            <el-option label="价格变动" value="价格变动" />
            <el-option label="其他原因" value="其他原因" />
          </el-select>
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input v-model="cancelForm.remark" type="textarea" :rows="3" placeholder="请输入备注" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="cancelDialogVisible = false">取消</el-button>
        <el-button type="danger" :loading="cancelSaving" @click="handleConfirmCancel">确认取消</el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="exportDialogVisible" title="导出预订单" width="560px" destroy-on-close>
      <el-form label-width="100px">
        <el-form-item label="导出范围">
          <el-radio-group v-model="exportForm.scope">
            <el-radio value="current">当前页 ({{ tableData.length }}条)</el-radio>
            <el-radio value="all">全部 ({{ total }}条)</el-radio>
            <el-radio value="selected">已选择 ({{ selectedRows.length }}条)</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="导出字段">
          <el-checkbox-group v-model="exportForm.exportFields">
            <el-checkbox v-for="f in exportFieldOptions" :key="f.value" :value="f.value">{{ f.label }}</el-checkbox>
          </el-checkbox-group>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="exportDialogVisible = false">取消</el-button>
        <el-button type="primary" :loading="exportSaving" @click="handleExport">确认导出</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import {
  Plus, Search, Refresh, Download, Calendar, Clock, User, CircleClose, CircleCheck
} from '@element-plus/icons-vue'
import { useUserStore } from '@/stores/user'
import api from '@/api'

const userStore = useUserStore()
const hasPermission = (p) => userStore.hasPermission(p)

const statusOptions = [
  { value: 1, label: '待确认' },
  { value: 2, label: '已确认' },
  { value: 3, label: '已支付' },
  { value: 4, label: '已入住' },
  { value: 5, label: '已完成' },
  { value: 6, label: '已取消' }
]

const sourceOptions = [
  { value: 'online', label: '线上预订' },
  { value: 'offline', label: '线下预订' },
  { value: 'phone', label: '电话预订' },
  { value: 'walkin', label: '散客' },
  { value: 'ota', label: 'OTA平台' },
  { value: 'member', label: '会员预订' }
]

const exportFieldOptions = [
  { value: 'bookingNo', label: '预订单号' },
  { value: 'customerName', label: '客户姓名' },
  { value: 'phone', label: '手机号' },
  { value: 'roomTypeName', label: '房型' },
  { value: 'roomNumber', label: '房号' },
  { value: 'checkinDate', label: '入住日期' },
  { value: 'checkoutDate', label: '退房日期' },
  { value: 'days', label: '天数' },
  { value: 'totalAmount', label: '应付金额' },
  { value: 'status', label: '状态' },
  { value: 'bookingTime', label: '预订时间' },
  { value: 'sourceName', label: '预订来源' }
]

const todayStats = reactive({
  newBookings: 0,
  pendingCheckin: 0,
  checkedIn: 0,
  cancelled: 0
})

const roomTypes = ref([])
const tableData = ref([])
const total = ref(0)
const tableLoading = ref(false)
const selectedRows = ref([])

const queryParams = reactive({
  keyword: '',
  roomTypeId: null,
  status: null,
  source: null,
  checkinDateRange: [],
  bookingTimeRange: [],
  sortBy: 'bookingTime_desc',
  pageNum: 1,
  pageSize: 20
})

const canBatchConfirm = computed(() => {
  return selectedRows.value.every(row => row.status === 1)
})

const statusTagType = (status) => {
  const map = { 1: 'warning', 2: 'primary', 3: 'success', 4: 'info', 5: 'success', 6: 'danger' }
  return map[status] || 'info'
}

const statusLabel = (status) => {
  const map = { 1: '待确认', 2: '已确认', 3: '已支付', 4: '已入住', 5: '已完成', 6: '已取消' }
  return map[status] || '未知'
}

const loadTodayStats = async () => {
  try {
    const res = await api.booking.stats.today()
    if (res.code === 200 && res.data) {
      todayStats.newBookings = res.data.newBookings || 0
      todayStats.pendingCheckin = res.data.pendingCheckin || 0
      todayStats.checkedIn = res.data.checkedIn || 0
      todayStats.cancelled = res.data.cancelled || 0
    }
  } catch {}
}

const loadRoomTypes = async () => {
  try {
    const res = await api.hotel.getRoomTypes()
    if (res.code === 200) {
      roomTypes.value = res.data || []
    }
  } catch {
    roomTypes.value = []
  }
}

const loadBookings = async () => {
  tableLoading.value = true
  try {
    const params = { ...queryParams }
    if (params.checkinDateRange && params.checkinDateRange.length === 2) {
      params.checkinStartDate = params.checkinDateRange[0]
      params.checkinEndDate = params.checkinDateRange[1]
    }
    delete params.checkinDateRange
    if (params.bookingTimeRange && params.bookingTimeRange.length === 2) {
      params.bookingStartTime = params.bookingTimeRange[0]
      params.bookingEndTime = params.bookingTimeRange[1]
    }
    delete params.bookingTimeRange
    if (params.sortBy) {
      const [sortField, sortOrder] = params.sortBy.split('_')
      params.sortField = sortField
      params.sortOrder = sortOrder
    }
    delete params.sortBy
    const res = await api.booking.page(params)
    if (res.code === 200) {
      tableData.value = res.data?.records || res.data?.list || []
      total.value = res.data?.total || 0
    }
  } catch {
    tableData.value = []
    total.value = 0
  } finally {
    tableLoading.value = false
  }
}

const handleSearch = () => {
  queryParams.pageNum = 1
  loadBookings()
}

const handleReset = () => {
  queryParams.keyword = ''
  queryParams.roomTypeId = null
  queryParams.status = null
  queryParams.source = null
  queryParams.checkinDateRange = []
  queryParams.bookingTimeRange = []
  queryParams.sortBy = 'bookingTime_desc'
  queryParams.pageNum = 1
  loadBookings()
}

const handleSelectionChange = (rows) => {
  selectedRows.value = rows
}

const clearSelection = () => {
  selectedRows.value = []
}

const handleAddBooking = () => {
  ElMessage.info('新增预订功能待开发')
}

const handleConfirm = async (row) => {
  try {
    await ElMessageBox.confirm(`确认预订「${row.bookingNo}」？`, '提示', { type: 'warning' })
    const res = await api.booking.confirm(row.id)
    if (res.code === 200) {
      ElMessage.success('确认成功')
      await loadBookings()
      await loadTodayStats()
    } else {
      ElMessage.error(res.message || '确认失败')
    }
  } catch {}
}

const handleBatchConfirm = async () => {
  const invalidRows = selectedRows.value.filter(row => row.status !== 1)
  if (invalidRows.length > 0) {
    ElMessage.warning('只能批量确认只能选择待确认状态的预订单')
    return
  }
  try {
    await ElMessageBox.confirm(`确认选中的 ${selectedRows.value.length} 条预订单？`, '提示', { type: 'warning' })
    const ids = selectedRows.value.map(r => r.id)
    const res = await api.booking.batchConfirm({ ids })
    if (res.code === 200) {
      ElMessage.success('批量确认成功')
      clearSelection()
      await loadBookings()
      await loadTodayStats()
    } else {
      ElMessage.error(res.message || '批量确认失败')
    }
  } catch {}
}

const handleEdit = (row) => {
  ElMessage.info('修改预订功能待开发')
}

const cancelDialogVisible = ref(false)
const cancelSaving = ref(false)
const cancelFormRef = ref(null)
const cancelForm = reactive({
  id: null,
  bookingNo: '',
  reason: '',
  remark: ''
})
const cancelRules = {
  reason: [{ required: true, message: '请选择取消原因', trigger: 'change' }]
}

const handleCancel = (row) => {
  cancelForm.id = row.id
  cancelForm.bookingNo = row.bookingNo
  cancelForm.reason = ''
  cancelForm.remark = ''
  cancelDialogVisible.value = true
}

const handleConfirmCancel = async () => {
  const valid = await cancelFormRef.value.validate().catch(() => false)
  if (!valid) return
  cancelSaving.value = true
  try {
    const res = await api.booking.cancel(cancelForm.id, {
      reason: cancelForm.reason,
      remark: cancelForm.remark
    })
    if (res.code === 200) {
      ElMessage.success('取消成功')
      cancelDialogVisible.value = false
      await loadBookings()
      await loadTodayStats()
    } else {
      ElMessage.error(res.message || '取消失败')
    }
  } catch {
    ElMessage.error('取消失败')
  } finally {
    cancelSaving.value = false
  }
}

const handleCheckin = (row) => {
  ElMessage.info('入住功能待开发')
}

const handleRefund = (row) => {
  ElMessage.info('退款功能待开发')
}

const handleViewCheckin = (row) => {
  ElMessage.info('查看入住功能待开发')
}

const handleViewCancel = (row) => {
  ElMessage.info('查看取消功能待开发')
}

const exportDialogVisible = ref(false)
const exportSaving = ref(false)
const exportForm = reactive({
  scope: 'current',
  exportFields: ['bookingNo', 'customerName', 'phone', 'roomTypeName', 'checkinDate', 'checkoutDate', 'totalAmount', 'status']
})

const openExportDialog = () => {
  exportForm.scope = 'current'
  exportDialogVisible.value = true
}

const handleExport = async () => {
  exportSaving.value = true
  try {
    let params = {
      exportFields: exportForm.exportFields
    }
    if (exportForm.scope === 'current') {
      params.pageNum = queryParams.pageNum
      params.pageSize = queryParams.pageSize
    } else if (exportForm.scope === 'selected') {
      params.ids = selectedRows.value.map(r => r.id)
    }
    const res = await api.booking.export(params)
    const blob = new Blob([res], { type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' })
    const url = window.URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = `预订单列表_${new Date().toISOString().slice(0, 10)}.xlsx`
    link.click()
    window.URL.revokeObjectURL(url)
    ElMessage.success('导出成功')
    exportDialogVisible.value = false
  } catch {
    ElMessage.error('导出失败')
  } finally {
    exportSaving.value = false
  }
}

onMounted(() => {
  loadTodayStats()
  loadRoomTypes()
  loadBookings()
})
</script>

<style scoped>
.booking-list-container {
  padding: 16px;
}

.stats-row {
  margin-bottom: 16px;
}

.stat-card {
  border-radius: 8px;
  overflow: hidden;
}

.stat-card :deep(.el-card__body) {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px;
}

.stat-content {
  flex: 1;
}

.stat-value {
  font-size: 28px;
  font-weight: 600;
  color: #303133;
  margin-bottom: 4px;
}

.stat-label {
  font-size: 14px;
  color: #909399;
}

.stat-icon {
  opacity: 0.8;
}

.stat-new .stat-value,
.stat-new .stat-icon {
  color: #409eff;
}

.stat-pending .stat-value,
.stat-pending .stat-icon {
  color: #e6a23c;
}

.stat-checkedin .stat-value,
.stat-checkedin .stat-icon {
  color: #67c23a;
}

.stat-cancelled .stat-value,
.stat-cancelled .stat-icon {
  color: #f56c6c;
}

.filter-card {
  margin-bottom: 16px;
}

.filter-row {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
}

.table-card {
  margin-bottom: 16px;
}

.table-toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.table-toolbar-left {
  display: flex;
  gap: 8px;
}

.batch-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
}

.batch-count-tag {
  margin-right: 8px;
}

.pagination-wrap {
  display: flex;
  justify-content: flex-end;
  margin-top: 16px;
}
</style>
