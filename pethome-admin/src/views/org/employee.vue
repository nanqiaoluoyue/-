<template>
  <section>
    <!--工具条-->
    <el-col :span="24" class="toolbar" style="padding-bottom: 0px;">
      <el-form :inline="true" :model="query">
        <el-form-item>
          <el-input v-model="query.keyword" placeholder="关键字"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" v-on:click="getEmployee" icon="el-icon-search">查询</el-button>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="handleAdd" icon="el-icon-circle-plus-outline">新增</el-button>
        </el-form-item>
      </el-form>
    </el-col>

    <!--列表-->
    <el-table :data="employees" highlight-current-row v-loading="listLoading" @selection-change="selsChange" style="width: 100%;">
      <el-table-column type="selection" width="50">
      </el-table-column>
      <el-table-column type="index" label="#" width="50">
      </el-table-column>
      <el-table-column prop="username" label="用户名" width="100" sortable>
      </el-table-column>
      <el-table-column prop="age" label="年龄" width="80" sortable>
      </el-table-column>
      <el-table-column prop="password" label="密码" width="120" sortable>
      </el-table-column>
      <el-table-column prop="email" label="邮箱" width="180" sortable>
      </el-table-column>
      <el-table-column prop="phone" label="电话" width="150" sortable>
      </el-table-column>
      <el-table-column prop="state" label="状态" width="100" sortable>
        <template scope="scope">
          <span v-if="scope.row.state=='1'" style="color: green">正常</span>
          <span v-if="scope.row.state=='0'" style="color: red">停用</span>
        </template>
      </el-table-column>
      <el-table-column prop="department.name" label="部门" width="100" sortable>
      </el-table-column>
      <el-table-column label="操作" width="min-180" >
        <template scope="scope">
          <el-button size="small" @click="handleEdit(scope.$index, scope.row)" icon="el-icon-edit">编辑</el-button>
          <el-button type="danger" size="small" @click="handleDel(scope.$index, scope.row)" icon="el-icon-delete">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <!--工具条-->
    <el-col :span="24" class="toolbar">
      <el-button type="danger" @click="batchRemove" :disabled="this.sels.length===0" icon="el-icon-delete">批量删除</el-button>
      <el-pagination layout="prev, pager, next" @current-change="handleCurrentChange" :page-size="10" :total="total" style="float:right;">
      </el-pagination>
    </el-col>

    <!--编辑界面  :visible.sync="dialogVisible"-->
    <el-dialog :title="title" :visible.sync="employeeFormVisible" :close-on-click-modal="false">
      <el-form :model="employeeForm" label-width="80px" :rules="employeeFormRules" ref="employeeForm">
        <el-form-item label="用户名" prop="username">
          <el-input v-model="employeeForm.username" auto-complete="off"></el-input>
        </el-form-item>
        <el-form-item label="密码" prop="password">
          <el-input v-model="employeeForm.password" auto-complete="off"></el-input>
        </el-form-item>
        <el-form-item label="年龄" prop="age">
          <el-input v-model="employeeForm.age" auto-complete="off"></el-input>
        </el-form-item>
        <el-form-item label="邮箱" prop="email">
          <el-input v-model="employeeForm.email" auto-complete="off"></el-input>
        </el-form-item>
        <el-form-item label="电话" prop="phone">
          <el-input v-model="employeeForm.phone" auto-complete="off"></el-input>
        </el-form-item>
        <el-form-item label="状态">
          <el-radio-group v-model="employeeForm.state">
            <el-radio class="radio" :label="1">正常</el-radio>
            <el-radio class="radio" :label="0">停用</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="部门" prop="department">
          <el-select v-model="employeeForm.department" value-key="id" placeholder="请选择">
            <el-option
                v-for="item in Departments"
                :key="item.id"
                :label="item.name"
                :value="item">
              <span style="float: left">{{ item.name }}</span>
              <!--<span style="float: right; color: #8492a6; font-size: 13px">{{ item.email }}</span>-->
            </el-option>
          </el-select>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click.native="employeeFormVisible = false">取消</el-button>
        <el-button type="primary" @click.native="editSubmit" :loading="editLoading">提交</el-button>
      </div>
    </el-dialog>

  </section>
</template>

<script>
import util from '../../common/js/util'
//import NProgress from 'nprogress'
import { getUserListPage, removeUser, batchRemoveUser, editUser, addUser } from '../../api/api';

export default {
  data() {
    return {
      query: {
        keyword: ''
      },
      employees: [],
      total: 0,
      page: 1,
      listLoading: false,
      sels: [],//列表选中列
      parentId:null,

      title: '',
      // employees: [],//员工信息
      Departments: [],//部门

      employeeFormVisible: false,//编辑界面是否显示
      editLoading: false,
      employeeFormRules: {
        name: [
          { required: true, message: '请输入名称', trigger: 'blur' }
        ]
      },
      //编辑界面数据
      employeeForm: {
        id: null,
        username: '',
        email: '',
        phone: '',
        password: '',
        age: '',
        state: '',
        department: null
      },
    }
  },
  methods: {
    //获取所有部门
    getDepartment(){
      this.$http.get("/department")
          .then(result=>{
            this.Departments = result.data
          })
          .catch(result=>{

          })
    },
    handleCurrentChange(val) {
      this.page = val;
      this.getEmployee();
    },

    //获取用户列表
    getEmployee() {
      let para = {
        currentPage: this.page,
        keyword: this.query.keyword
      };
      this.listLoading = true;
      //NProgress.start();
      this.$http.post("/employee",para)
          .then(result=>{
            this.total = result.data.total;
            this.employees = result.data.data;
            this.listLoading = false;
          })
          .catch(result=>{
            alert(result)
          })
    },

    //删除
    handleDel: function (index, row) {
      this.$confirm('确认删除该记录吗?', '提示', {
        type: 'warning'
      }).then(() => {
        this.listLoading = true;
        var url = "/employee/"+row.id
        this.$http.delete(url)
            .then(result=>{
              this.listLoading = false;
              if(result.data.success){
                this.$message({
                  message: '删除成功',
                  type: 'success'
                });
                this.getEmployee();
              }else {
                this.listLoading = false;
                this.$message({
                  message: result.data.message,
                  type: 'error'
                });
              }
            })
            .catch(result=>{
              this.listLoading = false;
              this.$message({
                message: result.data.message,
                type: 'error'
              });
            })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        });
      });
    },
    //显示编辑界面
    handleEdit: function (index, row) {
      // this.getEmployee();
      this.getDepartment();
      this.parentId = row.parent?row.parent.id:'';
      this.employeeFormVisible = true;
      this.employeeForm = Object.assign({}, row);
    },
    //显示新增界面
    handleAdd: function () {
      this.getDepartment();
      this.employeeForm = {
        id: null,
        username: '',
        email: '',
        phone: '',
        password: '',
        age: '',
        state: 1,
        department: null
      };
      this.employeeFormVisible = true;
    },
    //编辑
    editSubmit: function () {
      this.$refs.employeeForm.validate((valid) => {
        if (valid) {
          this.$confirm('确认提交吗？', '提示', {}).then(() => {
            this.editLoading = true;
            this.$http.put("/employee",this.employeeForm)
                .then(result=>{
                  if(result.data.success){
                    this.editLoading = false;
                    this.$message({
                      message: '提交成功',
                      type: 'success'
                    });
                    this.$refs['employeeForm'].resetFields();
                    this.getEmployee();
                  }else{
                    this.editLoading = false;
                    this.$message({
                      message: result.data.message,
                      type: 'error'
                    });
                  }
                  this.employeeFormVisible = false;
                })
                .catch(result=>{
                  this.editLoading = false;
                  this.$message({
                    message: result.data.message,
                    type: 'error'
                  });
                  this.employeeFormVisible = false;
                })
          });
        }
      });
    },
    //多选
    selsChange: function (sels) {
      this.sels = sels;
    },
    //批量删除
    batchRemove: function () {
      var ids = this.sels.map(item => item.id);
      this.$confirm('确认删除选中记录吗？', '提示', {
        type: 'warning'
      }).then(() => {
        this.listLoading = true;
        this.$http.patch("/employee/bulk",ids)
            .then(result=>{
              if(result.data.success){
                this.listLoading = false;
                this.$message({
                  message: '删除成功',
                  type: 'success'
                });
                this.getEmployee();
              }else {
                this.listLoading = false;
                this.$message({
                  message: result.data.message,
                  type: 'error'
                });
              }
            })
            .catch(result=>{
              this.listLoading = false;
              this.$message({
                message: result.data.message,
                type: 'error'
              });
            })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        });
      });
    }
  },
  mounted() {
    this.getEmployee();
  }
}
</script>

<style scoped>
</style>