<template>
  <div class="mod-user">
    <el-form :inline="true" :model="dataForm" @keyup.enter.native="getDataList()">
      <el-form-item>
        <el-input v-model="dataForm.name" placeholder="姓名" clearable></el-input>
      </el-form-item>
      <el-form-item>
        <el-button @click="getDataList()">查询</el-button>
        <el-button v-if="isAuth('generator:profile:save')" type="primary" @click="addOrUpdateHandle()">新增</el-button>
        <el-button type="success" @click="exportJson2Excel(dataList, 'profile.xlsx')">导出Excel</el-button>
        <el-button v-if="isAuth('generator:profile:delete')" type="danger" @click="deleteHandle()" :disabled="dataListSelections.length <= 0">批量删除</el-button>
      </el-form-item>
    </el-form>
    <el-table
      id="export-table"
      :data="dataList"
      border
      v-loading="dataListLoading"
      @selection-change="selectionChangeHandle"
      style="width: 100%;">
      <el-table-column
        type="selection"
        header-align="center"
        align="center"
        width="50">
      </el-table-column>
      <el-table-column
        prop="id"
        header-align="center"
        align="center"
        width="80"
        label="ID">
      </el-table-column>
      <el-table-column
        prop="name"
        header-align="center"
        align="center"
        label="姓名">
      </el-table-column>
      <el-table-column
        prop="phone"
        header-align="center"
        align="center"
        label="手机号">
      </el-table-column>
      <el-table-column
        prop="address"
        header-align="center"
        align="center"
        label="地址">
      </el-table-column>
      <el-table-column
        prop="remarks"
        header-align="center"
        align="center"
        width="200"
        label="备注">
      </el-table-column>
      <el-table-column
        fixed="right"
        header-align="center"
        align="center"
        width="150"
        label="操作">
        <template slot-scope="scope">
          <el-button v-if="isAuth('generator:profile:update')" type="text" size="small" @click="addOrUpdateHandle(scope.row.id)">修改</el-button>
          <el-button v-if="isAuth('generator:profile:delete')" type="text" size="small" @click="deleteHandle(scope.row.id)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination
      @size-change="sizeChangeHandle"
      @current-change="currentChangeHandle"
      :current-page="pageIndex"
      :page-sizes="[10, 20, 50, 100]"
      :page-size="pageSize"
      :total="totalPage"
      layout="total, sizes, prev, pager, next, jumper">
    </el-pagination>
    <!-- 弹窗, 新增 / 修改 -->
    <add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="getDataList"></add-or-update>
  </div>
</template>

<script>
  import FileSaver from 'file-saver'
  import XLSX from 'xlsx'
  import AddOrUpdate from './profile-add-or-update'
  export default {
    data () {
      return {
        dataForm: {
          name: ''
        },
        dataList: [],
        pageIndex: 1,
        pageSize: 10,
        totalPage: 0,
        dataListLoading: false,
        dataListSelections: [],
        addOrUpdateVisible: false
      }
    },
    components: {
      AddOrUpdate
    },
    activated () {
      this.getDataList()
    },
    methods: {
      // 获取数据列表
      getDataList () {
        this.dataListLoading = true
        this.$http({
          url: this.$http.adornUrl('/generator/profile/list'),
          method: 'get',
          params: this.$http.adornParams({
            'page': this.pageIndex,
            'limit': this.pageSize,
            'name': this.dataForm.name
          })
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.dataList = data.page.list
            this.totalPage = data.page.totalCount
          } else {
            this.dataList = []
            this.totalPage = 0
          }
          this.dataListLoading = false
        })
      },
      // 每页数
      sizeChangeHandle (val) {
        this.pageSize = val
        this.pageIndex = 1
        this.getDataList()
      },
      // 当前页
      currentChangeHandle (val) {
        this.pageIndex = val
        this.getDataList()
      },
      // 多选
      selectionChangeHandle (val) {
        this.dataListSelections = val
      },
      // 新增 / 修改
      addOrUpdateHandle (id) {
        this.addOrUpdateVisible = true
        this.$nextTick(() => {
          this.$refs.addOrUpdate.init(id)
        })
      },
      // 删除
      deleteHandle (id) {
        var ids = id ? [id] : this.dataListSelections.map(item => {
          return item.id
        })
        this.$confirm(`确定对[id=${ids.join(',')}]进行[${id ? '删除' : '批量删除'}]操作?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/generator/profile/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
            if (data && data.code === 0) {
              this.$message({
                message: '操作成功',
                type: 'success',
                duration: 1500,
                onClose: () => {
                  this.getDataList()
                }
              })
            } else {
              this.$message.error(data.msg)
            }
          })
        }).catch(() => {
        })
      },
      // 定义导出Excel表格事件
      // exportExcel () {
      //   /* 从表生成工作簿对象 */
      //   var wb = XLSX.utils.table_to_book(document.querySelector('#export-table'))
      //   let fix = document.querySelector('#export-table > .el-table__fixed-right')
      //
      //   if (fix) {
      //     wb = XLSX.utils.table_to_book(document.querySelector('#export-table').removeChild(fix))
      //     document.querySelector('#export-table').appendChild(fix)
      //   } else {
      //     wb = XLSX.utils.table_to_book(document.querySelector('#export-table'))
      //   }
      //
      //   let wbout = XLSX.write(wb, {bookType: 'xlsx', bookSST: true, type: 'array'})
      //   try {
      //     FileSaver.saveAs(new Blob([wbout], {type: 'application/octet-stream'}), 'profile.xlsx')
      //   } catch (e) {
      //     if (typeof console !== 'undefined') console.log(e, wbout)
      //   }
      //   return wbout
      // },
      exportJson2Excel (data, fileName, sheetName) {
        const sheet = XLSX.utils.json_to_sheet(data)
        FileSaver.saveAs(new Blob([this.sheet2Blob(sheet, sheetName)], {type: 'application/octet-stream;charset=utf-8'}), fileName)
      },
      sheet2Blob (sheet, sheetName) {
        sheetName = sheetName || 'sheet1'
        const workbook = {
          SheetNames: [sheetName],
          Sheets: {}
        }
        workbook.Sheets[sheetName] = sheet
        // 生成excel的配置项
        const wopts = {
          bookType: 'xlsx', // 要生成的文件类型
          bookSST: false, // 是否生成Shared String Table，官方解释是，如果开启生成速度会下降，但在低版本IOS设备上有更好的兼容性
          type: 'binary'
        }
        const wbout = XLSX.write(workbook, wopts)
        const blob = new Blob([this.string2ArrayBuffer(wbout)], {type: 'application/octet-stream'})
        return blob
      },
      // 字符串转ArrayBuffer
      string2ArrayBuffer (s) {
        const buf = new ArrayBuffer(s.length)
        const view = new Uint8Array(buf)
        for (let i = 0; i < s.length; ++i) {
          view[i] = s.charCodeAt(i) & 0xFF
        }
        console.log(s)
        return buf
      }
    }
  }
</script>
