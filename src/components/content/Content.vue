<template>
    <div id="svgWrap" :style="svgWrap">
        <svg width="100%" height="100%">
            <!--<g v-for="item in nodes"  :style="{transform: `translate(${margin.left}px, ${margin.top}px)`}">-->
            <g class="svg-viewport">
                <g v-for="item in nodes" style="font-size: 20px;cursor: all-scroll;">
                    <foreignObject :data-id="item.id" v-drag class="node" :x="item.x" :y="item.y" width="130" height="30">
                        <body xmlns="http://www.w3.org/1999/xhtml">
                            <div :style="nodeWrap">
                                <span>{{ item.name }}</span>
                                <div class="pane-port-list in">
                                    <div v-for="input in item.input" class="pane-port-wrap" style="width: 50%;">
                                        <div class="pane-port">
                                            <span class="port-magnet"></span>
                                        </div>
                                    </div>
                                </div>
                                <div class="pane-port-list out">
                                    <div v-for="output in item.output" class="pane-port-wrap" style="width: 50%;">
                                        <div class="pane-port">
                                            <span class="port-magnet"></span>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </body>
                    </foreignObject>
                </g>
            </g>
        </svg>
    </div>
</template>

<script>
    import $ from 'jquery'

    export default {
        name: 'canvasContent',
        data () {
            return {
                nodes: [],
                svgWrap: {
                    width: '100%',
                    height: '100%'
                },
                nodeWrap: {
                    width: '100%',
                    height: '100%',
                    border: '1px solid gray'
                }
            }
        },
        methods: {
            createNode(nodeInfo) {
                debugger;
                this.nodes.push(nodeInfo);
            },
            getNodeInfo(node, position) {
                var that = this,
                    url = '/data/nodeInfo';
                that.$http.get(url).then(function(res) {
                    // 这里是处理正确的回调
                    var _nodeInfo = (res.body && res.body.data && res.body.data[node.id]) || {};

                    $.extend(_nodeInfo, node || {});

                    _nodeInfo.x = position[0];
                    _nodeInfo.y = position[1];

                    that.createNode(_nodeInfo);

                }, function(res) {
                    // 这里是处理错误的回调
                    console.info('fail');
                });
            }
        },
        directives: {
            drag(el) {
                el.onmousedown = function(e){
                    e.stopPropagation();
                    e.preventDefault();

                    var _doc = document.getElementById('svgWrap'),
                        maxX = _doc.offsetWidth - el.getAttribute('width'),
                        maxY = _doc.offsetHeight - el.getAttribute('height');

                    var _dis = [e.offsetX, e.offsetY]; // 考虑初始距离差

                    _doc.onmousemove = function(ev){
                        if (ev.target.tagName !='svg') return;

                        el.setAttribute('y', Math.min(Math.max(0, ev.offsetY - _dis[1]), maxY));
                        el.setAttribute('x', Math.min(Math.max(0, ev.offsetX - _dis[0]), maxX));
                    };
                    _doc.onmouseup = function(){
                        this.onmousemove=null;
                        this.onmouseup=null;
                    };
                };
            }
        }
    }

</script>

<style scoped>
    .pane-port-list {
        clear: both;
    }
    .pane-port-list.in {
        margin-top: -34px;
    }
    .pane-port-list.out {
        margin-top: 31px;
    }
    .pane-port-wrap {
        float: left;
        height: 1px;
    }
    .pane-port {
        width: 10px;
        height: 10px;
        float: right;
        margin-right: -5px;
        border: 1px solid grey;
        border-radius: 50%;
        background-color: #fff;
        -webkit-transition: background-color .2s;
        transition: background-color .2s;
    }
    .pane-port .port-magnet {
        float: left;
        width: 20px;
        height: 20px;
        margin-top: -6px;
        margin-left: -6px;
        background-color: transparent;
    }
</style>
