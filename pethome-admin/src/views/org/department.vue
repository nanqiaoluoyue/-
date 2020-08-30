<template>
	<section>
		<!--工具条-->
		<el-col :span="24" class="toolbar" style="padding-bottom: 0px;">
			<el-form :inline="true" :model="query">
				<el-form-item>
					<el-input v-model="query.keyword" placeholder="关键字"></el-input>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" v-on:click="getDepartments" icon="el-icon-search">查询</el-button>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" @click="handleAdd" icon="el-icon-circle-plus-outline">新增</el-button>
				</el-form-item>
			</el-form>
		</el-col>

		<!--列表-->
		<el-table :data="departments" highlight-current-row v-loading="listLoading" @selection-change="selsChange" style="width: 100%;">
			<el-table-column type="selection" width="55">
			</el-table-column>
			<el-table-column type="index" width="60">
			</el-table-column>
			<el-table-column prop="name" label="部门名称 " width="150" sortable>
			</el-table-column>
			<el-table-column prop="sn" label="标识" width="150" sortable>
			</el-table-column>
			<el-table-column prop="dirPath" label="路径" width="120" sortable>
			</el-table-column>
			<el-table-column prop="state" label="状态" width="120" sortable>
        <template scope="scope">
          <span v-if="scope.row.state=='-1'" style="color: red">停用</span>
          <span v-if="scope.row.state=='0'" style="color: green">正常</span>
        </template>
			</el-table-column>
      <el-table-column prop="manager.username" label="经理" width="130" sortable>
      </el-table-column>
			<el-table-column prop="parent.name" label="上级部门" width="130" sortable>
			</el-table-column>
			<el-table-column label="操作" width="min-180">
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
		<el-dialog :title="title" :visible.sync="departmentFormVisible" :close-on-click-modal="false">
			<el-form :model="departmentForm" label-width="80px" :rules="departmentFormRules" ref="departmentForm">
				<el-form-item label="名称" prop="name">
					<el-input v-model="departmentForm.name" auto-complete="off"></el-input>
				</el-form-item>
        <el-form-item label="标识" prop="sn">
          <el-input v-model="departmentForm.sn" auto-complete="off"></el-input>
        </el-form-item>
        <el-form-item label="路径" prop="dirPath">
          <el-input v-model="departmentForm.dirPath" auto-complete="off"></el-input>
        </el-form-item>
				<el-form-item label="状态">
					<el-radio-group v-model="departmentForm.state">
						<el-radio class="radio" :label="0">启用</el-radio>
						<el-radio class="radio" :label="-1">禁用</el-radio>
					</el-radio-group>
				</el-form-item>
        <el-form-item label="部门经理" prop="manager">
          <el-select v-model="departmentForm.manager" value-key="id" placeholder="请选择">
            <el-option
                v-for="item in employees"
                :key="item.id"
                :label="item.username"
                :value="item">
              <span style="float: left">{{ item.username }}</span>
              <span style="float: right; color: #8492a6; font-size: 13px">{{ item.email }}</span>
            </el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="上级部门" prop="parent">
        <!--<el-input v-model="departmentForm.parent" auto-complete="off"></el-input>-->
          <el-cascader
              v-model="parentId"
              :options="DepartmentTree"
              :props="{ checkStrictly: true,label:'name',value:'id' }"
              clearable></el-cascader>
        </el-form-item>
			</el-form>
			<div slot="footer" class="dialog-footer">
				<el-button @click.native="departmentFormVisible = false">取消</el-button>
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
				departments: [],
				total: 0,
				page: 1,
				listLoading: false,
				sels: [],//列表选中列
        parentId:null,

        title: '',
        employees: [],//员工信息
        DepartmentTree: [],//部门树

				departmentFormVisible: false,//编辑界面是否显示
				editLoading: false,
				departmentFormRules: {
					name: [
						{ required: true, message: '请输入名称', trigger: 'blur' }
					]
				},
				//编辑界面数据
				departmentForm: {
					id: null,
					name: '',
          sn: '',
					state: 0,
					dirPath: '',
          manager: null,
          parent: null
				},
			}
		},
		methods: {
      getEmployee(){
        this.$http.get("/employee")
            .then(result=>{
              this.employees = result.data
            })
            .catch(result=>{

            })
      },
      getDepartmentTree(){
        this.$http.get("/department/departmentTree")
            .then(result=>{
              this.DepartmentTree = result.data
            })
            .catch(result=>{

            })
      },
			handleCurrentChange(val) {
				this.page = val;
				this.getDepartments();
			},
			//获取用户列表
			getDepartments() {
				let para = {
					currentPage: this.page,
					keyword: this.query.keyword
				};
				this.listLoading = true;
				//NProgress.start();
        this.$http.post("/department",para)
            .then(result=>{
              this.total = result.data.total;
              this.departments = result.data.data;
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
          var url = "/department/"+row.id
          this.$http.delete(url)
          .then(result=>{
            this.listLoading = false;
            if(result.data.success){
              this.$message({
                message: '删除成功',
                type: 'success'
              });
              this.getDepartments();
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
        this.getEmployee();
        this.getDepartmentTree();
        this.parentId = row.parent?row.parent.id:'';
				this.departmentFormVisible = true;
				this.departmentForm = Object.assign({}, row);



			},
			//显示新增界面
			handleAdd: function () {
        this.getEmployee();
        this.getDepartmentTree();
				this.departmentFormVisible = true;
				this.departmentForm = {
          id: null,
          name: '',
          sn: '',
          state: 0,
          dirPath: '',
          manager: null,
          parent: null
				};
			},
			//编辑
			editSubmit: function () {
				this.$refs.departmentForm.validate((valid) => {
					if (valid) {
						this.$confirm('确认提交吗？', '提示', {}).then(() => {
							this.editLoading = true;
              this.departmentForm.parent={
                //gei
                id:this.parentId[this.parentId.length-1]
              }
              this.$http.put("/department",this.departmentForm)
                .then(result=>{
                  if(result.data.success){
                    this.editLoading = false;
                    this.$message({
                      message: '提交成功',
                      type: 'success'
                    });
                    this.$refs['departmentForm'].resetFields();
                    this.getDepartments();
                  }else{
                    this.editLoading = false;
                    this.$message({
                      message: result.data.message,
                      type: 'error'
                    });
                  }
                  this.departmentFormVisible = false;
                })
                .catch(result=>{
                  this.editLoading = false;
                  this.$message({
                    message: result.data.message,
                    type: 'error'
                  });
                  this.departmentFormVisible = false;
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
          this.$http.patch("/department/bulk",ids)
            .then(result=>{
              if(result.data.success){
                this.listLoading = false;
                this.$message({
                  message: '删除成功',
                  type: 'success'
                });
                this.getDepartments();
              }else {
                this.listLoading = false;
                this.$message({
                  message: result.data.message,
                  type: 'error'
                });
                this.getDepartments();
              }
            })
            .catch(result=>{
              this.listLoading = false;
              this.$message({
                message: result.data.message,
                type: 'error'
              });
              this.getDepartments();
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
			this.getDepartments();
		}
	}
</script>

<style scoped>
</style>