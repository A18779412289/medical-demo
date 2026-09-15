<template>
  <div class="container">
    <!-- 加载提示 -->
    <div v-if="loading" class="loading-mask">
      <el-spinner size="40"></el-spinner>
      <p>数据加载中...</p>
    </div>

    <h2 class="page-title">糖尿病患者血糖风险可视化分析平台</h2>

    <!-- 统计卡片 -->
    <el-row :gutter="20" class="stat-row">
      <el-col :span="6">
        <el-card class="stat-card" shadow="hover">
          <div class="stat-num">{{ tableData.length }}</div>
          <div class="stat-label">患者总人数</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="stat-card" shadow="hover">
          <div class="stat-num" style="color:#F56C6C">{{ highCount }}</div>
          <div class="stat-label">高风险人数</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="stat-card" shadow="hover">
          <div class="stat-num" style="color:#E6A23C">{{ midCount }}</div>
          <div class="stat-label">中风险人数</div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card class="stat-card" shadow="hover">
          <div class="stat-num" style="color:#67C23A">{{ lowCount }}</div>
          <div class="stat-label">低风险人数</div>
        </el-card>
      </el-col>
    </el-row>

    <!-- 筛选区 + 搜索 + 按钮 -->
    <el-row class="filter-row" :gutter="10">
      <el-col :span="6">
        <span class="filter-label">风险筛选：</span>
        <el-select v-model="filterRisk" placeholder="全部风险" clearable style="width:150px">
          <el-option label="高风险" value="高风险"></el-option>
          <el-option label="中风险" value="中风险"></el-option>
          <el-option label="低风险" value="低风险"></el-option>
        </el-select>
      </el-col>
      <el-col :span="7">
        <span class="filter-label">姓名搜索：</span>
        <el-input v-model="searchName" placeholder="输入姓名筛选" clearable style="width:170px"></el-input>
      </el-col>
      <el-col :span="6" style="text-align:center;line-height:40px;color:#606266">
        当前显示 <b>{{ showTableData.length }}</b> 位
      </el-col>
      <el-col :span="5" style="text-align:right">
        <el-button size="mini" @click="resetData">重置数据</el-button>
        <el-button type="success" size="mini" @click="openAddDialog">新增患者</el-button>
      </el-col>
    </el-row>

    <!-- 表格 + 风险饼图 -->
    <el-row :gutter="20" class="content-row">
      <el-col :span="14">
        <el-card shadow="hover">
          <div slot="header" class="card-header">
            <span>患者信息表</span>
            <el-button type="primary" size="mini" @click="exportExcel">导出CSV</el-button>
          </div>
          <el-table :data="showTableData" border stripe height="360">
            <el-table-column prop="id" label="编号" width="70" sortable></el-table-column>
            <el-table-column prop="name" label="姓名"></el-table-column>
            <el-table-column prop="age" label="年龄" sortable></el-table-column>
            <el-table-column prop="glucose" label="血糖值(mmol/L)" sortable>
              <template slot-scope="scope">
                <span :style="scope.row.glucose >=7 ? 'background:#fde2e2;color:#f56c6c;padding:2px 4px;border-radius:2px' : ''">
                  {{ scope.row.glucose }}
                </span>
              </template>
            </el-table-column>
            <el-table-column prop="sex" label="性别"></el-table-column>
            <el-table-column prop="result" label="患病风险">
              <template slot-scope="scope">
                <el-tag :type="getTagType(scope.row.result)">{{ scope.row.result }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column label="操作" width="160">
              <template slot-scope="scope">
                <el-button size="mini" type="primary" @click="openEditDialog(scope.row)">编辑</el-button>
                <el-button size="mini" type="danger" @click="delPatient(scope.row.id)">删除</el-button>
              </template>
            </el-table-column>
          </el-table>
        </el-card>
      </el-col>
      <el-col :span="10">
        <el-card header="患病风险分布饼图" shadow="hover">
          <div id="pieChart" style="width:100%;height:360px;"></div>
        </el-card>
      </el-col>
    </el-row>

    <!-- 性别饼图 + 血糖柱状图 -->
    <el-row :gutter="20" class="content-row">
      <el-col :span="10">
        <el-card header="性别分布饼图" shadow="hover">
          <div id="sexChart" style="width:100%;height:340px;"></div>
        </el-card>
      </el-col>
      <el-col :span="14">
        <el-card header="患者血糖对比柱状图" shadow="hover">
          <div id="barChart" style="width:100%;height:340px;"></div>
        </el-card>
      </el-col>
    </el-row>

    <!-- 年龄分段折线图 -->
    <el-row :gutter="20" class="content-row">
      <el-col :span="24">
        <el-card header="不同年龄段平均血糖趋势图" shadow="hover">
          <div id="ageLineChart" style="width:100%;height:300px;"></div>
        </el-card>
      </el-col>
    </el-row>

    <!-- 新增/编辑共用弹窗 -->
    <el-dialog :title="isEdit ? '编辑患者信息' : '新增患者信息'" :visible.sync="dialogVisible" width="40%">
      <el-form :model="form" ref="formRef" label-width="80px">
        <el-form-item label="编号" prop="id"><el-input v-model="form.id"></el-input></el-form-item>
        <el-form-item label="姓名" prop="name"><el-input v-model="form.name"></el-input></el-form-item>
        <el-form-item label="年龄" prop="age"><el-input v-model="form.age"></el-input></el-form-item>
        <el-form-item label="血糖值" prop="glucose"><el-input v-model="form.glucose"></el-input></el-form-item>
        <el-form-item label="性别" prop="sex">
          <el-select v-model="form.sex">
            <el-option label="男" value="男"></el-option>
            <el-option label="女" value="女"></el-option>
          </el-select>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" @click="submitForm">{{ isEdit ? '保存修改' : '确定添加' }}</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import * as echarts from 'echarts'
const DEFAULT_DATA = [
  {id:1, name:'张三', age:45, glucose:7.8, sex:'男', result:'高风险'},
  {id:2, name:'李四', age:32, glucose:5.3, sex:'女', result:'低风险'},
  {id:3, name:'王五', age:56, glucose:8.6, sex:'男', result:'高风险'},
  {id:4, name:'赵六', age:41, glucose:6.1, sex:'女', result:'中风险'},
  {id:5, name:'陈七', age:28, glucose:4.9, sex:'女', result:'低风险'},
  {id:6, name:'周八', age:63, glucose:9.2, sex:'男', result:'高风险'},
  {id:7, name:'吴九', age:50, glucose:6.8, sex:'男', result:'中风险'},
  {id:8, name:'郑十', age:37, glucose:5.6, sex:'女', result:'低风险'},
  {id:9, name:'孙十一', age:48, glucose:7.1, sex:'女', result:'中风险'},
  {id:10,name:'刘十二', age:68, glucose:8.1, sex:'男', result:'高风险'}
]
export default {
  name: 'App',
  data() {
    return {
      loading: true,
      filterRisk: '',
      searchName: '',
      dialogVisible: false,
      isEdit: false,
      editId: null,
      form: { id:'', name:'', age:'', glucose:'', sex:'' },
      tableData: []
    }
  },
  computed: {
    showTableData() {
      let list = this.tableData
      if (this.filterRisk) list = list.filter(i => i.result === this.filterRisk)
      if (this.searchName) list = list.filter(i => i.name.includes(this.searchName))
      return list
    },
    highCount() { return this.tableData.filter(i => i.result==='高风险').length },
    midCount()  { return this.tableData.filter(i => i.result==='中风险').length },
    lowCount()  { return this.tableData.filter(i => i.result==='低风险').length }
  },
  created() {
    // 从本地存储读取数据
    const saved = localStorage.getItem('patients')
    this.tableData = saved ? JSON.parse(saved) : JSON.parse(JSON.stringify(DEFAULT_DATA))
  },
  mounted() {
    setTimeout(()=>{ this.loading = false; this.renderAll() },600)
  },
  watch: {
    filterRisk(){ this.$nextTick(()=>this.renderAll()) },
    searchName(){ this.$nextTick(()=>this.renderAll()) },
    // 深度监听数据变化，自动保存到本地
    tableData:{
      handler(){
        localStorage.setItem('patients', JSON.stringify(this.tableData))
        this.$nextTick(()=>this.renderAll())
      },
      deep:true
    }
  },
  methods: {
    calcRisk(v){
      if(v >=7.0) return '高风险'
      else if(v >=6.1) return '中风险'
      else return '低风险'
    },
    resetData(){
      this.$confirm('重置将恢复为初始示例数据，确认继续？','提示',{type:'warning'}).then(()=>{
        this.tableData = JSON.parse(JSON.stringify(DEFAULT_DATA))
        this.filterRisk = ''
        this.searchName = ''
        this.$message.success('已恢复示例数据')
      }).catch(()=>{})
    },
    openAddDialog(){
      this.isEdit = false
      this.form = {id:'',name:'',age:'',glucose:'',sex:''}
      this.dialogVisible = true
    },
    openEditDialog(row){
      this.isEdit = true
      this.editId = row.id
      this.form = {...row}
      this.dialogVisible = true
    },
    delPatient(id){
      this.$confirm('确认删除这条患者记录？','提示',{type:'warning'}).then(()=>{
        const idx = this.tableData.findIndex(item=>item.id === id)
        this.tableData.splice(idx,1)
        this.$message.success('删除成功')
      }).catch(()=>{})
    },
    submitForm(){
      const idNum = Number(this.form.id)
      const ageNum = Number(this.form.age)
      const gluNum = Number(this.form.glucose)
      if(!this.form.id || !this.form.name || !this.form.age || !this.form.glucose || !this.form.sex){
        this.$message.error("所有项目不能为空！"); return
      }
      if(isNaN(idNum) || isNaN(ageNum) || isNaN(gluNum)){
        this.$message.error("编号、年龄、血糖值必须填写数字！"); return
      }
      if(ageNum <=0 || gluNum <=0){
        this.$message.error("年龄和血糖不能小于等于0！"); return
      }
      const risk = this.calcRisk(gluNum)
      if(this.isEdit){
        const index = this.tableData.findIndex(i=>i.id === this.editId)
        this.tableData[index] = {id:idNum, name:this.form.name, age:ageNum, glucose:gluNum, sex:this.form.sex, result:risk}
      }else{
        this.tableData.push({id:idNum, name:this.form.name, age:ageNum, glucose:gluNum, sex:this.form.sex, result:risk})
      }
      this.dialogVisible = false
      this.$message.success(this.isEdit ? "修改成功" : "添加成功")
    },
    getTagType(v){
      if(v==='高风险') return 'danger'
      if(v==='中风险') return 'warning'
      return 'success'
    },
    renderAll(){ this.renderPie(); this.renderSex(); this.renderBar(); this.renderAgeLine() },
    renderPie(){
      const c = echarts.init(document.getElementById('pieChart'))
      c.setOption({ tooltip:{trigger:'item'}, legend:{orient:'vertical',left:'left'},
        series:[{type:'pie',radius:'60%',data:[
          {name:'高风险',value:this.highCount},{name:'中风险',value:this.midCount},{name:'低风险',value:this.lowCount}
        ]}] })
      window.addEventListener('resize',()=>c.resize())
    },
    renderSex(){
      const list = this.showTableData
      const male = list.filter(i=>i.sex==='男').length
      const female = list.filter(i=>i.sex==='女').length
      const c = echarts.init(document.getElementById('sexChart'))
      c.setOption({ tooltip:{trigger:'item'}, legend:{orient:'vertical',left:'left'},
        series:[{type:'pie',radius:'60%',data:[{name:'男',value:male},{name:'女',value:female}]}] })
      window.addEventListener('resize',()=>c.resize())
    },
    renderBar(){
      const c = echarts.init(document.getElementById('barChart'))
      c.setOption({ tooltip:{trigger:'axis'},
        xAxis:{type:'category',data:this.showTableData.map(i=>i.name)},
        yAxis:{type:'value',name:'血糖值 mmol/L'},
        series:[{type:'bar',data:this.showTableData.map(i=>i.glucose),
          itemStyle:{color:new echarts.graphic.LinearGradient(0,0,0,1,[{offset:0,color:'#409EFF'},{offset:1,color:'#a0cfff'}])}}] })
      window.addEventListener('resize',()=>c.resize())
    },
    renderAgeLine(){
      const allData = this.showTableData
      const youth = allData.filter(p=>p.age<40)
      const middle = allData.filter(p=>p.age>=40 && p.age<=60)
      const elder = allData.filter(p=>p.age>60)
      const getAvg = arr => arr.length ? (arr.reduce((s,it)=>s+it.glucose,0)/arr.length).toFixed(2) : 0
      const c = echarts.init(document.getElementById('ageLineChart'))
      c.setOption({ tooltip:{trigger:'axis'},
        xAxis:{type:'category',data:['青年(<40)','中年(40~60)','老年(>60)']},
        yAxis:{type:'value',name:'平均血糖 mmol/L'},
        series:[{name:'平均血糖',type:'line',smooth:true,itemStyle:{color:'#27AE60'},
          data:[getAvg(youth),getAvg(middle),getAvg(elder)]}] })
      window.addEventListener('resize',()=>c.resize())
    },
    exportExcel(){
      const list = this.showTableData
      let csv = '\uFEFF编号,姓名,年龄,血糖值,性别,患病风险\n'
      list.forEach(r=>{ csv += `${r.id},${r.name},${r.age},${r.glucose},${r.sex},${r.result}\n` })
      const blob = new Blob([csv],{type:'text/csv;charset=utf-8;'})
      const a = document.createElement('a')
      a.href = URL.createObjectURL(blob)
      a.download = '患者数据.csv'
      a.click()
      URL.revokeObjectURL(a.href)
    }
  }
}
</script>

<style scoped>
.container { padding:24px; background:#f5f7fa; min-height:100vh; box-sizing:border-box; }
.page-title { text-align:center; color:#303133; margin-bottom:24px; font-weight:600; }
.stat-row { margin-bottom:20px; }
.stat-card { text-align:center; }
.stat-num { font-size:30px; font-weight:bold; color:#409EFF; }
.stat-label { color:#666; margin-top:4px; }
.filter-row { margin-bottom:20px; line-height:40px; }
.filter-label { color:#606266; }
.content-row { margin-bottom:20px; }
.card-header { display:flex; justify-content:space-between; align-items:center; }
.dialog-footer { text-align:right; }
.loading-mask { position:fixed;top:0;left:0;right:0;bottom:0;background:rgba(255,255,255,0.7);z-index:9999;display:flex;flex-direction:column;align-items:center;justify-content:center; }
@media (max-width:768px){ .container{padding:12px;} }
</style>
