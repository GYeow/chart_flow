<template>
    <div id="svgWrap" :style="svgWrap">
        <svg width="100%" height="100%">
            <!--<g v-for="item in nodes"  :style="{transform: `translate(${margin.left}px, ${margin.top}px)`}">-->
            <g v-for="item in nodes" style="font-size: 14pt;">
                <foreignObject class="node" :x="item.x" :y="item.y" width="130" height="30">
                    <body xmlns="http://www.w3.org/1999/xhtml">
                    <div :style="nodeWrap">
                        <span>{{ item.name }}</span>
                    </div>
                    </body>
                </foreignObject>
            </g>
        </svg>
    </div>
</template>

<script>
    import * as d3 from 'd3'

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
            createNode(name, position) {
                this.nodes.push({
                    name: name,
                    x: position[0],
                    y: position[1]
                });
            }
        }
    }


</script>

<style scoped>

</style>
