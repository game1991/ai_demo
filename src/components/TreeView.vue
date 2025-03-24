<template>
    <div>
        <!-- 处理原始JSON数据 -->
        <div v-if="treeData.isRawJson" class="json-view">
            <h3>JSON数据</h3>
            <div class="json-content">
                <div class="json-formatter">
                    <div v-for="(item, index) in formattedJson" :key="index" class="json-line">
                        <div v-if="item.collapsible" class="toggle-btn" @click="toggleJsonNode(item.path)"
                            :class="{ 'collapsed': item.collapsed }">
                            {{ item.collapsed ? '▶' : '▼' }}
                        </div>
                        <div v-else class="toggle-btn-placeholder"></div>
                        <div class="json-line-content" :style="{ 'padding-left': `${item.indent * 20}px` }"
                            :class="{ 'hidden': item.hidden }">
                            <span v-if="item.key" class="json-key">"{{ item.key }}":</span>
                            <span class="json-value" :class="item.type">
                                {{ item.value }}
                            </span>
                            <span v-if="item.comma" class="json-comma">,</span>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- 处理原始文本数据 -->
        <div v-else-if="treeData.isRawText" class="text-view">
            <h3>文本数据</h3>
            <pre class="text-content">{{ treeData.content }}</pre>
        </div>

        <!-- 处理树形结构数据 -->
        <div v-else>
            <div v-for="(node, index) in normalizedData.items" :key="index">
                <div class="tree-node" :data-level="node.level" :style="{ '--tree-level': node.level }"
                    @click="toggleNode(node)">
                    <span class="node-icon" :class="{ 'is-folder': node.items && node.items.length }">
                        {{ node.items && node.items.length ? (node.expanded ? '📂' : '📁') : '📄' }}
                    </span>
                    <span class="tree-connector">{{ getConnector(node, index) }}</span>
                    {{ node.text }}
                    <span v-if="node.items && node.items.length" class="stats">
                        ({{ getStats(node) }})
                    </span>
                </div>
                <transition name="expand">
                    <TreeView v-if="node.items && node.expanded" :treeData="{ items: node.items }"
                        :parentLevel="node.level" />
                </transition>
            </div>
        </div>
    </div>
</template>

<script>
import { ref, reactive, computed } from 'vue'
export default {
    setup(props) {
        const state = reactive({
            expandedNodes: new Map(),
            jsonExpandedNodes: new Map()
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

        const getStats = (node) => {
            let folders = 0
            let files = 0
            node.items.forEach(item => {
                if (item.items && item.items.length) {
                    folders++;
                } else {
                    files++;
                }
            });
            const parts = [];
            if (folders > 0) parts.push(`${folders} 个目录`);
            if (files > 0) parts.push(`${files} 个文件`);
            return parts.join('，');
        }

        // 格式化JSON数据，美化显示
        const formatJson = (json) => {
            try {
                if (typeof json === 'string') {
                    json = JSON.parse(json);
                }
                return JSON.stringify(json, null, 2);
            } catch (e) {
                console.error('JSON格式化错误:', e);
                return JSON.stringify(json);
            }
        }

        // 切换JSON节点的展开/折叠状态
        const toggleJsonNode = (path) => {
            // 默认状态是展开的，所以当Map中没有值时，认为是展开状态
            // 只有明确设置为false时才是折叠状态
            const currentState = state.jsonExpandedNodes.get(path);
            // 如果当前没有状态（undefined），则设置为折叠状态（false）
            // 如果当前是折叠状态（false），则设置为展开状态（true）
            // 如果当前是展开状态（true），则设置为折叠状态（false）
            state.jsonExpandedNodes.set(path, currentState === undefined ? false : !currentState);
        }

        // 解析JSON并创建格式化的行数据
        const formattedJson = computed(() => {
            if (!props.treeData?.isRawJson || !props.treeData?.content) {
                return [];
            }

            let json = props.treeData.content;
            if (typeof json === 'string') {
                try {
                    json = JSON.parse(json);
                } catch (e) {
                    console.error('JSON解析错误:', e);
                    return [{ value: String(json), type: 'string', indent: 0 }];
                }
            }

            const lines = [];
            const parseValue = (value, key = null, path = '', indent = 0, isLast = false) => {
                const type = Array.isArray(value) ? 'array' : typeof value;
                const isCollapsible = type === 'object' || type === 'array';
                // 默认展开状态，只有明确设置为false时才是折叠状态
                const collapsed = state.jsonExpandedNodes.get(path) === false;

                if (value === null) {
                    lines.push({
                        key,
                        value: 'null',
                        type: 'null',
                        indent,
                        comma: !isLast,
                        path,
                        hidden: false
                    });
                    return;
                }

                if (type === 'object' || type === 'array') {
                    const isEmpty = Object.keys(value).length === 0;
                    const openBracket = type === 'array' ? '[' : '{';
                    const closeBracket = type === 'array' ? ']' : '}';

                    lines.push({
                        key,
                        value: openBracket + (isEmpty ? closeBracket : ''),
                        type,
                        indent,
                        comma: isEmpty && !isLast,
                        collapsible: !isEmpty,
                        collapsed,
                        path,
                        hidden: false
                    });

                    if (!isEmpty && !collapsed) {
                        const entries = Object.entries(value);
                        entries.forEach(([k, v], i) => {
                            const newPath = `${path}.${k}`;
                            parseValue(v, type === 'array' ? null : k, newPath, indent + 1, i === entries.length - 1);
                        });

                        lines.push({
                            value: closeBracket,
                            type,
                            indent,
                            comma: !isLast,
                            path,
                            hidden: false
                        });
                    }
                } else {
                    let displayValue = value;
                    if (type === 'string') {
                        displayValue = `"${value}"`;
                    } else if (type === 'undefined') {
                        displayValue = 'undefined';
                    }

                    lines.push({
                        key,
                        value: displayValue,
                        type,
                        indent,
                        comma: !isLast,
                        path,
                        hidden: false
                    });
                }
            };

            parseValue(json, null, 'root', 0, true);

            // 处理折叠状态下的隐藏行
            for (let i = 0; i < lines.length; i++) {
                const line = lines[i];
                if (line.collapsible && line.collapsed) {
                    // 记录当前折叠节点的缩进级别
                    const currentIndent = line.indent;
                    let j = i + 1;
                    // 隐藏所有缩进级别大于当前折叠节点的行，直到遇到同级或更低级别的节点
                    while (j < lines.length && lines[j].indent > currentIndent) {
                        lines[j].hidden = true;
                        j++;
                    }
                }
            }

            return lines;
        });

        const normalizedData = computed(() => {
            // 如果是原始JSON或文本，不需要规范化
            if (props.treeData.isRawJson || props.treeData.isRawText) {
                return { items: [] }
            }

            if (!props.treeData || !Array.isArray(props.treeData.items)) {
                console.error('无效的树形数据结构:', props.treeData)
                return { items: [] }
            }
            return {
                ...props.treeData,
                items: props.treeData.items.map((item, index) => ({
                    ...item,
                    id: `${props.parentLevel}-${index}`,
                    level: props.parentLevel + 1,
                    expanded: !!state.expandedNodes.get(`${props.parentLevel}-${index}`),
                    items: Array.isArray(item.items) ? item.items : []
                }))
            }
        })

        return {
            state,
            normalizedData,
            toggleNode,
            getConnector,
            getStats,
            formatJson,
            formattedJson,
            toggleJsonNode
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

.stats {
    font-size: 12px;
    color: #666;
    margin-left: 8px;
}

/* JSON显示样式 */
.json-view,
.text-view {
    padding: 16px;
    border-radius: 8px;
    background-color: #f8f8f8;
    margin-bottom: 24px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.json-view h3,
.text-view h3 {
    margin-top: 0;
    color: #42b883;
    font-size: 18px;
    margin-bottom: 12px;
    font-weight: 600;
}

.json-content,
.text-content {
    font-family: 'Consolas', 'Monaco', monospace;
    background-color: #fff;
    padding: 16px;
    border-radius: 6px;
    border: 1px solid #e0e0e0;
    overflow: auto;
    max-height: 600px;
    font-size: 14px;
    line-height: 1.6;
    box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.05);
}

.text-content {
    white-space: pre-wrap;
    word-break: break-word;
    color: #555;
}

/* JSON格式化样式 */
.json-formatter {
    font-family: 'Consolas', 'Monaco', monospace;
    font-size: 14px;
    line-height: 1.6;
}

.json-line {
    display: flex;
    align-items: flex-start;
    min-height: 22px;
    transition: background-color 0.2s ease;
}

.json-line:hover {
    background-color: rgba(66, 184, 131, 0.05);
}

.json-line-content {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    transition: all 0.3s ease;
}

.json-line-content.hidden {
    display: none;
}

.toggle-btn {
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    color: #666;
    user-select: none;
    font-size: 12px;
    transition: all 0.2s ease;
    border-radius: 3px;
}

.toggle-btn-placeholder {
    width: 20px;
}

.toggle-btn:hover {
    color: #42b883;
    background-color: rgba(66, 184, 131, 0.1);
}

.json-key {
    color: #881391;
    margin-right: 4px;
    font-weight: 500;
}

.json-value {
    margin-right: 2px;
    transition: color 0.2s ease;
}

.json-comma {
    color: #666;
}

.json-value.string {
    color: #c41a16;
}

.json-value.number {
    color: #1c00cf;
    font-weight: 500;
}

.json-value.boolean {
    color: #0d22aa;
    font-weight: 500;
}

.json-value.null {
    color: #808080;
    font-style: italic;
}

.json-value.object,
.json-value.array {
    color: #444;
    font-weight: 600;
}

/* 树形结构和JSON的展开/折叠动画 */
.expand-enter-active,
.expand-leave-active {
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        max-height: 2000px;
        overflow: hidden;
        opacity: 1;
    }
    
    .expand-enter-from,
    .expand-leave-to {
        max-height: 0;
        opacity: 0;
        margin: 0;
        padding: 0;
}
</style>