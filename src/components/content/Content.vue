<template>
    <div id="svgWrap" :style="svgWrap">
        <svg :style="svgWrap">
            <g class="svg-viewport">
                <!--links-->
                <g>
                    <g v-for="(val, key) in links" class="pane-cell pane-link">
                        <span></span>
                        <path class="connector-wrap" :d="val.path"></path>
                        <path class="connector" :d="val.path"></path>
                        <path class="source-marker"></path>
                        <path class="target-marker"></path>
                        <rect class="comment-bg" width="" height="" transform=""></rect>
                        <text class="comment" y="" transform=""></text>
                    </g>
                </g>
                <!--nodes-->
                <g>
                    <g v-for="(val, key) in nodes">
                        <foreignObject :data-id="key" @mousedown.stop.prevent="draggingCtrl" class="node"
                                       :x="val.x" :y="val.y" width="180" height="30">
                            <body xmlns="http://www.w3.org/1999/xhtml">
                            <div class="pane-node-content"
                                 :class="{'is-connectable': linkingNodeId && linkingNodeId!=key && !val.source}">
                                <span>{{ val.name }}</span>
                                <div class="pane-port-list in">
                                    <div v-for="inputId in val.inputPorts" class="pane-port-wrap"
                                         :style="'width: '+ 100/(val.input+1) + '%;'">
                                        <div class="pane-port in"
                                             :class="{'is-connectable': linkingNodeId && linkingNodeId!=key && !val.source,
                                                        'is-connected': val.source}"
                                             :data-id="inputId">
                                            <span class="port-magnet"></span>
                                        </div>
                                    </div>
                                </div>
                                <div class="pane-port-list out">
                                    <div v-for="outputId in val.outputPorts" class="pane-port-wrap"
                                         :style="'width: '+ 100/(val.output+1) + '%;'">
                                        <div class="pane-port out"
                                             :class="{'is-connecting': linkingPortId ==outputId}"
                                             :data-id="outputId">
                                            <span class="port-magnet"></span>
                                        </div>
                                    </div>
                                </div>
                            </div>
                            </body>
                        </foreignObject>
                    </g>
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
                linkingNodeId: '', //正在连线的节点id
                linkingPortId: '', //正在连线的接口id
                linkingLinkId: '', //正在连线的连线id
                nodes: {},
                links: {},
                linksDir: {}, //link哈希表
                svgWrap: {
                    width: '100%',
                    height: '100%'
                }
            }
        },
        methods: {
            /**
             * 获取输入输出个数等信息
             * @param {[Object]}  node    [节点信息]
             * @param {[Array]}  position [节点坐标]
             */
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
            },
            draggingCtrl(e) {
                var $tar = $(e.target),
                    $panePort = $tar.hasClass('pane-port out') ? $tar : $tar.parents('.pane-port.out');
                if ($panePort.length) this.linkingNodes(e, $panePort); // 连线
                else this.draggingNode(e); // 移动节点位置
            },
            // 拖拽节点
            draggingNode(e) {
                var $ele = e.target.tagName == 'foreignObject' ? $(e.target) : $(e.target).parents('foreignObject'),
                    _doc = document.getElementById('svgWrap'),
                    maxX = _doc.offsetWidth - $ele.attr('width'),
                    maxY = _doc.offsetHeight - $ele.attr('height');
                console.log('down-node')
                var _dis = [e.clientX - $ele.attr('x'), e.clientY - $ele.attr('y')]; // 考虑初始距离差
                _doc.onmousemove = function(ev) {
                    console.log('moving-node')
                    ev.stopPropagation();
                    $ele.attr('y', Math.min(Math.max(0, ev.clientY - _dis[1]), maxY))
                        .attr('x', Math.min(Math.max(0, ev.clientX - _dis[0]), maxX));
                };
                _doc.onmouseup = function() {
                    console.log('up-node');
                    this.onmousemove=null;
                    this.onmouseup=null;
                };
            },
            /**
             * 连接节点
             * @param {[Object]}  e          [鼠标事件]
             * @param {[Object]}  $panePort  [输出接口的jQuery对象]
             */
            linkingNodes(e, $panePort) {
                var that = this;
                that.linkingPortId = $panePort.data('id');
                that.linkingNodeId = ('' + that.linkingPortId).slice(0, -2);

                var offset = $panePort.offset(),
                    svgOffset = $panePort.closest('svg').offset(),
                    _dis = [offset.left + 5 - svgOffset.left, offset.top + 5 - svgOffset.top], // 初始距离差即为连线起点
                    _doc = document.getElementById('svgWrap'),
                    maxX = _doc.offsetWidth,
                    maxY = _doc.offsetHeight;

                this.creatLink(_dis); // mousedown时绘制连线

                _doc.onmousemove = function(ev) {
                    console.log('linking');
                    ev.stopPropagation();
                    // 移动连线
                    that.updateLink({
                        end: [ev.clientX - svgOffset.left, ev.clientY - svgOffset.top]
                    });

                };
                _doc.onmouseup = function() {
                    console.log('link-end');

                    this.onmousemove=null;
                    this.onmouseup=null;

                    that.linkingPortId = '';
                    that.linkingNodeId = '';
                    that.linkingLinkId = '';
                };
            },
            /**
             * 新建节点至画布
             * @param {[Object]}  nodeInfo    [节点信息]
             */
            createNode(nodeInfo) {
                var index = this.getRandomIndex(true);

                nodeInfo.index = index;

                // 根据输入、输出个数生成相应的接口id
                nodeInfo.inputPorts = [];
                nodeInfo.outputPorts = [];
                for (let i = 0, inputNum = nodeInfo.input; i < inputNum + nodeInfo.output; i++) {
                    if (i < inputNum) nodeInfo.inputPorts[i] = index + ('0' + i).slice(-2);
                    else nodeInfo.outputPorts[i - inputNum] = index + ('0' + i).slice(-2);
                }
                this.$set(this.nodes, index, nodeInfo); //添加至本地节点列表
            },
            /**
             * 新建连线至画布
             * @param {[Array]}  pos    [起点坐标]
             */
            creatLink(pos) {
                console.log('create-link');
                var linkInfo = this.getBezierCurve(pos, pos),
                    index = this.getRandomIndex(false);
                this.linkingLinkId = index;

                this.$set(this.links, index, linkInfo); //添加至本地连线列表
            },
            /**
             * 更新连线位置
             * @param {[Object]} opt   [需更新的连线信息，若未传连线id则默认为linkingNodeId]
             */
            updateLink(opt) {
                if (!opt) return;

                let newLinkInfo = this.links[this.linkingLinkId];

                $.extend(newLinkInfo, opt, true);

                // 若修改连线起止位置，则需重新计算路径
                if (opt.start || opt.end) $.extend(newLinkInfo, this.getBezierCurve(newLinkInfo.start, newLinkInfo.end), true);

                this.$set(this.links, this.linkingLinkId, newLinkInfo); //更新本地连线数据
            },
            removeLink(id) {},
            /**
             * 设置画布中 节点/连线数据 的key值
             * @param {[Boolean]}  isNode    [true: 节点; false:连线]
             */
            getRandomIndex(isNode) {
                var index;
                do {
                    index = Math.floor(Math.random() * 1e4);
                } while (!!(isNode ? this.nodes[index] : this.links[index]))
                return index;
            },
            /**
             * 根据起止点坐标返回曲线(二次贝塞尔)路径
             * @param {[Array]}  start    [起点坐标]
             * @param {[Array]}  end      [终点坐标]
             */
            getBezierCurve(start, end) {
                let x1 = start[0], y1 = start[1],
                    x2 = end[0], y2 = end[1];
                return {
                    start: start,
                    end: end,
                    path: ['M', x1, y1, 'Q', x1, 30 + y1, (x1 + x2) / 2, (y1 + y2) / 2, 'T', x2, y2].join(' ')
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
    .pane-node-content {
        width: 100%;
        height: 30px;
        background-color: rgba(255,255,255,.9);
        border: 1px solid #289DE9;
        border-radius: 15px;
        font-size: 16px;
        line-height: 28px;
        -webkit-transition: background-color .2s;
        transition: background-color .2s;
        cursor: pointer;
    }
    .pane-node-content:hover,
    .pane-node.selected .pane-node-content {
        background-color: rgba(227,244,255,.9);
    }
    .pane-node-content.is-connectable {
        border-color: #39CA74;
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
        cursor: crosshair;
    }
    .pane-port-list.out .pane-port.is-connecting,
    .pane-port-list.out .pane-port:hover {
        background-color: grey;
    }
    .pane-port .port-magnet {
        float: left;
        width: 20px;
        height: 20px;
        margin-top: -6px;
        margin-left: -6px;
        background-color: transparent;
    }
    .pane-port-list.in .pane-port.is-connected {
        width: 0;
        height: 0;
        border-style: solid;
        border-width: 5px 4px 0;
        border-color: grey transparent transparent;
        background-color: transparent;
        border-radius: 0;
        margin-top: 1px;
    }
    .pane-port-list.in .pane-port.is-connectable .port-magnet {
        float: left;
        width: 40px;
        height: 40px;
        margin-top: -16px;
        margin-left: -16px;
        background-color: transparent;
        border-radius: 50%;
    }
    .pane-port-list.in .pane-port.is-connectable .port-magnet:before {
        float: left;
        width: 20px;
        height: 20px;
        margin-left: 10px;
        margin-top: 10px;
        box-sizing: content-box;
        border-radius: 50%;
        background-color: rgba(57,202,116,.6);
        content: ' ';
    }
    .pane-port-list.in .pane-port.is-connectable .port-magnet:after {
        float: left;
        clear: both;
        width: 10px;
        height: 10px;
        margin-top: -15px;
        margin-left: 15px;
        box-sizing: border-box;
        border: 1px solid #39CA74;
        border-radius: 50%;
        background-color: #FFF;
        content: ' ';
    }
    .pane-port-list.in .pane-port.is-connectable .port-magnet.is-adsorbed:before {
        width: 28px;
        height: 28px;
        margin-top: 6px;
        margin-left: 6px;
    }
    .pane-port-list.in .pane-port.is-connectable .port-magnet.is-adsorbed:after {
        margin-top: -19px;
    }
    .pane-port-list.in .pane-port.is-connected .port-magnet {
        display: none;
    }
    .pane-link .connector-wrap,
    .pane-link-connecting .connector-wrap {
        fill: none;
        stroke: rgba(255,255,255,0);
        stroke-width: 15px;
    }
    .pane-link .connector,
    .pane-link-connecting .connector {
        fill: none;
        stroke: grey;
        stroke-width: 1px;
    }
    .pane-link .comment-bg {
        fill: #F1F9FD;
        stroke: #CCC;
        stroke-dasharray: 3;
    }
</style>
