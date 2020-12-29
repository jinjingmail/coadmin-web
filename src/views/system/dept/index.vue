<template>
  <div class="app-container">
    <el-row :gutter="20">
      <!--侧边机构数据-->
      <el-col :xs="9" :sm="6" :md="5" :lg="4" :xl="4">
        <div class="head-container">
          <el-input
            v-model="filterDeptName"
            clearable
            size="small"
            placeholder="输入机构名称搜索"
            prefix-icon="el-icon-search"
            class="filter-item"
          />
        </div>
        <el-tree
          ref="treeDept"
          :data="deptDatas"
          :load="getDeptDatas"
          :props="defaultProps"
          :expand-on-click-node="false"
          :filter-node-method="treeDeptFilter"
          @node-click="handleTreeDeptNodeClick"
        />
      </el-col>

      <!--用户数据-->
      <el-col :xs="15" :sm="18" :md="19" :lg="20" :xl="20">
        <!--工具栏-->
        <div class="head-container">
          <div v-if="crud.props.searchToggle">
            <!-- 搜索 -->
            <el-select v-model="query.enabled" clearable size="small" placeholder="状态" class="filter-item" style="width: 90px" @change="crud.toQuery">
              <el-option v-for="item in enabledTypeOptions" :key="item.key" :label="item.display_name" :value="item.key" />
            </el-select>
            <rrOperation />
          </div>
          <crudOperation :permission="permission" />
        </div>
        <!--表单组件-->
        <el-dialog append-to-body :close-on-click-modal="false" :before-close="crud.cancelCU" :visible="crud.status.cu > 0" :title="crud.status.title" width="500px">
          <el-form ref="form" inline :model="form" :rules="rules" size="small" label-width="80px">
            <el-form-item label="机构名称" prop="name">
              <el-input v-model="form.name" style="width: 370px;" />
            </el-form-item>
            <el-form-item label="排序" prop="sort">
              <el-input-number
                v-model.number="form.sort"
                :min="0"
                :max="999"
                controls-position="right"
                style="width: 370px;"
              />
            </el-form-item>
            <el-form-item label="顶级机构">
              <el-radio-group v-model="form.isTop" style="width: 140px">
                <el-radio label="1">是</el-radio>
                <el-radio label="0">否</el-radio>
              </el-radio-group>
            </el-form-item>
            <el-form-item label="状态" prop="enabled">
              <el-radio v-for="item in dict.dept_status" :key="item.id" v-model="form.enabled" :label="item.value">{{ item.label }}</el-radio>
            </el-form-item>
            <el-form-item v-if="form.isTop === '0'" style="margin-bottom: 0;" label="上级机构" prop="pid">
              <treeselect
                v-model="form.pid"
                :load-options="loadDepts"
                :options="depts"
                style="width: 370px;"
                placeholder="选择上级类目"
              />
            </el-form-item>
          </el-form>
          <div slot="footer" class="dialog-footer">
            <el-button type="text" @click="crud.cancelCU">取消</el-button>
            <el-button :loading="crud.status.cu === 2" type="primary" @click="crud.submitCU">确认</el-button>
          </div>
        </el-dialog>
        <!--表格渲染-->
        <el-table
          ref="table"
          v-loading="crud.loading"
          :data="crud.data"
          lazy
          :load="lazyLoad"
          :tree-props="{children: 'children', hasChildren: 'hasChildren'}"
          row-key="id"
          @select="crud.selectChange"
          @select-all="crud.selectAllChange"
          @selection-change="crud.selectionChangeHandler"
        >
          <el-table-column :selectable="checkboxT" type="selection" width="55" />
          <el-table-column label="名称" prop="name" />
          <el-table-column label="排序" prop="sort" />
          <el-table-column label="状态" align="center" prop="enabled">
            <template slot-scope="scope">
              <el-switch
                v-model="scope.row.enabled"
                :disabled="scope.row.id === 1"
                active-color="#409EFF"
                inactive-color="#F56C6C"
                @change="changeEnabled(scope.row, scope.row.enabled,)"
              />
            </template>
          </el-table-column>
          <el-table-column prop="createTime" label="创建日期">
            <template slot-scope="scope">
              <span>{{ parseTime(scope.row.createTime) }}</span>
            </template>
          </el-table-column>
          <el-table-column v-permission="['admin','dept:edit','dept:del']" label="操作" width="130px" align="center" fixed="right">
            <template slot-scope="scope">
              <udOperation
                :data="scope.row"
                :permission="permission"
                :disabled-dle="scope.row.id === 1"
                msg="确定删除吗,如果存在下级节点则一并删除，此操作不能撤销！"
              />
            </template>
          </el-table-column>
        </el-table>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import crudDept, { getDepts } from '@/api/system/dept'
import Treeselect from '@riophae/vue-treeselect'
import '@riophae/vue-treeselect/dist/vue-treeselect.css'
import { LOAD_CHILDREN_OPTIONS } from '@riophae/vue-treeselect'
import CRUD, { presenter, header, form, crud } from '@crud/crud'
import rrOperation from '@crud/RR.operation'
import crudOperation from '@crud/CRUD.operation'
import udOperation from '@crud/UD.operation'

const defaultForm = { id: null, name: null, isTop: '0', pid: null, sort: 999, enabled: 'true' }
export default {
  name: 'Dept',
  components: { Treeselect, crudOperation, rrOperation, udOperation },
  cruds() {
    return CRUD({ title: '机构', url: 'api/dept', crudMethod: { ...crudDept }})
  },
  mixins: [presenter(), header(), form(defaultForm), crud()],
  // 设置数据字典
  dicts: ['dept_status'],
  data() {
    return {
      depts: [],
      deptDatas: [],
      filterDeptName: '',
      defaultProps: { children: 'children', label: 'name', isLeaf: 'treeLeaf' },
      rules: {
        name: [
          { required: true, message: '请输入名称', trigger: 'blur' }
        ],
        sort: [
          { required: true, message: '请输入排序', trigger: 'blur', type: 'number' }
        ]
      },
      permission: {
        add: ['admin', 'dept:add'],
        edit: ['admin', 'dept:edit'],
        del: ['admin', 'dept:del']
      },
      enabledTypeOptions: [
        { key: 'true', display_name: '正常' },
        { key: 'false', display_name: '禁用' }
      ]
    }
  },
  watch: {
    filterDeptName(val) {
      this.$refs.treeDept.filter(val)
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.getDeptDatas()
    })
  },
  methods: {
    lazyLoad(tree, treeNode, resolve) {
      const a = this.crud.data
      resolve(this.findChildren(tree.id, a))
    },
    findChildren(id, a) {
      for (var i = 0; i < a.length; ++i) {
        if (a[i].id === id) {
          return (a[i].children)
        } else {
          if (a[i].children && a[i].children.length > 0) {
            var x = this.findChildren(id, a[i].children)
            if (x && x.length && x.length > 0) {
              return x
            }
          }
        }
      }
      return []
    },
    // 获取左侧机构数据
    getDeptDatas(node, resolve) {
      const sort = 'id,desc'
      const params = { sort: sort }
      if (typeof node !== 'object') {
        if (node) {
          params['name'] = node
        }
      } else if (node.level !== 0) {
        params['pid'] = node.data.id
      }
      setTimeout(() => {
        getDepts(params).then(res => {
          if (resolve) {
            resolve(res.content)
          } else {
            this.deptDatas = res.content
          }
        })
      }, 100)
    },
    // 切换机构
    handleTreeDeptNodeClick(data) {
      if (data.pid === 0) {
        this.query.pids = data.id
        this.query.treeId = data.id
      } else {
        this.query.pids = data.id
        this.query.treeId = data.id
      }
      this.crud.toQuery()
    },
    treeDeptFilter(value, data) {
      if (!value) return true
      return data.label.indexOf(value) !== -1
    },
    // 新增与编辑前做的操作
    [CRUD.HOOK.afterToCU](crud, form) {
      if (form.pid !== null) {
        form.isTop = '0'
      } else if (form.id !== null) {
        form.isTop = '1'
      }
      form.enabled = `${form.enabled}`
      if (form.id != null) {
        this.getSupDepts(form.id)
      } else {
        this.getDepts()
      }
    },
    getSupDepts(id) {
      crudDept.getDeptSuperior(Array.of(id)).then(res => {
        const date = res.content
        this.depts = date
      })
    },
    getDepts() {
      crudDept.getDepts({ enabled: true }).then(res => {
        this.depts = res.content
      })
    },
    // 获取弹窗内机构数据
    loadDepts({ action, parentNode, callback }) {
      if (action === LOAD_CHILDREN_OPTIONS) {
        crudDept.getDepts({ enabled: true, pid: parentNode.id }).then(res => {
          parentNode.children = res.content
          setTimeout(() => {
            callback()
          }, 100)
        })
      }
    },
    // 提交前的验证
    [CRUD.HOOK.afterValidateCU]() {
      if (this.form.pid !== null && this.form.pid === this.form.id) {
        this.$message({
          message: '上级机构不能为空',
          type: 'warning'
        })
        return false
      }
      if (this.form.isTop === '1') {
        this.form.pid = null
      }
      console.log('afterValidateCU:', this.form)
      this.form.children = null
      console.log('afterValidateCU2:', this.form)
      return true
    },
    // 改变状态
    changeEnabled(data, val) {
      this.$confirm('此操作将 "' + this.dict.label.dept_status[val] + '" ' + data.name + '机构, 是否继续？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        crudDept.edit(data).then(res => {
          this.crud.notify(this.dict.label.dept_status[val] + '成功', CRUD.NOTIFICATION_TYPE.SUCCESS)
        }).catch(err => {
          data.enabled = !data.enabled
          console.log(err.response.data.message)
        })
      }).catch(() => {
        data.enabled = !data.enabled
      })
    },
    checkboxT(row, rowIndex) {
      return row.id !== 1
    }
  }
}
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
 ::v-deep .vue-treeselect__control,::v-deep .vue-treeselect__placeholder,::v-deep .vue-treeselect__single-value {
    height: 30px;
    line-height: 30px;
  }
</style>
<style rel="stylesheet/scss" lang="scss" scoped>
 ::v-deep .el-input-number .el-input__inner {
    text-align: left;
  }
</style>
