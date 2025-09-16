<template>
  <div class="x6-graph-container">
    <div ref="container" class="graph" />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch, nextTick } from 'vue';
import { Graph } from '@antv/x6';

interface NodeItem {
  connectorKey?: string;
  operation?: string;
  nodeConfigId: string;
  title?: string;
  inputMappings?: any[];
}

interface EdgeItem {
  fromId: string;
  toId: string;
  name?: string;
}

const props = defineProps<{
  graphData: { nodes?: NodeItem[]; nodeEdges?: EdgeItem[] } | null;
}>();

const container = ref<HTMLDivElement | null>(null);
let graph: any = null;

const createGraph = () => {
  if (!container.value) return;
  graph = new Graph({
    container: container.value,
    width: container.value.clientWidth,
    height: Math.max(400, container.value.clientHeight || 600),
    grid: true,
  });
};

const buildAndRender = (data: { nodes?: NodeItem[]; nodeEdges?: EdgeItem[] } | null) => {
  if (!graph) return;
  if (!data || !data.nodes) {
    graph.clear();
    return;
  }

  const nodes = data.nodes;
  const edges = data.nodeEdges || [];

  // build incoming map and adjacency
  const incoming = new Map<string, number>();
  const adj = new Map<string, string[]>();
  nodes.forEach((n) => incoming.set(n.nodeConfigId, 0));
  edges.forEach((e) => {
    incoming.set(e.toId, (incoming.get(e.toId) || 0) + 1);
    if (!adj.has(e.fromId)) adj.set(e.fromId, []);
    adj.get(e.fromId)!.push(e.toId);
  });

  const roots = nodes.filter((n) => (incoming.get(n.nodeConfigId) || 0) === 0).map((n) => n.nodeConfigId);
  if (roots.length === 0 && nodes.length) roots.push(nodes[0].nodeConfigId);

  const level = new Map<string, number>();
  const q: string[] = [];
  roots.forEach((r) => {
    level.set(r, 0);
    q.push(r);
  });

  while (q.length) {
    const id = q.shift()!;
    const l = level.get(id)!;
    const neighbors = adj.get(id) || [];
    neighbors.forEach((nid) => {
      const cur = level.get(nid);
      if (cur === undefined || cur > l + 1) {
        level.set(nid, l + 1);
        q.push(nid);
      }
    });
  }

  // ensure all nodes have a level
  let maxLevel = 0;
  nodes.forEach((n) => {
    const id = n.nodeConfigId;
    if (!level.has(id)) {
      level.set(id, maxLevel + 1);
      maxLevel++;
    } else {
      maxLevel = Math.max(maxLevel, level.get(id)!);
    }
  });

  // group nodes by level
  const groups = new Map<number, NodeItem[]>();
  nodes.forEach((n) => {
    const l = level.get(n.nodeConfigId) || 0;
    if (!groups.has(l)) groups.set(l, []);
    groups.get(l)!.push(n);
  });

  const nodeWidth = 140;
  const nodeHeight = 48;
  const hGap = 40;
  const vGap = 80;
  const leftPadding = 40;
  const topPadding = 40;

  const x6Nodes: any[] = [];
  const sortedLevels = Array.from(groups.entries()).sort((a, b) => a[0] - b[0]);
  sortedLevels.forEach(([l, group]) => {
    group.forEach((n, i) => {
      const x = leftPadding + i * (nodeWidth + hGap);
      const y = topPadding + l * (nodeHeight + vGap);
      const fill = n.operation === 'if' ? '#FEF3C7' : n.operation === 'end' ? '#ECFCCB' : '#FFFFFF';
      x6Nodes.push({
        id: n.nodeConfigId,
        shape: 'rect',
        x,
        y,
        width: nodeWidth,
        height: nodeHeight,
        attrs: {
          body: { fill, stroke: '#5F95FF', rx: 6 },
          label: { text: n.title || n.nodeConfigId, fill: '#000' },
        },
      });
    });
  });

  const x6Edges: any[] = edges.map((e) => {
    const xedge: any = {
      source: { cell: e.fromId },
      target: { cell: e.toId },
      attrs: {
        line: {
          stroke: '#A2B1C3',
          strokeWidth: 2,
          targetMarker: { name: 'classic', args: { size: 6 } },
        },
      },
    };
    if (e.name) {
      xedge.labels = [{ position: 0.5, attrs: { label: { text: e.name, fill: '#000' } } }];
    }
    return xedge;
  });

  try {
    graph.fromJSON({ nodes: x6Nodes, edges: x6Edges });
    (graph as any).centerContent?.();
  } catch (err) {
    // fallback: remove all cells and add nodes/edges manually
    graph.clear();
    x6Nodes.forEach((n) => graph.addNode(n));
    x6Edges.forEach((ed) => graph.addEdge(ed));
    (graph as any).centerContent?.();
  }
};

onMounted(() => {
  nextTick(createGraph);
  const onResize = () => {
    if (!graph || !container.value) return;
    graph.resize(container.value.clientWidth, container.value.clientHeight || 600);
  };
  window.addEventListener('resize', onResize);
  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize);
    graph?.dispose();
  });
});

watch(
  () => props.graphData,
  (v) => {
    buildAndRender(v || null);
  },
  { immediate: true }
);
</script>

<style scoped>
.x6-graph-container {
  width: 100%;
  height: 600px;
  border: 1px solid #e6e6e6;
  border-radius: 6px;
  overflow: hidden;
}
.graph {
  width: 100%;
  height: 100%;
}
</style>