<template>
    <div>
        <div v-for="(node, index) in normalizedData.items" :key="index">
            <div class="tree-node" :data-level="node.level" :style="{ '--tree-level': node.level }" @click="toggleNode(node)">
                <span class="node-icon" :class="{ 'is-folder': node.items && node.items.length }">
                    {{ node.items && node.items.length ? (node.expanded ? '📂' : '📁') : '📄' }}
                </span>
                <span class="tree-connector">{{ getConnector(node, index) }}</span>
                {{ node.text }}
            </div>
            <transition name="expand">
                <TreeView v-if="node.items && node.expanded" :treeData="{ items: node.items }" :parentLevel="node.level" />
            </transition>
        </div>
    </div>
</template>

<script>
import { ref, reactive, computed } from 'vue'
export default {
    setup(props) {
        const state = reactive({
            expandedNodes: new Map()
        })

        const toggleNode = (node) => {
            if (!node?.items) return
            const expanded = state.expandedNodes.get(node.id)
            state.expandedNodes.set(node.id, !expanded)
        }

        const getConnector = (node, index) => {
            const isLast = index === props.treeData.items.length - 1
            return isLast ? '└─' : '├─'
        }

        const normalizedData = computed(() => ({
            ...props.treeData,
            items: props.treeData.items.map(item => ({
                ...item,
                level: props.parentLevel + 1,
                expanded: !!state.expandedNodes.get(item.id),
                items: Array.isArray(item.items) ? item.items : []
            }))
        }))

        return {
            state,
            normalizedData,
            toggleNode,
            getConnector
        }
    },
    name: 'TreeView',
    props: {
        treeData: {
            type: Object,
            required: true
        },
        parentLevel: {
            type: Number,
            default: 0
        }
    }
}
</script>

<style scoped>
.tree-node {
    position: relative;
    cursor: pointer;
    padding: 4px 8px 4px calc(12px + calc(var(--tree-level, 0) * 20px));
    margin: 0;
    font-family: monospace;
    font-size: 14px;
    line-height: 20px;
    display: flex;
    align-items: center;
    gap: 4px;
}

.node-icon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 20px;
    height: 20px;
}

.tree-connector {
    color: #666;
    margin: 0 4px;
}

.tree-node:hover {
    background: #f5f5f5;
}

.expand-enter-active,
.expand-leave-active {
    transition: all 0.3s ease;
    max-height: 1000px;
    overflow: hidden;
}

.expand-enter-from,
.expand-leave-to {
    max-height: 0;
    opacity: 0;
}
</style>