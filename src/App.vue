<script setup lang="ts">
import { ref } from 'vue';
import X6Graph from './components/X6Graph.vue';

type NodeItem = {
  connectorKey?: string;
  operation?: string;
  nodeConfigId: string;
  title?: string;
  inputMappings?: any[];
};
type EdgeItem = { fromId: string; toId: string; name?: string };
type GraphData = {
  apiName?: string;
  name?: string;
  description?: string;
  nodes?: NodeItem[];
  nodeEdges?: EdgeItem[];
};

const url = ref<string>('');
const graphData = ref<GraphData | null>(null);
const loading = ref(false);
const error = ref<string | null>(null);

const load = async () => {
  if (!url.value) {
    error.value = '请输入 URL';
    return;
  }
  loading.value = true;
  error.value = null;
  try {
    const res = await fetch(url.value, { headers: { Accept: 'application/json' } });
    if (!res.ok) throw new Error(`请求失败 ${res.status}`);
    const data = await res.json();
    graphData.value = data;
  } catch (err: any) {
    error.value = err?.message || String(err);
  } finally {
    loading.value = false;
  }
};

const useSample = () => {
  graphData.value = {
    apiName: 'Fc1nLZ7_CfU_if',
    name: '测试部署-{{$date.now}}',
    description: '{{$lorem.paragraph(max=3)}}',
    nodes: [
      {
        connectorKey: 'generic',
        operation: 'manualTrigger',
        nodeConfigId: 'trigger',
        title: 'manualTrigger',
      },
      {
        connectorKey: 'generic',
        operation: 'if',
        nodeConfigId: 'if',
        title: 'if',
        inputMappings: [
          {
            apiName: 'condition',
            source: 'CONDITION',
            compositeConditionMapping: {
              logicOperator: 'OR',
              children: [
                {
                  logicOperator: 'AND',
                  conditions: [
                    {
                      left: {
                        source: 'FIXED_VALUE',
                        fixedValue: 1,
                      },
                      right: {
                        source: 'FIXED_VALUE',
                        fixedValue: 1,
                      },
                      comparator: 'eq',
                    },
                  ],
                },
              ],
            },
          },
        ],
      },
      {
        connectorKey: 'generic',
        operation: 'doNothing',
        nodeConfigId: 'doNothing',
        title: 'doNothing-true',
      },
      {
        connectorKey: 'generic',
        operation: 'doNothing',
        nodeConfigId: 'doNothing1',
        title: 'doNothing-false',
      },
      {
        connectorKey: 'generic',
        operation: 'end',
        nodeConfigId: 'end',
        title: 'end',
      },
    ],
    nodeEdges: [
      { fromId: 'trigger', toId: 'if' },
      { name: 'true', fromId: 'if', toId: 'doNothing' },
      { name: 'false', fromId: 'if', toId: 'doNothing1' },
      { fromId: 'doNothing', toId: 'doNothing1' },
      { fromId: 'doNothing', toId: 'end' },
      { fromId: 'doNothing1', toId: 'end' },
    ],
  };
};
</script>

<template>
  <div class="app">
    <header class="header">
      <h2>X6 Graph Viewer</h2>
    </header>

    <div class="controls">
      <input
        v-model="url"
        class="url-input"
        placeholder="输入获取节点信息的 URL，例如 https://example.com/api/graph"
      />
      <button @click="load" :disabled="loading">加载</button>
      <button @click="useSample">示例数据</button>
      <span v-if="loading" class="status">加载中…</span>
      <span v-if="error" class="error">错误：{{ error }}</span>
    </div>

    <div class="graph-area">
      <X6Graph :graphData="graphData" />
    </div>
  </div>
</template>

<style scoped>
.app {
  max-width: 1100px;
  margin: 20px auto;
  padding: 12px;
}
.header h2 {
  margin: 0 0 12px 0;
}
.controls {
  display: flex;
  gap: 8px;
  align-items: center;
  margin-bottom: 12px;
}
.url-input {
  flex: 1;
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
}
button {
  padding: 8px 12px;
  border-radius: 6px;
  background: #5f95ff;
  color: white;
  border: none;
  cursor: pointer;
}
button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
.status {
  margin-left: 8px;
  color: #666;
}
.error {
  color: #c92a2a;
  margin-left: 8px;
}
.graph-area {
  min-height: 520px;
}
</style>
