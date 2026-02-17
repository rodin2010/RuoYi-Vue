<template>
  <div class="app-container">
    <!-- 搜索栏 -->
    <el-form :model="queryParams" ref="queryForm" size="small" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="科目名称" prop="subjectName">
        <el-input
          v-model="queryParams.subjectName"
          placeholder="请输入科目名称"
          clearable
          style="width: 200px"
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item label="科目类型" prop="subjectType">
        <el-select v-model="queryParams.subjectType" placeholder="请选择科目类型" clearable style="width: 200px">
          <el-option
            v-for="dict in dict.type.account_subject_type"
            :key="dict.value"
            :label="dict.label"
            :value="dict.value"
          />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="el-icon-search" size="mini" @click="handleQuery">搜索</el-button>
        <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <!-- 操作栏 -->
    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="el-icon-plus"
          size="mini"
          @click="handleAdd"
          v-hasPermi="['account:subject:add']"
        >新增</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="success"
          plain
          icon="el-icon-edit"
          size="mini"
          :disabled="single"
          @click="handleUpdate"
          v-hasPermi="['account:subject:edit']"
        >修改</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="danger"
          plain
          icon="el-icon-delete"
          size="mini"
          :disabled="multiple"
          @click="handleDelete"
          v-hasPermi="['account:subject:remove']"
        >删除</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="warning"
          plain
          icon="el-icon-download"
          size="mini"
          @click="handleExport"
          v-hasPermi="['account:subject:export']"
        >导出</el-button>
      </el-col>
      <right-toolbar :showSearch.sync="showSearch" @queryTable="getList" :columns="columns" />
    </el-row>

    <!-- 数据表格 -->
    <el-table
      v-loading="loading"
      :data="subjectList"
      @selection-change="handleSelectionChange"
      :fit="true"
      border
    >
      <el-table-column type="selection" width="55" align="center" />
      <el-table-column label="序号" type="index" width="60" align="center" />
      <el-table-column label="科目ID" align="center" prop="subjectId" v-if="columns[0].visible" min-width="80" />
      <el-table-column label="科目类型" align="center" prop="subjectType" v-if="columns[1].visible" min-width="100">
        <template slot-scope="scope">
          <dict-tag :options="dict.type.account_subject_type" :value="scope.row.subjectType" />
        </template>
      </el-table-column>
      <el-table-column label="科目编码" align="center" prop="subjectCode" v-if="columns[2].visible" min-width="150" :show-overflow-tooltip="true">
        <template slot-scope="scope">
          <el-link type="primary" @click="handleDetail(scope.row)" class="link-ellipsis">{{ scope.row.subjectCode }}</el-link>
        </template>
      </el-table-column>
      <el-table-column label="科目名称" align="center" prop="subjectName" v-if="columns[3].visible" min-width="120" :show-overflow-tooltip="true" />
      <el-table-column label="科目区域" align="center" prop="subjectArea" v-if="columns[4].visible" min-width="120" :show-overflow-tooltip="true">
        <template slot-scope="scope">
          <span>{{ formatArea(scope.row.subjectArea) }}</span>
        </template>
      </el-table-column>
      <el-table-column label="金额" align="right" prop="subjectAmount" v-if="columns[5].visible" min-width="100">
        <template slot-scope="scope">
          <span class="amount-text">{{ formatAmount(scope.row.subjectAmount) }}</span>
        </template>
      </el-table-column>
      <el-table-column label="相关人员" align="center" prop="userName" v-if="columns[6].visible" min-width="100" :show-overflow-tooltip="true" />
      <el-table-column label="附件" align="center" prop="attachments" v-if="columns[7].visible" min-width="80">
        <template slot-scope="scope">
          <el-link type="primary" v-if="scope.row.attachments" @click="handlePreview(scope.row.attachments)">
            查看({{ getFileCount(scope.row.attachments) }})
          </el-link>
          <span v-else>-</span>
        </template>
      </el-table-column>
      <el-table-column label="状态" align="center" prop="status" v-if="columns[8].visible" min-width="80">
        <template slot-scope="scope">
          <el-switch
            v-model="scope.row.status"
            active-value="0"
            inactive-value="1"
            @change="handleStatusChange(scope.row)"
          />
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" width="150" fixed="right" class-name="small-padding fixed-width">
        <template slot-scope="scope">
          <el-button
            size="mini"
            type="text"
            icon="el-icon-edit"
            @click="handleUpdate(scope.row)"
            v-hasPermi="['account:subject:edit']"
          >修改</el-button>
          <el-button
            size="mini"
            type="text"
            icon="el-icon-delete"
            @click="handleDelete(scope.row)"
            v-hasPermi="['account:subject:remove']"
          >删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 分页 -->
    <pagination
      v-show="total > 0"
      :total="total"
      :page.sync="queryParams.pageNum"
      :limit.sync="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 新增/修改对话框 -->
    <el-dialog :title="title" :visible.sync="open" width="500px" append-to-body :close-on-click-modal="false" custom-class="dialog-title-left">
      <el-form ref="form" :model="form" :rules="rules" label-width="80px">
        <!-- 固定部分：科目编码 -->
        <el-form-item label="科目编码">
          <el-input v-model="form.subjectCode" disabled placeholder="自动生成" />
        </el-form-item>
        <!-- 可滚动部分 -->
        <div class="form-scroll-container">
          <el-form-item label="科目类型" prop="subjectType">
            <el-select v-model="form.subjectType" placeholder="请选择科目类型" style="width: 100%">
              <el-option
                v-for="dict in dict.type.account_subject_type"
                :key="dict.value"
                :label="dict.label"
                :value="dict.value"
              />
            </el-select>
          </el-form-item>
          <el-form-item label="科目名称" prop="subjectName">
            <el-input v-model="form.subjectName" placeholder="请输入科目名称" />
          </el-form-item>
          <el-form-item label="科目区域" prop="subjectArea">
            <el-select v-model="form.subjectArea" multiple placeholder="请选择科目区域" style="width: 100%">
              <el-option
                v-for="area in areaOptions"
                :key="area.value"
                :label="area.label"
                :value="area.value"
              />
            </el-select>
          </el-form-item>
          <el-form-item label="金额" prop="subjectAmount">
            <el-input
              v-model="form.subjectAmountDisplay"
              placeholder="请输入金额"
              class="amount-input"
              @input="handleAmountInput"
              @focus="handleAmountFocus"
              @blur="handleAmountBlur"
            />
          </el-form-item>
          <el-form-item label="相关人员" prop="userId">
            <el-input v-model="form.userName" readonly placeholder="请选择相关人员">
              <el-button slot="append" icon="el-icon-search" @click="handleSelectUser">选择</el-button>
            </el-input>
          </el-form-item>
          <el-form-item label="状态" prop="status">
            <el-radio-group v-model="form.status">
              <el-radio label="0">正常</el-radio>
              <el-radio label="1">停用</el-radio>
            </el-radio-group>
          </el-form-item>
          <el-form-item label="附件" prop="attachments">
            <FileUpload v-model="form.attachments" :limit="5" :fileSize="10" />
          </el-form-item>
          <el-form-item label="备注" prop="remark">
            <el-input v-model="form.remark" type="textarea" :rows="3" placeholder="请输入备注说明" />
          </el-form-item>
        </div>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" :loading="submitLoading" @click="submitForm">确 定</el-button>
        <el-button @click="cancel">取 消</el-button>
      </div>
    </el-dialog>

    <!-- 用户选择弹窗 -->
    <UserSelect ref="userSelect" @confirm="handleUserConfirm" />

    <!-- 详情对话框 -->
    <el-dialog title="科目详情" :visible.sync="detailOpen" width="550px" append-to-body custom-class="dialog-title-left">
      <el-descriptions :column="2" border>
        <el-descriptions-item label="科目编码">{{ detailData.subjectCode }}</el-descriptions-item>
        <el-descriptions-item label="科目类型">
          <dict-tag :options="dict.type.account_subject_type" :value="detailData.subjectType" />
        </el-descriptions-item>
        <el-descriptions-item label="科目名称" :span="2">{{ detailData.subjectName }}</el-descriptions-item>
        <el-descriptions-item label="科目区域" :span="2">{{ formatArea(detailData.subjectArea) }}</el-descriptions-item>
        <el-descriptions-item label="金额">{{ formatAmount(detailData.subjectAmount) }}</el-descriptions-item>
        <el-descriptions-item label="状态">
          <el-tag :type="detailData.status === '0' ? 'success' : 'danger'">
            {{ detailData.status === '0' ? '正常' : '停用' }}
          </el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="相关人员">{{ detailData.userName || '-' }}</el-descriptions-item>
        <el-descriptions-item label="所属部门">{{ detailData.deptName || '-' }}</el-descriptions-item>
        <el-descriptions-item label="附件" :span="2">
          <el-link type="primary" v-if="detailData.attachments" @click="handlePreview(detailData.attachments)">
            查看({{ getFileCount(detailData.attachments) }})
          </el-link>
          <span v-else>-</span>
        </el-descriptions-item>
        <el-descriptions-item label="创建人">{{ detailData.createBy }}</el-descriptions-item>
        <el-descriptions-item label="创建时间">{{ parseTime(detailData.createTime) }}</el-descriptions-item>
        <el-descriptions-item label="备注" :span="2">{{ detailData.remark || '-' }}</el-descriptions-item>
      </el-descriptions>
      <div slot="footer" class="dialog-footer">
        <el-button type="danger" @click="handleDetailDelete" v-hasPermi="['account:subject:remove']">删 除</el-button>
        <el-button type="primary" @click="handleDetailEdit" v-hasPermi="['account:subject:edit']">编 辑</el-button>
        <el-button @click="detailOpen = false">关 闭</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { listSubject, getSubject, delSubject, addSubject, updateSubject, changeStatus } from '@/api/account/subject'
import FileUpload from '@/components/FileUpload'
import UserSelect from '@/components/UserSelect'

// 默认表单数据
const DEFAULT_FORM = {
  subjectId: null,
  subjectType: '',
  subjectCode: '',
  subjectName: '',
  subjectArea: [],
  subjectAmount: 0,
  subjectAmountDisplay: '',
  userId: null,
  userName: '',
  status: '0',
  attachments: '',
  remark: ''
}

// 区域选项
const AREA_OPTIONS = [
  { value: 'west', label: '西区' },
  { value: 'north', label: '北区' },
  { value: 'south', label: '南区' },
  { value: 'east', label: '东区' }
]

export default {
  name: 'Subject',
  components: { FileUpload, UserSelect },
  dicts: ['account_subject_type'],
  data() {
    return {
      // 查询参数
      queryParams: {
        pageNum: 1,
        pageSize: 8,
        subjectName: '',
        subjectType: ''
      },
      // 表单数据
      form: { ...DEFAULT_FORM },
      // 区域选项
      areaOptions: AREA_OPTIONS,
      // 表单验证规则
      rules: {
        subjectType: [{ required: true, message: '请选择科目类型', trigger: 'change' }],
        subjectName: [
          { required: true, message: '请输入科目名称', trigger: 'blur' },
          { min: 2, max: 100, message: '长度在 2 到 100 个字符', trigger: 'blur' }
        ],
        subjectArea: [
          { required: true, message: '请选择科目区域', trigger: 'change', type: 'array' }
        ],
        subjectAmount: [
          { required: true, message: '请输入金额', trigger: 'blur' },
          { type: 'number', min: 0, message: '金额必须大于等于 0', trigger: 'blur' }
        ]
      },
      // 列表数据
      subjectList: [],
      total: 0,
      loading: false,
      // 弹窗控制
      open: false,
      title: '',
      submitLoading: false,
      // 初始表单数据（用于对比是否修改）
      originalForm: null,
      // 选择控制
      single: true,
      multiple: true,
      ids: [],
      // 显示搜索
      showSearch: true,
      // 列配置
      columns: [
        { key: 0, label: '科目ID', visible: false },
        { key: 1, label: '科目类型', visible: true },
        { key: 2, label: '科目编码', visible: true },
        { key: 3, label: '科目名称', visible: true },
        { key: 4, label: '科目区域', visible: true },
        { key: 5, label: '金额', visible: true },
        { key: 6, label: '相关人员', visible: true },
        { key: 7, label: '附件', visible: true },
        { key: 8, label: '状态', visible: true }
      ],
      // 详情弹窗
      detailOpen: false,
      detailData: {}
    }
  },
  created() {
    this.getList()
  },
  methods: {
    /** 查询列表 */
    getList() {
      this.loading = true
      listSubject(this.queryParams).then(response => {
        this.subjectList = response.rows
        this.total = response.total
      }).finally(() => {
        this.loading = false
      })
    },
    /** 取消按钮 */
    cancel() {
      if (this.hasFormChanged()) {
        this.$confirm('确认关闭？未保存的内容将会丢失', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.open = false
          this.reset()
        }).catch()
      } else {
        this.open = false
        this.reset()
      }
    },
    /** 重置表单 */
    reset() {
      this.form = { ...DEFAULT_FORM }
      this.originalForm = null
      this.resetForm('form')
    },
    /** 搜索按钮操作 */
    handleQuery() {
      this.queryParams.pageNum = 1
      this.getList()
    },
    /** 重置按钮操作 */
    resetQuery() {
      this.resetForm('queryForm')
      this.handleQuery()
    },
    /** 多选框选中数据 */
    handleSelectionChange(selection) {
      this.ids = selection.map(item => item.subjectId || item.subject_id)
      this.single = selection.length !== 1
      this.multiple = !selection.length
    },
    /** 新增按钮操作 */
    handleAdd() {
      this.reset()
      this.form.subjectCode = this.generateSubjectCode()
      this.originalForm = JSON.parse(JSON.stringify(this.form))
      this.open = true
      this.title = '新增科目'
    },
    /** 修改按钮操作 */
    handleUpdate(row) {
      this.reset()
      // 兼容从操作列点击（传入row）和从工具栏点击（无参数，使用ids）
      let id
      if (row && (row.subjectId || row.subject_id)) {
        id = row.subjectId || row.subject_id
      } else if (this.ids.length > 0) {
        id = this.ids[0]
      }
      if (!id) {
        this.$modal.msgWarning('请选择要修改的记录')
        return
      }
      getSubject(id).then(response => {
        this.form = {
          ...response.data,
          subjectArea: this.areaToArray(response.data.subjectArea)
        }
        // 格式化金额显示
        if (this.form.subjectAmount) {
          this.form.subjectAmountDisplay = this.form.subjectAmount.toLocaleString('zh-CN', {
            minimumFractionDigits: 2,
            maximumFractionDigits: 2
          })
        }
        this.originalForm = JSON.parse(JSON.stringify(this.form))
        this.open = true
        this.title = '修改科目'
      })
    },
    /** 打开用户选择弹窗 */
    handleSelectUser() {
      this.$refs.userSelect.show()
    },
    /** 用户选择确认回调 */
    handleUserConfirm(user) {
      if (user) {
        this.form.userId = user.userId
        this.form.userName = user.nickName
      }
    },
    /** 查看详情 */
    handleDetail(row) {
      const id = row.subjectId || row.subject_id
      if (!id) {
        this.$modal.msgWarning('数据异常，无法查看详情')
        return
      }
      getSubject(id).then(response => {
        this.detailData = response.data
        this.detailOpen = true
      })
    },
    /** 详情页编辑 */
    handleDetailEdit() {
      const id = this.detailData.subjectId
      if (!id) {
        this.$modal.msgWarning('数据异常，无法编辑')
        return
      }
      this.reset()
      getSubject(id).then(response => {
        this.form = {
          ...response.data,
          subjectArea: this.areaToArray(response.data.subjectArea)
        }
        // 格式化金额显示
        if (this.form.subjectAmount) {
          this.form.subjectAmountDisplay = this.form.subjectAmount.toLocaleString('zh-CN', {
            minimumFractionDigits: 2,
            maximumFractionDigits: 2
          })
        }
        this.originalForm = JSON.parse(JSON.stringify(this.form))
        this.open = true
        this.title = '修改科目'
        this.detailOpen = false
      })
    },
    /** 详情页删除 */
    handleDetailDelete() {
      const id = this.detailData.subjectId || this.detailData.subject_id
      const name = this.detailData.subjectName || `ID: ${id}`

      if (!id) {
        this.$modal.msgWarning('数据异常，无法删除')
        return
      }

      this.$modal.confirm(`是否确认删除科目"${name}"？`).then(() => {
        return delSubject([id])
      }).then(() => {
        this.detailOpen = false
        this.getList()
        this.$modal.msgSuccess('删除成功')
      }).catch(() => {})
    },
    /** 提交按钮 */
    submitForm() {
      this.$refs.form.validate(valid => {
        if (!valid) return

        this.submitLoading = true
        const { subjectAmountDisplay, subjectId, ...rest } = this.form
        const submitData = {
          ...rest,
          subjectArea: this.areaToString(this.form.subjectArea)
        }
        // 修改时才传 subjectId
        if (this.form.subjectId) {
          submitData.subjectId = this.form.subjectId
        }
        const request = this.form.subjectId ? updateSubject(submitData) : addSubject(submitData)
        const msg = this.form.subjectId ? '修改成功' : '新增成功'

        request.then(() => {
          this.$modal.msgSuccess(msg)
          this.open = false
          this.getList()
        }).finally(() => {
          this.submitLoading = false
        })
      })
    },
    /** 删除按钮操作 */
    handleDelete(row) {
      let ids = []
      let names = ''

      if (row) {
        // 从操作列点击删除，兼容多种字段名
        const id = row.subjectId ?? row.subject_id ?? row.SUBJECTID ?? row.SUBJECT_ID
        const name = row.subjectName ?? row.subject_name ?? row.SUBJECTNAME ?? row.SUBJECT_NAME

        if (!id) {
          this.$modal.msgWarning('数据异常，无法获取科目ID')
          return
        }

        ids = [id]
        names = name || `ID: ${id}`
      } else {
        // 从工具栏点击删除（批量删除）
        ids = this.ids
        if (!ids || ids.length === 0) {
          this.$modal.msgWarning('请选择要删除的记录')
          return
        }
        names = `${ids.length}条数据`
      }

      this.$modal.confirm(`是否确认删除科目"${names}"？`).then(() => {
        return delSubject(ids)
      }).then(() => {
        this.getList()
        this.$modal.msgSuccess('删除成功')
      }).catch(() => {})
    },
    /** 导出按钮操作 */
    handleExport() {
      this.download('account/subject/export', {
        ...this.queryParams
      }, `subject_${new Date().getTime()}.xlsx`)
    },
    /** 状态切换 */
    handleStatusChange(row) {
      const text = row.status === '0' ? '启用' : '停用'
      changeStatus(row.subjectId || row.subject_id, row.status).then(() => {
        this.$modal.msgSuccess(`${text}成功`)
      }).catch(() => {
        row.status = row.status === '0' ? '1' : '0'
      })
    },
    /** 预览附件 */
    handlePreview(attachments) {
      if (!attachments) return
      const files = attachments.split(',')
      files.forEach(url => {
        window.open(process.env.VUE_APP_BASE_API + url, '_blank')
      })
    },
    /** 格式化金额 */
    formatAmount(amount) {
      if (amount == null) return '-'
      return amount.toLocaleString('zh-CN', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    },
    /** 获取文件数量 */
    getFileCount(attachments) {
      if (!attachments) return 0
      return attachments.split(',').filter(Boolean).length
    },
    /** 生成科目编码 (K + yyyyMMddHHmmss) */
    generateSubjectCode() {
      const now = new Date()
      const year = now.getFullYear()
      const month = String(now.getMonth() + 1).padStart(2, '0')
      const day = String(now.getDate()).padStart(2, '0')
      const hour = String(now.getHours()).padStart(2, '0')
      const minute = String(now.getMinutes()).padStart(2, '0')
      const second = String(now.getSeconds()).padStart(2, '0')
      return `K${year}${month}${day}${hour}${minute}${second}`
    },
    /** 格式化区域显示 */
    formatArea(area) {
      if (!area) return '-'
      const areas = Array.isArray(area) ? area : area.split(',')
      const labels = areas.map(val => {
        const found = AREA_OPTIONS.find(opt => opt.value === val)
        return found ? found.label : val
      })
      return labels.join('、')
    },
    /** 金额输入处理（只允许数字和小数点） */
    handleAmountInput(value) {
      // 只保留数字和一个小数点
      let val = value.replace(/[^\d.]/g, '')
      // 处理多个小数点，只保留第一个
      const dotIndex = val.indexOf('.')
      if (dotIndex !== -1) {
        val = val.substring(0, dotIndex + 1) + val.substring(dotIndex + 1).replace(/\./g, '')
      }
      // 限制小数点后最多2位
      if (dotIndex !== -1) {
        const parts = val.split('.')
        if (parts[1] && parts[1].length > 2) {
          val = parts[0] + '.' + parts[1].substring(0, 2)
        }
      }
      this.form.subjectAmountDisplay = val
      this.form.subjectAmount = val ? parseFloat(val) : 0
    },
    /** 金额聚焦时显示原始数值 */
    handleAmountFocus() {
      if (this.form.subjectAmount) {
        this.form.subjectAmountDisplay = String(this.form.subjectAmount)
      }
    },
    /** 金额失焦时格式化为千分位 */
    handleAmountBlur() {
      if (this.form.subjectAmount) {
        this.form.subjectAmountDisplay = this.form.subjectAmount.toLocaleString('zh-CN', {
          minimumFractionDigits: 2,
          maximumFractionDigits: 2
        })
      }
    },
    /** 字符串转数组（用于编辑回显） */
    areaToArray(area) {
      if (!area) return []
      return Array.isArray(area) ? area : area.split(',')
    },
    /** 数组转字符串（用于提交） */
    areaToString(area) {
      if (!area) return ''
      return Array.isArray(area) ? area.join(',') : area
    },
    /** 检查表单是否有修改 */
    hasFormChanged() {
      if (!this.originalForm) return false
      const currentForm = JSON.parse(JSON.stringify(this.form))
      // 比较每个字段
      const keys = Object.keys(this.originalForm)
      for (const key of keys) {
        const original = this.originalForm[key]
        const current = currentForm[key]
        // 处理数组类型（如 subjectArea）
        if (Array.isArray(original) || Array.isArray(current)) {
          const arr1 = Array.isArray(original) ? original : []
          const arr2 = Array.isArray(current) ? current : []
          if (JSON.stringify(arr1) !== JSON.stringify(arr2)) {
            return true
          }
        } else if (original !== current) {
          return true
        }
      }
      return false
    }
  }
}
</script>

<style scoped>
.form-scroll-container {
  max-height: 400px;
  overflow-y: auto;
  padding-right: 10px;
}
.form-scroll-container::-webkit-scrollbar {
  width: 6px;
}
.form-scroll-container::-webkit-scrollbar-thumb {
  background-color: #dcdfe6;
  border-radius: 3px;
}
.form-scroll-container::-webkit-scrollbar-track {
  background-color: #f5f7fa;
}
.link-ellipsis {
  display: inline-block;
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  vertical-align: bottom;
}
.amount-text {
  display: block;
  text-align: right;
}
.amount-input ::v-deep .el-input__inner {
  text-align: right;
}
</style>

<style>
.dialog-title-left .el-dialog__title {
  float: left;
}
.dialog-title-left .el-form-item__label {
  text-align: left;
  padding-left: 0;
}
.dialog-title-left .el-form-item__label::before {
  display: none;
}
.dialog-title-left .el-form-item__label::after {
  content: '*';
  color: #f56c6c;
  margin-left: 4px;
}
.dialog-title-left .el-form-item.is-required:not(.is-no-asterisk) .el-form-item__label::after {
  content: '*';
}
.dialog-title-left .el-form-item:not(.is-required) .el-form-item__label::after {
  content: '';
}
</style>
