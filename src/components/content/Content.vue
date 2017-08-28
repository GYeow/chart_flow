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
                        <path class="target-marker"
                              :d="key==linkingLinkId ? 'M 5 0 L 0 3.6327126400268037 L 5 7.265425280053607 Z' : ''"
                              :transform="key==linkingLinkId ? 'translate(' + (val.end[0]-3.63) + ',' + val.end[1] + ') scale(1,1) rotate(-90)' : ''"></path>
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
                                 :class="{'is-connectable': linkingNodeId && linkingNodeId!=key && val.connectable && !relatedNodes[key]}">
                                <span>{{ val.name }}</span>
                                <div class="pane-port-list in">
                                    <div v-for="(value, inputId) in val.inputPorts" class="pane-port-wrap"
                                         :style="'width: '+ 100/(val.input+1) + '%;'">
                                        <div class="pane-port in"
                                             :class="{'is-connectable': linkingNodeId && linkingNodeId!=key && !relatedNodes[key] && !value.startId,
                                                        'is-connected': value.startId}"
                                             :data-id="inputId">
                                            <span class="port-magnet"></span>
                                        </div>
                                    </div>
                                </div>
                                <div class="pane-port-list out">
                                    <div v-for="(value, outputId) in val.outputPorts" class="pane-port-wrap"
                                         :style="'width: '+ 100/(val.output+1) + '%;'">
                                        <div class="pane-port out"
                                             :class="{'is-connecting': linkingPortId==outputId}"
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
                relatedNodes: {}, //直接或间接输出为linkingNode的节点(禁止连接)
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
                let that = this,
                    $ele = e.target.tagName == 'foreignObject' ? $(e.target) : $(e.target).parents('foreignObject'),
                    _doc = document.getElementById('svgWrap');

                let nodeId = $ele.attr('data-id'),
                    nodeInfo = that.nodes[nodeId],
                    notAlone = !!that.linksDir[nodeId];

                console.log('down-node')

                // 减少鼠标移动时的计算量
                // 存储与当前节点相关连线的linkId
                let relatedLinks = [];
                if (notAlone) {
                    for (let relatedNodeId in that.linksDir[nodeId]) {
                        if (!relatedNodeId) continue;

                        let linkIdArr = that.linksDir[nodeId][relatedNodeId];

                        for (let i in linkIdArr) {
                            relatedLinks.push({
                                linkId: linkIdArr[i],
                                isStart: ('' + that.links[linkIdArr[i]].startId).slice(0, -2) == nodeId // 当前节点是否是连线的起始节点
                            });
                        }
                    }
                }
                // 用于计算鼠标移动时与初始位置的偏差
                var initX = nodeInfo.x, initY = nodeInfo.y, // 初始节点坐标
                    clientX = e.clientX, clientY = e.clientY, //初始鼠标坐标
                    maxX = _doc.offsetWidth - $ele.attr('width'), // 禁止拖拽出边界
                    maxY = _doc.offsetHeight - $ele.attr('height');

                _doc.onmousemove = function(ev) {
                    console.log('moving-node')
                    ev.stopPropagation();

                    let deltaX = Math.min(Math.max(0, ev.clientX - clientX + initX), maxX) - initX,
                        deltaY = Math.min(Math.max(0, ev.clientY - clientY + initY), maxY) - initY;
                    nodeInfo.x += deltaX;
                    nodeInfo.y += deltaY;

                    if (notAlone) {
                        // 同时移动连线
                        for (let i in relatedLinks) {
                            let linkInfo = relatedLinks[i],
                                linkId =  linkInfo.linkId,
                                newLinkInfo = {
                                    id: linkId
                                },
                                initPos = !!linkInfo.isStart ? that.links[linkId].start : that.links[linkId].end;

                            initPos[0] += deltaX;
                            initPos[1] += deltaY;
                            newLinkInfo[linkInfo.isStart ? 'start' : 'end'] = initPos;

                            that.updateLink(newLinkInfo);

                        }
                    }

                    clientX = ev.clientX;
                    clientY = ev.clientY;
                    initX = nodeInfo.x;
                    initY = nodeInfo.y;
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
            linkingNodes(e, panePort) {
                let that = this,
                    $panePort = panePort;
                that.linkingPortId = $panePort.attr('data-id');
                that.linkingNodeId = ('' + that.linkingPortId).slice(0, -2);
                let offset = $panePort.offset(),
                    svgOffset = $panePort.closest('svg').offset(),
                    _dis = [offset.left + 5 - svgOffset.left, offset.top + 5 - svgOffset.top], // 初始距离差即为连线起点
                    _doc = document.getElementById('svgWrap'),
                    maxX = _doc.offsetWidth,
                    maxY = _doc.offsetHeight;

                that.creatLink(_dis); // mousedown时绘制连线

                that.relatedNodes = that.getAllSourceId(that.linkingNodeId); // 禁止生成有向环图

                _doc.onmousemove = function(ev) {
                    console.log('linking');
                    ev.stopPropagation();
                    // 移动连线
                    that.updateLink({
                        end: [ev.clientX - svgOffset.left, ev.clientY - svgOffset.top]
                    });
                };
                _doc.onmouseup = function(e) {
                    console.log('link-end');

                    this.onmousemove=null;
                    this.onmouseup=null;

                    let $tar = $(e.target),
                        $endPort = $tar.parent(),
                        endOffset = $endPort.offset(),
                        isConnected = $tar.hasClass('port-magnet') && $endPort.hasClass('is-connectable');

                    // 修正连线终点位置 并 添加连线终止接口ID
                    if (isConnected) {
                        that.updateLink({
                            end: [endOffset.left + 5 - svgOffset.left, endOffset.top + 5 - svgOffset.top],
                            endId: $endPort.attr('data-id')
                        });
                    }

                    // 根据是否连接到另一节点的接口 添加字典项 / 从画布中删除连线
                    that.updateLinkDir(!isConnected);

                    // 清空临时数据
                    that.linkingPortId = '';
                    that.linkingNodeId = '';
                    that.linkingLinkId = '';
                    that.relatedNodes = {};
                };
            },
            /**
             * 新建节点至画布
             * @param {[Object]}  nodeInfo    [节点信息]
             */
            createNode(nodeInfo) {
                let index = this.getRandomIndex(true);

                $.extend(nodeInfo, {
                    index: index,
                    connectable: true, //是否还有未连接的输入接口
                    sourceId: '',
                    targetId: '',
                    inputPorts : {},
                    outputPorts: {},
                    relatedNodes: {} // 有直接或间接关系的节点，用于禁止产生循环
                },true);

                // 根据输入、输出个数生成相应的接口id
                for (let i = 0, inputNum = nodeInfo.input; i < inputNum + nodeInfo.output; i++) {

                    let targetObj = i < inputNum ? nodeInfo.inputPorts : nodeInfo.outputPorts;

                    targetObj[index + ('0' + i).slice(-2)] = i < inputNum ? {startId: ''} : {endId: ''};

                    targetObj[index + ('0' + i).slice(-2)].text = '';
                }
                this.$set(this.nodes, index, nodeInfo); //添加至本地节点列表
            },
            /**
             * 新建连线至画布
             * @param {[Array]}  pos    [起点坐标]
             * @param {[String]} portId [起点接口id, 默认为linkingPortId]
             */
            creatLink(pos, portId) {
                console.log('create-link');
                let linkInfo = this.getBezierCurve(pos, pos),
                    index = this.getRandomIndex(false),
                    _portId = portId || this.linkingPortId;
                this.linkingLinkId = index;

                linkInfo.startId = _portId; //起始接口id
                linkInfo.endId = '';

                this.$set(this.links, index, linkInfo); //添加至本地连线列表
            },
            /**
             * 更新连线位置
             * @param {[Object]} opt   [需更新的连线信息，若未传连线id则默认为linkingNodeId]
             */
            updateLink(opt) {
                if (!opt) return;

                let linkId = opt.id || this.linkingLinkId,
                    newLinkInfo = this.links[linkId];

                if (!newLinkInfo) return;

                $.extend(newLinkInfo, opt, true);

                // 若修改连线起止位置，则需重新计算路径
                if (opt.start || opt.end) $.extend(newLinkInfo, this.getBezierCurve(newLinkInfo.start, newLinkInfo.end), true);

//                this.$set(this.links, linkId, newLinkInfo); //更新本地连线数据
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
                    path: ['M', x1, y1, 'Q', x1, 30 + (y1 - y2 > 0) *20 + y1, (x1 + x2) / 2, (y1 + y2) / 2, 'T', x2, y2].join(' ')
                };
            },
            /**
             * 更新字典项便于查找连线
             * @param {[Boolean]}  isDel    [添加为false; 删除为true; 默认为添加]
             * @param {[String]}   linkId   [连线id; 默认为linkingLinkId]
             */
            updateLinkDir(isDel, linkId) {
                let _isDel = isDel || false,
                    _linkId = linkId || this.linkingLinkId;

                if (_isDel) {
                    delete this.links[_linkId]; // 从画布中删除连线
                    return;
                }

                // 添加至字典项
                let linkInfo = this.links[_linkId],
                    newDir = {},
                    sourceId = ('' + linkInfo.startId).slice(0, -2), //连线的起始节点Id
                    targetId = ('' + linkInfo.endId).slice(0, -2); //连线的结束节点Id

                $.extend(newDir, this.linksDir, true);

                newDir[sourceId] = newDir[sourceId] || {};
                newDir[targetId] = newDir[targetId] || {};

                newDir[sourceId][targetId] = newDir[sourceId][targetId] || [];
                newDir[targetId][sourceId] = newDir[targetId][sourceId]  || [];

                newDir[sourceId][targetId].push(_linkId);
                newDir[targetId][sourceId].push(_linkId);

                let targetNode = this.nodes[targetId];

                this.$set(this.linksDir, sourceId, newDir[sourceId]); //更新本地连线数据
                this.$set(this.linksDir, targetId, newDir[targetId]);
                this.nodes[sourceId].outputPorts[linkInfo.startId].endId = linkInfo.endId;
                targetNode.inputPorts[linkInfo.endId].startId = linkInfo.startId;

                // 同步目标节点的connectable属性
                let connectable = false;

                for (let port in targetNode.inputPorts) {
                    if (port && targetNode.inputPorts[port] && !targetNode.inputPorts[port].startId) {
                        connectable = true;
                        break;
                    }
                }

                targetNode.connectable = connectable;
            },
            /**
             * 查找指定id节点的父级节点
             * @param {[String]}   id       [待查找的节点ID]
             * @return {[Object]}  nodes    [指定id节点的父级节点集合]
             */
            getSourceId(id) {
                let linksInfo = this.linksDir[id];

                if (!linksInfo || !Object.keys(linksInfo).length) return {};

                let links = []; //与指定id节点相关的连线

                for (let keys in linksInfo) {
                    Array.prototype.push.apply(links, linksInfo[keys])
                }

                let nodes = {}; //指定id节点的父级节点
                for (let i in links) {
                    let link = this.links[links[i]];
                    if ((link.endId).slice(0, -2) == id) {
                        nodes[(link.startId).slice(0, -2)] = true
                    }
                }
                return nodes
            },
            /**
             * 查找指定id节点的所有祖辈节点
             * @param {[String]}   id       [待查找的节点ID]
             * @return {[Object]}  nodes    [指定id节点的祖辈节点集合]
             */
            getAllSourceId(id) {
                let nodes = {},
                    parentIds = this.getSourceId(id);
                if (Object.keys(parentIds || {}).length) {
                    $.extend(nodes, parentIds, true);
                    for (let keys in parentIds) {
                        $.extend(nodes, this.getAllSourceId(keys), true);
                    }
                }
                return nodes;
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
    }
    .pane-port.out {
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
        width: 2px;
        height: 0;
        margin-top: 0px;
        margin-right: -3px;
        border-style: solid;
        border-width: 5px 4px 0;
        border-color: grey transparent transparent;
        background-color: transparent;
        border-radius: 0;
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
    .pane-link-connecting.focused .connector,
    .pane-link-connecting:hover .connector,
    .pane-link.focused .connector,
    .pane-link:hover .connector {
        stroke: rgba(128,128,128,.6);
        stroke-width: 4px;
    }
    .pane-link .target-marker,
    .pane-link-connecting .target-marker {
        fill: grey;
        stroke: grey;
        stroke-width: 1px;
    }
</style>
