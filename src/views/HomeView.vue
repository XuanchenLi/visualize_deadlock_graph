<template>
  <div class="home">
    <el-container class="main-header">
      <el-header style="text-align: right; font-size: 12px; height: 40px">
        <div class="header-title" style="color:azure">
         <h1 style="text-align: center; font-size:30px; margin: 0px; height: 40px">资源图约简可视化</h1>
        </div>
      </el-header>
    </el-container>
    <el-container>
      <div class="graph" id="graph" style="margin-top:40px;width: 1600px;height:895px;" @contextmenu="openMenu" @dblclick="openTab">
      </div>
      <div style="margin-top:40px; height: 895px; width: 330px">
        <el-row style="margin-top: 5px">
          <el-button type="primary" style="width: 60px; margin-left: 13px; padding-left: 18px" @click="simplify($event)">
            GO
          </el-button>
          <el-button type="success" @click="this.clear($event)">
            Clear
          </el-button>
          <el-button type="warning" style="width: 60px; padding-left: 15px" @click="this.simplifyStep($event)">
            Step
          </el-button>
          <el-button type="info" style="width: 70px; padding-left: 18px" :disabled="this.curIndex === -1" @click="this.rollback($event)">
            Back
          </el-button>
        </el-row>
        <el-row>
          <div class="slider-block">
            <span class="demonstration" style="font-size: 15px">自动执行间隔时间(ms) </span>
            <el-slider v-model="intervalTime" :min="100" :max="3000" :step="100"/>
          </div>
        </el-row>
        <!--
        <el-row>
          <div class="slider-block">
            <span class="demonstration" style="font-size: 15px">结点尺寸 </span>
            <el-slider v-model="symbolSize" :min="40" :max="100" :step="5" @change="changeSize"/>
          </div>
        </el-row>
        -->
        <div class="chat" style="overflow-y: scroll">

          <el-collapse style="" v-show="this.curIndex >= 0">
            <el-collapse-item v-for="i in this.curIndex + 1" :name="i" class="step-item">
              <template #title>
                Step{{i}}
              </template>
              <div>
                <el-timeline>
                  <el-timeline-item
                    color="#0bbd87"
                  >
                    找到非孤立无请求边进程结点：<br/>{{this.rollbackQ[i - 1][2][0]}}
                  </el-timeline-item>
                  <el-timeline-item
                      color="#24406F"
                  >
                    <span v-if="this.rollbackQ[i - 1][2].length > 1">当前非孤立无请求边进程结点：</span>
                    <span v-if="this.rollbackQ[i - 1][2].length > 1" v-for="j in (this.rollbackQ[i - 1][2].length)"> {{this.rollbackQ[i - 1][2][j]}}<br/></span>
                    <span v-if="this.rollbackQ[i - 1][2].length === 1">无非孤立无请求边结点</span>
                  </el-timeline-item>

                </el-timeline>
              </div>
            </el-collapse-item>
          </el-collapse>

        </div>
      </div>
    </el-container>
    <ul v-show="visible" :style="{left:left+'px',top:top+'px'}" class="contextmenu" @contextmenu="prevent">
      <!-- <li v-if="rightClickItem.fileType==99" @click="handleClickFolder(rightClickItem)">打开</li>
      <li>{{gleft}}, {{gtop}}</li>
      -->

      <li v-show="addPNodeVis" @click="addPNode">添加进程节点</li>
      <li v-show="addRNodeVis" @click="addRNode">添加资源节点</li>
      <li v-show="delNodeVis" @click="delNode">删除节点</li>
      <li v-show="delEdgeVis" @click="delEdge">删除连接</li>

    </ul>
    <ul v-show="nodeDataVis" :style="{left:left+'px',top:top+'px'}" style="width: 300px" class="dataTab" @click="cancelBubble" >
      <el-form
          ref="ruleFormRef"
          :model="ruleForm"
          status-icon
          :rules="rules"
          label-width="80px"
          class="demo-ruleForm"
      >
        <el-form-item label="名称" prop="name">
          <el-input style="width: 200px" v-model="ruleForm.name" type="name" autocomplete="off" />
        </el-form-item>
        <el-form-item label="个数" prop="value">
          <el-input-number
              :min="1"
              style="width: 200px"
              v-model="ruleForm.value"
              type="value"
              autocomplete="off"
              :disabled="curNodeData == null || curNodeData.data['category'] === 'Process'"
          />
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="submitForm(ruleFormRef, $event)"
          >Submit</el-button
          >
        </el-form-item>
      </el-form>

    </ul>
  </div>
</template>

<script lang="ts">
import { Options, Vue } from 'vue-class-component';
import * as echarts from 'echarts'
import {GraphEdgeItemOption, GraphNodeItemOption} from "echarts/types/src/chart/graph/GraphSeries";
import {EChartsType} from "echarts/core";
import {reactive, ref, watch} from "vue";
import {ElFormItem, ElMessage} from "element-plus";
import {Refresh} from '@element-plus/icons'


type EchartsOption = echarts.EChartsOption
interface Edge {
  from: String
  to: String
}
interface Node {
  name: String
  value?: String
  x: number
  y: number
  symbolSize?: number
  draggable?: boolean
  category?: String

}


@Options({
  components: {
    Refresh
  },
})
export default class HomeView extends Vue {
  //public main: HTMLElement = document.getElementById("graph") as HTMLElement;
  visible: boolean = false
  delNodeVis: boolean = false
  addPNodeVis: boolean = false
  addRNodeVis: boolean = false
  delEdgeVis: boolean = false
  history: any = []
  left:number = 0
  top:number = 0
  gleft: number = 0
  gtop: number = 0
  totalP: number = 0
  totalR: number = 0
  curObj: any = null
  fromNode: any = null
  toNode: any = null
  nodeDataVis: boolean = false
  curNodeData: any = null
  ruleForm = reactive({
    name: '',
    value: 1,
  })
  intervalTime:number = 1000
  symbolSize:number = 50
  curIndex: number = -1
  rollbackQ: any[] = []
  ruleFormRef = ref<typeof ElFormItem>()
  stake: GraphNodeItemOption[] = [
    {
      name: '__stake1',
      x: 0,
      y: 0,
      symbolSize: 0,
      label: {
        show: false
      }

    },
    {
      name: '__stake2',
      x: 800,
      y: 0,
      symbolSize: 0,
      label: {
        show: false
      }
    },
    {
      name: '__stake3',
      x: 0,
      y: 600,
      symbolSize: 0,
      label: {
        show: false
      }
    },
    {
      name: '__stake4',
      x: 800,
      y: 600,
      symbolSize: 0,
      label: {
        show: false
      }
    }
  ]


  links: GraphEdgeItemOption[] = [

  ]

  nodes: GraphNodeItemOption[] = [

  ]
  option: EchartsOption = {
    title: {
      text: ''
    },
    // nodeScaleRatio: 0,

    animationDurationUpdate: 1300,
    animationEasingUpdate: 'quinticInOut',
    series : [
      {
        type: 'graph',
        layout: 'none',
        symbolSize: 50,
        roam: false,

        scaleLimit: {
          min:0.4,
          max:1,
        },
        label: {
          show: true,
          fontSize: 15,
        },
        edgeSymbol: ['circle', 'arrow'],
        edgeSymbolSize: [3, 15],
        edgeLabel: {
          fontSize: 25
        },
        data: this.nodes,
        // links: [],
        links: this.links,
        lineStyle: {
          opacity: 0.9,
          width: 3,
          curveness: 0
        }
      }
    ]
  }
  graph: any = null
  id: number = 0;
  mounted() {

    let main: HTMLElement = document.getElementById("graph") as HTMLElement;
    this.graph = echarts.init(main)
    this.graph.on('contextmenu', (param: any) => {
          let self:any = this
          this.visible = false
          this.addPNodeVis = false
          this.delNodeVis = false
          this.delEdgeVis = false
          if (param.dataType == "node") {
            self.delNodeVis = true
            self.curObj = param
          }else if (param.dataType == "edge") {
            self.delEdgeVis = true
            self.curObj = param
          }

        }
    )
    this.graph.on('mousedown', (param: any) => {
      let self:any = this
      // console.log("down")
      if (param.dataType == "node") {
        self.delNodeVis = true
        self.fromObj = param
        self.ruleForm.name = param.data['name']
        self.ruleForm.value = param.data['value']
      }
    })
    this.graph.on('mouseup', (param: any) => {
      let self:any = this
      // console.log("up")
      // console.log(self.fromObj.data["category"], self.toObj)
      if (param.dataType == "node") {
        clearInterval(this.id)
        self.delNodeVis = true
        self.toObj = param

        if (self.fromObj != null && self.fromObj.dataType == "node" && self.fromObj.name != self.toObj.name) {
          if (self.toObj.data["category"] == self.fromObj.data["category"]) {
            this.inform("同种对象不可连接", 'warning')
            return
          }

          if (self.fromObj.data["category"] == "Resource") {
            let cnt = this.getAskRest("", self.fromObj.name)
            if (cnt[1] <= 0)
            {
              this.inform("超出资源个数", 'warning')
              return
            }
          }
          //let cnt:number = 0
          self.links.push(
              {
                source: self.fromObj.name,
                target: self.toObj.name,
                lineStyle: {
                  color: self.fromObj.data["category"] === "Resource" ? "rgb(0, 0, 128)" : "rgb(128, 0, 0)",
                  curveness: 0.0
                }
              }
          )
          self.clean(self.fromObj.name, self.toObj.name)
          this.curIndex = -1
          this.rollbackQ.splice(0, this.rollbackQ.length)
          self.graph.setOption(self.option)
        }else {

        }
      }
      //console.log(self.links)
      self.fromObj = null
      self.toObj = null
    })
    this.graph.on('dblclick', (param: any) => {
      let self:any = this
      // console.log(param)
      if (param.dataType == "node") {
        self.nodeDataVis = true
        self.curNodeData = param
      }
    })
    for (let i in this.stake) {
      this.nodes.push(this.stake[i])
    }
    //console.log(this.nodes)

    this.graph.setOption(this.option)



  }
  inform(msg: string, type:any) {
    ElMessage({
      message: msg,
      type: type,
    })
  }
  clean (from: string, to: string) {
    //console.log(11)
    let cnt = 0
    let self = this
    for (var i in this.links) {
      if ((self.links[i].source == from && self.links[i].target == to) ||
          (self.links[i].source == to && self.links[i].target == from)) {
        cnt += 1
      }
    }
    //console.log(cnt)
    if (cnt >= 2) {
      let tmp: number = 0
      for (var i in self.links) {
        if ((self.links[i].source == from && self.links[i].target == to) ||
            (self.links[i].source == to && self.links[i].target == from)) {
          //tmp += 1
          let flag: number = self.links[i].source == from && self.links[i].target == to ? 1 : -1
          this!.links[i]!.lineStyle!["curveness"] = -0.4 + 0.8 * (tmp / (cnt - 1))
          this!.links[i]!.lineStyle!["curveness"]! *= flag
          tmp += 1
          // console.log(self.links[i].lineStyle["curveness"])
        }
      }
    }else if (cnt == 1) {
      for (var i in self.links) {
        if ((self.links[i].source == from && self.links[i].target == to) ||
            (self.links[i].source == to && self.links[i].target == from)) {
          this!.links[i]!.lineStyle!["curveness"] = 0
          break
          // console.log(self.links[i].lineStyle["curveness"])
        }
      }
    }
  }
  prevent(e:any) {
    e.preventDefault()
  }
  cancelBubble(e:any) {
    e.cancelBubble = true
  }
  openMenu(e:any) {

    e.preventDefault()
    let x = e.clientX;
    let y = e.clientY;
    this.left = x
    this.top = y
    let result = this.graph.convertFromPixel({
      seriesIndex:0,
      xAxisIndex:0
    },[x,y]);
    this.gleft = parseInt(result[0])
    this.gtop = parseInt(result[1]) - 30
    //console.log(this.gleft, this.gtop)

    var menu = document.querySelector('.contextmenu')
    this.styleMenu(e, menu)
    this.visible = true
    this.addPNodeVis = true
    this.addRNodeVis = true
    //console.log(this.gleft, this.gtop)
    //console.log(this.option.series)

  }
  closeMenu() {
    this.visible = false;
  }
  foo() {

// 取消鼠标监听事件 菜单栏
    this.visible = false
    this.addPNodeVis = false
    this.delNodeVis = false
    this.delEdgeVis = false
    this.addRNodeVis = false
    this.nodeDataVis = false

    document.removeEventListener('click', this.foo) // 关掉监听，
    //document.removeEventListener('contextmenu', this.foo)
  }

  styleMenu(e:any, menu:any) {

    if (e.clientX > 1800) {

      menu.style.left = e.clientX - 100 + 'px'

    } else {

      menu.style.left = e.clientX + 1 + 'px'

    }

    document.addEventListener('click', this.foo) // 给整个document新增监听鼠标事件，点击任何位置执行foo方法
    //document.addEventListener('contextmenu', this.foo)
    if (e.clientY > 700) {

      menu.style.top = e.clientY - 30 + 'px'

    } else {

      menu.style.top = e.clientY - 10 + 'px'

    }
  }

  openTab(e:any) {

    e.preventDefault()
    let x = e.clientX;
    let y = e.clientY;
    this.left = x
    this.top = y


    var tab = document.querySelector('.dataTab')
    this.styleTab(e, tab)

  }
  styleTab(e:any, menu:any) {

    if (e.clientX > 1800) {

      menu.style.left = e.clientX - 100 + 'px'

    } else {

      menu.style.left = e.clientX + 1 + 'px'

    }

    document.addEventListener('click', this.foo) // 给整个document新增监听鼠标事件，点击任何位置执行foo方法
    //document.addEventListener('contextmenu', this.foo)
    if (e.clientY > 700) {

      menu.style.top = e.clientY - 30 + 'px'

    } else {

      menu.style.top = e.clientY - 10 + 'px'

    }
  }

  addPNode(e:any) {
    clearInterval(this.id)
    this.nodes.push(
        {
          name: 'Process ' + this.totalP++,
          x: this.gleft,
          y: this.gtop,
          symbol: "circle",
          category: "Process",
          value: 1,
          itemStyle: {
            color: this.randomHexColor()
          },
          label: {
            formatter: "{b}"
          }
        }
    )
    this.curIndex = -1
    this.rollbackQ.splice(0, this.rollbackQ.length)
    this.graph.setOption(this.option)
  }
  addRNode(e:any) {
    clearInterval(this.id)
    this.nodes.push(
        {
          name: 'Resource ' + this.totalR++,
          x: this.gleft,
          y: this.gtop,

          symbol: "rect",
          category: "Resource",
          value: 1,
          itemStyle: {
            color: this.randomHexColor()
          },
          label: {
            formatter: "{b}\n{c}"
          }
        }
    )
    this.curIndex = -1
    this.rollbackQ.splice(0, this.rollbackQ.length)
    this.graph.setOption(this.option)
  }
  randomHexColor() { //随机生成十六进制颜色
    var hex = Math.floor(Math.random() * 16777216).toString(16); //生成ffffff以内16进制数
    while (hex.length < 6) { //while循环判断hex位数，少于6位前面加0凑够6位
      hex = '0' + hex;
    }
    return '#' + hex; //返回‘#'开头16进制颜色
  }
  delNode() {
    for (var i in this.nodes) {
      if (this.nodes[i].name === this.curObj.name){
        this.nodes.splice(parseInt(i), 1)
        break
      }
    }
    for (let i = this.links.length - 1; i >=0; --i) {
      if (this.links[i].source === this.curObj.name || this.links[i].target === this.curObj.name){
        this.links.splice(i, 1)
      }
    }
    this.curObj = null
    this.curIndex = -1
    this.rollbackQ.splice(0, this.rollbackQ.length)
    this.graph.setOption(this.option)
  }
  delEdge() {
    for (var i in this.links) {
      if (this.links[i].source === this.curObj.data["source"] && this.links[i].target === this.curObj.data["target"]){
        this.links.splice(parseInt(i), 1)
        break
      }
    }
    this.clean(this.curObj.data["source"], this.curObj.data["target"])
    this.curObj = null
    this.curIndex = -1
    this.rollbackQ.splice(0, this.rollbackQ.length)
    this.graph.setOption(this.option)
  }
  submitForm = (formEl: any, e:any) => {
    // console.log(formEl)
    //const tab: any = this.$refs[formEl]
    this.myBlur(e)
    formEl.validate((valid: any) => {
      if (valid) {
        for (let i in this.nodes) {
          if (this.nodes[i].name == this.curNodeData.name) {
            this.nodes[i].name = this.ruleForm.name
            this.nodes[i].value = this.ruleForm.value
            break
          }
        }
        for (let i in this.links) {
          if (this.links[i].source == this.curNodeData.name) {
            this.links[i].source = this.ruleForm.name
          }else if (this.links[i].target == this.curNodeData.name) {
            this.links[i].target = this.ruleForm.name
          }
        }
        this.nodeDataVis = false
        // console.log(this.nodes)
        this.curIndex = 0
        this.rollbackQ.splice(0, this.rollbackQ.length)
        this.graph.setOption(this.option)
      } else {
        console.log('error submit!')
        return false
      }
    })

  }
  validateName = (rule: any, value: any, callback: any) => {
    if (value === '') {
      callback(new Error('不可为空'))
    } else {
      for (var i in this.nodes) {
        if (this.nodes[i].name == value) {
          callback(new Error('名称重复'))
        }
        break
      }
      }
      callback()
    }
  validateValue = (rule: any, value: any, callback: any) => {
    if (value <= 0) {
      callback(new Error('必须为正'))
    }
    callback()
  }
  rules = reactive({
    name: [{ validator: this.validateName, trigger: 'blur' }],
    value: [{ validator: this.validateValue, trigger: 'blur' }],
  })
  myBlur(event: any) {
    let tar = event.target
    while (tar.nodeName !== "BUTTON") {
      tar = tar.parentNode
    }
    tar.blur()
  }
  clear(e: any) {
    this.myBlur(e)
    this.nodes.splice(0, this.nodes.length)
    this.links.splice(0, this.links.length)
    for (let i in this.stake) {
      this.nodes.push(this.stake[i])
    }
    this.totalP = 0
    this.totalR = 0
    this.curIndex = -1
    this.rollbackQ.splice(0, this.rollbackQ.length)
    this.graph.setOption(this.option)
  }
  findNextP() {
    //console.log(this.links)
    for (let i in this.nodes) {
      let cnt = 0
      let valid = true
      if (this.nodes[i].category != "Process") continue
      for (let j in this.links) {
        //console.log(j)
         if (this.links[j].source === this.nodes[i].name) {
           // 有请求边
           valid = false
           break;
         }else if (this.links[j].target === this.nodes[i].name) {
           cnt += 1
         }
      }
      //console.log(this.links)
      if (!valid || cnt == 0) continue
      return this.nodes[i].name
    }
    return null
  }
  getAskRest(name1: string, name2: string) {
    let cntA = 0
    let cntR = 0
    for (var i in this.links) {
      if (this.links[i].source === name2 ) {
        cntR += 1
      }
      if (this.links[i].source === name1 && this.links[i].target === name2) {
        cntA += 1
      }
    }
    for (var i in this.nodes) {
      if (this.nodes[i].name == name2) {
        cntR = (this.nodes[i].value as number) - cntR
        break
      }
    }
    return [cntA, cntR]
  }
  updateSatisfy() {
    let minus = []
    let plus = []
    let newSet = []
    for (let i in this.nodes) {
      let valid = true
      let isolate = true
      if (this.nodes[i].category != "Process") continue
      for (let j in this.links) {
        if (this.links[j].source == this.nodes[i].name || this.links[j].target == this.nodes[i].name)
          isolate = false
        if (this.links[j].source == this.nodes[i].name) {
          let cnt = this.getAskRest(this.nodes[i].name as string, this.links[j].target as string)
          if (cnt[1] < cnt[0]) {
            valid = false
            break
          }
        }
      }
      if (!valid || isolate) continue
      for (let j in this.links) {
        if (this.links[j].source == this.nodes[i].name) {
          minus.push(JSON.parse(JSON.stringify(this.links[j])))
          this.links[j].source = this.links[j].target
          this.links[j].target = this.nodes[i].name
          this.links[j]!.lineStyle!.color = "rgb(0, 0, 128)"
          plus.push(JSON.parse(JSON.stringify(this.links[j])))
          this.clean(this.links[j].source as string, this.links[j].target as string)
        }
      }
      newSet.push(this.nodes[i].name)
    }
    //this.graph.clear()
    //this.sleepAsync(0, () => {this.graph.setOption(this.option)})
    for (let i in minus) {
      this.rollbackQ[this.curIndex][0].push(minus[i])
    }
    for (let i in plus) {
      this.rollbackQ[this.curIndex][1].push(plus[i])
    }
    //console.log(this.rollbackQ)
    this.graph.setOption(this.option)
    return newSet
  }
  simplifyStep(e: any) {

    let minus = []
    let plus: never[] = []
    //console.log(this.links)
    this.myBlur(e)
    //console.log(this.links)
    let next = this.findNextP()
    //console.log(this.links)

    if (next == null) {
      clearInterval(this.id)
      if (this.links.length != 0) {
        this.inform("不可完全约简！", "error")
        return null
      }
      this.inform("约简完毕", "success")
      return null
    }
    else {
      // 变为孤立
      for (let i = this.links.length-1; i >= 0; i--) {
        //console.log(i)
        if (this.links[i].target === next) {
          minus.push(this.links[i])
          this.links.splice(i, 1)
        }
      }
      this.curIndex += 1
      if (this.rollbackQ.length - 1 > this.curIndex) {
        this.rollbackQ.splice(this.curIndex)
      }
      //console.log(this.rollbackQ.length)
      this.rollbackQ.push([])
      //console.log(this.rollbackQ.length)
      //console.log(this.rollbackQ[this.curIndex][1])
      //this.graph.setOption(this.option)
      // 更新其余节点
      this.rollbackQ[this.curIndex] = [minus, plus, []]
      // console.log(this.curIndex)
      let updateSet = this.updateSatisfy()
      this.rollbackQ[this.curIndex][2].push(next)
      this.rollbackQ[this.curIndex][2] = this.rollbackQ[this.curIndex][2].concat(updateSet)
      //console.log(this.rollbackQ[this.curIndex][2])
      return updateSet
    }

  }
  simplify(e: any) {
    clearInterval(this.id)
    this.myBlur(e)
    let res:(string | undefined)[] | null = ["init"]
    this.id = setInterval(() => this.simplifyStep(e), this.intervalTime)
    this.simplifyStep(e)
  }
  rollback(e: any) {
    console.log(this.curIndex)
    clearInterval(this.id)
    this.myBlur(e)
    if (this.curIndex >= 0) {
     for (let i in this.rollbackQ[this.curIndex][1]) {
       for (let j in this.links) {
         if (this.links[j].source == this.rollbackQ[this.curIndex][1][i].source
             && this.links[j].target == this.rollbackQ[this.curIndex][1][i].target) {
           this.links.splice(parseInt(j), 1)
           this.clean(this.rollbackQ[this.curIndex][1][i].source, this.rollbackQ[this.curIndex][1][i].target)
           break
         }
       }
     }
      for (let i in this.rollbackQ[this.curIndex][0]) {
        this.links.push(this.rollbackQ[this.curIndex][0][i])
        this.clean(this.rollbackQ[this.curIndex][0][i].source, this.rollbackQ[this.curIndex][0][i].target)
      }
      this.curIndex -= 1
      //console.log(this.links)
      this.graph.setOption(this.option)
    }
  }

  sleep(delay:number): void {
    var start = (new Date()).getTime();
    while ((new Date()).getTime() - start < delay) {
      continue;
    }
  }

  changeSize() {
    console.log(this.option)
    //this.option.series = this.symbolSize
    this.graph.clear()
    this.graph.setOption(this.option)
  }

}


</script>


<style>
.main-header {
  background-color: #24406f;
  height: 40px;
  position: fixed;
  z-index: 2;
  width: 100%;
}
.graph {
  background-color: #ffffff;

  background-image: linear-gradient(90deg, rgba(50, 0, 0, 0.15) 3%, rgba(0, 0, 0, 0) 3%),
  linear-gradient(360deg, rgba(50, 0, 0, 0.15) 3%, rgba(0, 0, 0, 0) 3%);
  background-size: 20px 20px;
  background-repeat: repeat;
  background-position: center center;


}

.contextmenu {
  margin: 0;
  background: #223449;
  z-index: 3000;
  position: fixed;
  list-style-type: none;
  padding: 5px 0;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 400;
  color: #333;
  box-shadow: 2px 2px 3px 0 rgba(0, 0, 0, 0.3);
}

.contextmenu li {
  color: white;
  font-size: 15px;
  margin: 0;
  padding: 7px 16px;
  cursor: pointer;
}

.contextmenu li:hover {
  background-color: rgb(3, 125, 243);;
  color: white;
}

.dataTab {
  margin: 0;
  background: #dadadb;
  z-index: 3000;
  position: fixed;
  list-style-type: none;
  padding: 5px 0;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 400;
  color: #333;
  box-shadow: 2px 2px 3px 0 rgba(0, 0, 0, 0.3);
}

.dataTab li {
  color: white;
  font-size: 15px;
  margin: 0;
  padding: 7px 16px;
  cursor: pointer;
}

.dataTab li:hover {
  background-color: rgb(3, 125, 243);;
  color: white;
}

.chat {
  margin-top: 5px;
  border-radius: 15px;
  border: solid azure;
  height: 800px;
  box-shadow:0px 0px 8px -2px #000;
}

.step-item .el-collapse-item__header {
  color: rgb(30, 34, 45);
  font-size: medium;
  font-family: "Arial","Microsoft YaHei","黑体","宋体",serif;
  background-color: #f0f0f3;
}
.step-item .el-collapse-item__wrap {
  font-family: "Arial","Microsoft YaHei","黑体","宋体",serif;

}
.slider-block {
  display: flex;
  align-items: center;
  width: 340px;
}
.slider-block .el-slider {
  margin-top: 0;
  margin-left: 12px;
  width: 200px;
  margin-left: 0px;
}
.demonstration {
  width: 120px;
}
.slider-demo-block .demonstration + .el-slider {
  flex: 0 0 70%;
}
</style>
