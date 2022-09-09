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
      <div class="graph" id="graph" style="margin-top:40px;width: 1600px;height:895px;" @contextmenu="openMenu">
      </div>
    </el-container>
    <ul v-show="visible" :style="{left:left+'px',top:top+'px'}" class="contextmenu" @contextmenu="prevent">
      <!-- <li v-if="rightClickItem.fileType==99" @click="handleClickFolder(rightClickItem)">打开</li>
      <li>{{gleft}}, {{gtop}}</li>
      -->

      <li v-show="addPNodeVis" @click="addPNode">添加进程节点</li>
      <li v-show="addRNodeVis" @click="addRNode">添加资源节点</li>
      <li v-show="delNodeVis" @click="delNode">删除节点</li>
      <li v-show="delEdgeVis">删除连接</li>

    </ul>
  </div>
</template>

<script lang="ts">
import { Options, Vue } from 'vue-class-component';
import * as echarts from 'echarts'
import {GraphEdgeItemOption, GraphNodeItemOption} from "echarts/types/src/chart/graph/GraphSeries";
import {EChartsType} from "echarts/core";
import {watch} from "vue";


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

  },
})
export default class HomeView extends Vue {
  //public main: HTMLElement = document.getElementById("graph") as HTMLElement;
  visible: boolean = false
  delNodeVis: boolean = false
  addPNodeVis: boolean = false
  addRNodeVis: boolean = false
  delEdgeVis: boolean = false
  left:number = 0
  top:number = 0
  gleft: number = 0
  gtop: number = 0
  totalP: number = 0
  totalR: number = 0
  curObj: any = null




  links: GraphEdgeItemOption[] = [
    {
      source: 'Node 2',
      target: 'Node 1',

    },
    {
      source: 'Node 1',
      target: 'Node 3'
    },
    {
      source: 'Node 2',
      target: 'Node 3'
    },
    {
      source: 'Node 2',
      target: 'Node 4'
    },
    {
      source: 'Node 1',
      target: 'Node 4'
    }
  ]

  nodes: GraphNodeItemOption[] = [
    {
      name: 'Node 1',
      x: 0,
      y: 0
    },
    {
      name: 'Node 2',
      x: 1100,
      y: 0
    },
    {
      name: 'Node 3',
      x: 0,
      y: 600
    },
    {
      name: 'Node 4',
      x: 1100,
      y: 600,
      symbol: "rect"
    },
    {
      name: 'Node 5',
      x: 650,
      y: 600,
      draggable: false
    }
  ]
  option: EchartsOption = {
    title: {
      text: ''
    },
    tooltip: {},
    // nodeScaleRatio: 0,

    animationDurationUpdate: 1500,
    animationEasingUpdate: 'quinticInOut',
    series : [
      {
        type: 'graph',
        layout: 'none',
        symbolSize: 70,
        roam: false,
        scaleLimit: {
          min:0.4,
          max:1,
        },
        label: {
          show: true,
          fontSize: 15
        },
        edgeSymbol: ['circle', 'arrow'],
        edgeSymbolSize: [3, 20],
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
    this.graph.setOption(this.option)
  }
  prevent(e:any) {
    e.preventDefault()
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


    var menu = document.querySelector('.contextmenu')
    this.styleMenu(e, menu)
    this.visible = true
    this.addPNodeVis = true
    this.addRNodeVis = true


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
  addPNode(e:any) {
    this.nodes.push(
        {
          name: 'Process ' + this.totalP++,
          x: this.gleft,
          y: this.gtop,
          symbol: "circle",
          itemStyle: {
            color: this.randomHexColor()
          }
        }
    )
    this.graph.setOption(this.option)
  }
  addRNode(e:any) {
    this.nodes.push(
        {
          name: 'Resource ' + this.totalR++,
          x: this.gleft,
          y: this.gtop,
          symbol: "rect",
          itemStyle: {
            color: this.randomHexColor()
          }
        }
    )
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
      console.log(this.nodes[i])
    }
    this.curObj = null
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
  box-shadow:5px 3px 8px -2px #000;
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
</style>
