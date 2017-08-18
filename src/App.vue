<template>
    <div id="app" @mousemove.stop.prevent="dragingNodeName?startDragingNode($event):null"
         @mouseup.stop.prevent="stopDragingNode">
        <div class="float-left container tree-container ">
            <menuTree @dragingNode="dragingNodeController"></menuTree>
        </div>
        <div class="float-left container canvas-container">
            <canvasContent></canvasContent>
        </div>

        <!--<nodeDetail></nodeDetail>-->

        <div id="dragingNode" :class="dragingNodeName ? 'inline-block' : 'hide'" class="draging-node-container">
            <div class="draging-node-wrap">
                <span>{{ dragingNodeName }}</span>
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
                dragingNodeName: null, // 正在移动的节点名称
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
            dragingNodeController: function (e) {
                var _x = e.clientX,
                    _y = e.clientY,
                    _tar = e.target;

                this.dragingNodeDis = [_x - _tar.offsetLeft, _y - _tar.offsetTop]; //计算距离差

                this.setDragingNodePos(_x, _y); //先设置虚拟DOM的位置
                debugger;

                this.dragingNodeName = e.target.innerText;
            },
            // 开始移动
            startDragingNode: function (e) {
                console.log('move');
                this.setDragingNodePos(e.clientX, e.clientY);
            },
            // 结束移动
            stopDragingNode: function () {
                if (!this.dragingNodeName) return;
                this.dragingNodeName = null;

                console.log('up');
            },
            setDragingNodePos: function (x, y) {
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
    div.container {
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
        margin-top: 20px;
        padding: 5px;
        height: calc(100% - 30px);
    }
    .tree-container {
        width: 13%;
        margin-right: 5px;
        padding-right: 5px;
        border-right: 1px solid #CCC;
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
