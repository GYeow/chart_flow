<template>
    <div id="app" @mousemove.stop.prevent="dragingNode?startDragingNode($event):null"
         @mouseup.stop.prevent="stopDragingNode">
        <aside class="float-left container tree-container ">
            <menuTree @dragingNode="dragingNodeController"></menuTree>
        </aside>
        <div class="float-left container canvas-container">
            <canvasContent ref="svgMethod"></canvasContent>
        </div>

        <!--<nodeDetail></nodeDetail>-->

        <div id="dragingNode" :class="dragingNode ? 'inline-block' : 'hide'" class="draging-node-container">
            <div class="draging-node-wrap">
                <span>{{ dragingNode && dragingNode.name }}</span>
            </div>
        </div>
    </div>
</template>

<script>
    import menuTree from './components/tree/Tree.vue'
    import canvasContent from '@/components/content/Content.vue'
    import nodeDetail from '@/components/detail/Detail.vue'

    import $ from 'jquery'

    export default {
        name: 'app',
        data () {
            return {
                dragingNode: null, // 正在移动的节点名称
                dragingNodeDis: [0, 0] // 移动的鼠标坐标与初始DOM之间的距离差[x, y]
            }
        },
        components: {
            menuTree,
            canvasContent,
            nodeDetail
        },
        methods: {
            // 管理移动菜单节点
            dragingNodeController(e) {
                var _x = e.clientX,
                    _y = e.clientY,
                    _tar = e.target;

                this.dragingNodeDis = [_x - _tar.offsetLeft, _y - _tar.offsetTop]; //计算距离差

                this.setDragingNodePos(_x, _y); //先设置虚拟DOM的位置

                var _td = $(_tar).is('td') ? $(_tar) : $(_tar).parents('td');
                this.dragingNode = {
                    id: _td.attr('data-id'),
                    name: e.target.innerText
                };
            },
            // 开始移动
            startDragingNode(e) {
                this.setDragingNodePos(e.clientX, e.clientY);
            },
            // 结束移动
            stopDragingNode() {
                if (!this.dragingNode) return;

                var _svg = document.getElementById('svgWrap'),
                    _nodeDOM = document.getElementById('dragingNode');

                var dis = [_nodeDOM.offsetLeft - _svg.offsetLeft, _nodeDOM.offsetTop - _svg.offsetTop],
                    center = [_nodeDOM.offsetWidth / 2 + dis[0], _nodeDOM.offsetHeight / 2 + dis[1]];

                // 拖动至画布内
                if (Math.max(0, Math.min(center[0], _svg.offsetWidth)) && Math.max(0, Math.min(center[0], _svg.offsetHeight))){
                    this.$refs.svgMethod.getNodeInfo(this.dragingNode, dis);
                }

                this.dragingNode = null;
            },
            setDragingNodePos(x, y) {
//                TODO
//                var _app = document.getElementById('app'),
//                _style = document.getElementById('dragingNode').style;
//                // 考虑初始距离差
//                _style.left = x - this.dragingNodeDis[0] + _app.offsetLeft + 'px';
//                _style.top = y - this.dragingNodeDis[1] + _app.offsetTop + 'px';

                var _style = document.getElementById('dragingNode').style;
                _style.left = x + 'px';
                _style.top = y + 'px';
            }
        }
    }
</script>

<style>
    html, body {
        margin: 0;
        padding: 0;
        height: 100%;
    }
    .float-left {
        float: left;
    }
    .inline-block {
        display: inline-block;
    }
    .hide {
        display: none;
    }
    .relative {
        position: relative;
    }
    .icon-arrow-down {
        border: 5px solid transparent;
        border-top-color: #999;
    }
    .icon-arrow-right {
        border: 5px solid transparent;
        border-left-color: #999;
    }
    .container {
        display: inline-flex;
        height: 100%;
    }
</style>

<style scoped>
    #app {
        font-family: 'Avenir', Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        text-align: center;
        color: #2c3e50;
        padding: 5px;
        padding-top: 20px;
        height: calc(100% - 30px);
    }
    .tree-container {
        width: 13%;
        margin-right: 5px;
        padding-right: 5px;
        border-right: 1px solid #CCC;
        overflow: auto;
    }
    .canvas-container {
        width: 85%;
        border: 1px solid black;
    }
    .draging-node-container {
        position: absolute;
    }
    .draging-node-wrap {
        padding:5px 15px;
        border: 1px solid #222;
    }
</style>
