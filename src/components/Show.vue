<template>
    <div class="show">

      <div class="classTree">
        <p>知识分类</p>
        <div class="clzTreeContent">
          <el-tree :props="treeProps" lazy :load="loadClasses" @node-click="classClick"></el-tree>
        </div>
      </div>

      <div class="instanceTree">
        <p>知识实例</p>
        <div class="insTreeContent">
          <el-tree
            :data="instanceData"
            :props="instanceTreeProps"
            @node-click="instanceClick">
          </el-tree>
        </div>
      </div>

      <div class="showInsInfo">
        <p>知识图谱</p>
        <Chart :chartId="'showChart'" :chartName="chartName" :chartData="chartData" :height="'670px'" :weight="'720px'"></Chart>
      </div>

    </div>
</template>

<script>
    import Chart from './chart/Chart'

    export default {
      name: "Show",
      components: {
        Chart
      },
      data() {
        return {
          appPath: process.env.API_BASE_URL,
          treeProps: {
            children: 'children',
            label: 'label',
            isLeaf: 'leaf'
          },
          instanceTreeProps: {
            children: 'children',
            label: 'label',
            isLeaf: 'leaf'
          },
          instanceData: [],
          relatedPeople: [],
          relatedEvents: [],
          chartName: '',
          chartData: {
            nodes: [],
            links: [],
            categories: []
          },
          // 静态分类数据 - 两级结构
          staticTreeData: [
            {
              label: '香山文化',
              leaf: false,
              children: [
                { label: '时间', leaf: false },
                { label: '相关人物', leaf: false }
              ]
            },
            {
              label: '文化遗存',
              leaf: false,
              children: [
                { label: '人造遗存', leaf: false },
                { label: '自然遗存', leaf: false }
              ]
            }
          ],
          // 分类下的实例数据（直接展示最终实例）
          categoryInstances: {
            '时间': [
              { label: '1949年3月25日：进驻香山', instanceName: '1949年3月25日进驻香山' },
              { label: '1949年6月19日：毛泽东致宋庆龄信', instanceName: '毛泽东致宋庆龄信' },
              { label: '1949年6月21日：周恩来致宋庆龄信', instanceName: '周恩来致宋庆龄信' }
            ],
            '相关人物': [
              { label: '毛泽东', instanceName: '毛泽东_香山' },
              { label: '周恩来', instanceName: '周恩来_香山' },
              { label: '宋庆龄', instanceName: '宋庆龄' },
              { label: '邓颖超', instanceName: '邓颖超' }
            ],
            '人造遗存': [
              { label: '香山公园', instanceName: '香山公园' },
              { label: '香山故居', instanceName: '香山故居' },
              { label: '香山革命纪念地', instanceName: '香山革命纪念地' },
              { label: '香山革命纪念馆', instanceName: '香山革命纪念馆' }
            ],
            '自然遗存': [
              { label: '香山自然景观', instanceName: '香山自然景观' }
            ]
          }
        }
      },
      methods: {
        loadClasses(node, resolve) {
          if (node.level === 0) {
            // 返回根节点（香山文化、文化遗存）
            return resolve(this.staticTreeData);
          } else if (node.level === 1) {
            // 返回第二级子节点
            const parent = this.staticTreeData.find(item => item.label === node.label);
            if (parent && parent.children) {
              return resolve(parent.children);
            }
            return resolve([]);
          } else {
            // 不再展开更深层级
            return resolve([]);
          }
        },
        classClick(node) {
          const label = node.label;

          // 如果是根节点（香山文化、文化遗存），不处理
          if (label === '香山文化' || label === '文化遗存') {
            return;
          }

          // 清空右侧实例列表
          this.instanceData.splice(0, this.instanceData.length);

          // 查找该分类下的实例
          const instances = this.categoryInstances[label];
          if (instances) {
            instances.forEach(item => {
              this.instanceData.push({
                label: item.label,
                instanceName: item.instanceName || null,
                leaf: true
              });
            });
          }
        },
        instanceClick(node) {
          // 直接展示图谱
          if (node.instanceName) {
            this.chartName = node.instanceName;
            this.drawGraph();
          }
        },
        drawGraph() {
          this.$ajax.get(this.appPath + '/getRelationshipOfIns?individualName=' + this.chartName)
            .then(res => {
              this.chartData.nodes.splice(0, this.chartData.nodes.length);
              this.chartData.links.splice(0, this.chartData.links.length);
              this.chartData.categories.splice(0, this.chartData.categories.length);

              let graphData = res.data.data;

              // 节点预处理
              graphData.nodes.forEach(node => {
                node.itemStyle = null;
                node.symbolSize = 45;
                node.symbol = "circle";
                node.label = {
                  show: true,
                  color: 'white',
                  fontFamily: 'myFont, sans-serif',
                  fontSize: 13,
                  fontWeight: 'bold',
                  align: 'center',
                  verticalAlign: 'middle',
                  position: 'inside'
                };

                this.chartData.nodes.push(node);

                // 目录预处理
                let hasThisCate = false;
                this.chartData.categories.forEach(cate => {
                  if (cate.name === node.category) {
                    hasThisCate = true;
                  }
                });
                if (!hasThisCate) {
                  let obj = {};
                  obj.name = node.category;
                  obj.symbolSize = 30;
                  this.chartData.categories.push(obj);
                }
              });

              // 链接预处理
              graphData.links.forEach(link => {
                link.source = link.subject;
                link.target = link.object;
                link.value = link.predicate;
                this.chartData.links.push(link);
              });

            })
            .catch(error => {
              console.log(error);
              window.alert("生成知识图谱失败！");
            });

        }
      },
    }
</script>

<style scoped>

  .show {
    margin: 0 auto;
    width: 100%;
    height: 780px;
  }

  .classTree {
    float: left;
    position: relative;
    top: 50px;
    left: 15px;
    border-radius: 10px;
    width: 290px;
    height: 730px;
    background: rgba(0, 0, 0, 0.7);
    overflow: hidden;
  }

  .clzTreeContent {
    height: 689px;
    overflow: auto;
  }

  .instanceTree {
    float: left;
    position: relative;
    top: 50px;
    left: 30px;
    border-radius: 10px;
    width: 290px;
    height: 730px;
    background: rgba(0, 0, 0, 0.7);
    overflow: hidden;
  }

  .insTreeContent {
    height: 689px;
    overflow: auto;
  }

  .classTree p {
    font-size: 17px;
    margin: 10px 0;
    text-align: center;
    color: white;
    font-weight: bold;
  }

  .instanceTree p {
    font-size: 17px;
    margin: 10px 0;
    text-align: center;
    color: white;
    font-weight: bold;
  }

  .showInsInfo {
    float: left;
    position: relative;
    top: 50px;
    left: 45px;
    border-radius: 10px;
    width: 720px;
    height: 730px;
    background: rgba(0, 0, 0, 0.7);
  }

  .showInsInfo p {
    font-size: 17px;
    margin: 10px 0;
    text-align: center;
    color: white;
    font-weight: bold;
  }

</style>

<style>

  .classTree .el-tree {
    background: none;
    color: wheat;
  }

  .classTree .el-tree .el-tree-node__content:hover {
    background-color:	#8B4513;
  }

  .classTree .el-tree-node:focus>.el-tree-node__content {
    background-color:	#8B4513;
  }

  .classTree .el-tree-node__label {
    font-size: 15px;
  }

  .instanceTree .el-tree {
    background: none;
    color: wheat;
  }

  .instanceTree .el-tree .el-tree-node__content:hover {
    background-color:	#8B4513;
  }

  .instanceTree .el-tree-node:focus>.el-tree-node__content {
    background-color:	#8B4513;
  }

  .instanceTree .el-tree-node__label {
    font-size: 15px;
  }

  /* 子分类样式 - 带有箭头图标 */
  .instanceTree .el-tree-node__expand-icon.expanded {
    color: #FFA500;
  }

  /* 实例样式 - 没有箭头，直接可点击 */
  .instanceTree .el-tree-node.is-leaf > .el-tree-node__content .el-tree-node__expand-icon {
    display: none;
  }

</style>
