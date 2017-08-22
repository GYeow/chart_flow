<template>
    <h2 v-if="!treeData">NO DATA</h2>
    <div v-else class="tree-wrap">
        <details v-for="item in treeData" open>
            <summary>{{ item.name }}</summary>
            <table border="0" cellspacing="0" cellpadding="0">
                <tbody>
                    <tr v-for="node in item.children">
                        <td class="tree-node-td" :data-id="node.id" @mousedown.stop.prevent="nodeMousedown">
                            <span class="tree-node-label-wrapper">
                                <span class="tree-node-icon icon-data"></span>
                                <span class="tree-node-label">{{ node.name }}</span>
                            </span>
                        </td>
                    </tr>
                </tbody>
            </table>
        </details>
    </div>
</template>

<script>
    export default {
        name: 'menuTree',
        data () {
            return {
                treeData: this.treeData
            }
        },
        methods: {
            fetchData() {
                var that = this,
                    url = '/data/treeMenu';
                that.$http.get(url).then(function(res) {
                    // 这里是处理正确的回调
                    that.treeData = (res.body && res.body.data) || {};

                }, function(res) {
                    // 这里是处理错误的回调
                    console.info('fail');
                });
            },
            nodeMousedown(e) {
                this.$emit('dragingNode', e); // 移动
            }
        },
        created() {
            this.fetchData();
        },
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    @keyframes fadeInDown {
        0% {
            opacity: 0;
        }
        100% {
            opacity: 1;
        }
    }
    h1, h2 {
        font-weight: normal;
    }
    .tree-wrap {
        width: 100%;
        text-align: left;
        text-indent: 20px;
        font-size: 16px;
    }
    details {
        position: relative;
        overflow: hidden;
        margin-bottom: 8px;
    }
    details summary {
        cursor: pointer;
        outline: none;
        user-select: none;
        height: 30px;
        line-height: 30px;
        background: linear-gradient(#fff, #e1e1e1);
        border: 1px solid #ccc;
        border-radius: 5px;
    }
    details summary::-webkit-details-marker {
        display: none;
        padding-left: 8px;
        height: 20px;
        line-height: 20px;
    }
    details summary:after {
        content: "+";
        float: left;
        width: 20px;
        font-weight: bold;
        font-size: 1.5em;
    }
    details[open] summary::after {
        content: "−";
    }
    details table {
        width: 100%;
    }
    details tr {
        animation: fadeInDown 0.25s linear;
        text-indent: 38px;
    }
    details td {
        border: 1px solid #CCC;
        border-top-color: transparent;
        line-height: 28px;
    }
    .tree-node-td {
        cursor: all-scroll;
    }

</style>
