<template>
    <div id="app" @mousemove.stop.prevent="movingNodeName?startMovingNode($event):null" @mouseup.stop.prevent="stopMovingNode">
        <div class="float-left container tree-container ">
            <menuTree @movingNode="movingNodeController"></menuTree>
        </div>
        <div class="float-left container canvas-container">
            <canvasContent></canvasContent>
        </div>

        <!--<nodeDetail></nodeDetail>-->

        <div id="MovingNode" :class="movingNodeFlag && movingNodeName ? 'inline-block' : 'hide'" class="moving-node-container">
            <div class="moving-node-wrap">
                <span>{{ movingNodeName }}</span>
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
                movingNodeFlag: false,
                movingNodeName: null,
                movingNodePos: [0, 0],
            }
        },
        components: {
            menuTree,
            canvasContent,
            nodeDetail
        },
        methods: {
            // 管理移动菜单节点
            movingNodeController: function (e) {
                debugger;
                this.setMovingNodePos();
                this.movingNodeName = e.target.innerText;
            },
            // 开始移动
            startMovingNode: function (e) {
                this.movingNodeFlag = true;
                console.log('move');
                this.setMovingNodePos(e.clientX, e.clientY);
            },
            // 结束移动
            stopMovingNode: function () {
                if (!this.movingNodeName) return;
                this.movingNodeName = null;
                console.log('up');

            },
            setMovingNodePos: function (left, top) {
                var _style = document.getElementById('MovingNode').style;
                _style.left = left + 'px';
                _style.top = top + 'px';
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
    .moving-node-container {
        position: absolute;
    }
    .moving-node-wrap {
        padding:5px 15px;
        border: 1px solid #222;
    }
</style>
