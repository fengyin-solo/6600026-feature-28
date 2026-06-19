<script setup lang="ts">
import { ref, onMounted, watch, nextTick } from 'vue';
import type { PhyloNode } from '../types';

const props = defineProps<{
  tree: PhyloNode | null;
}>();

const canvasRef = ref<HTMLCanvasElement | null>(null);

function countLeaves(node: PhyloNode): number {
  if (node.children.length === 0) return 1;
  return node.children.reduce((sum, child) => sum + countLeaves(child), 0);
}

function getMaxDepth(node: PhyloNode, current = 0): number {
  if (node.children.length === 0) return current + node.branchLength;
  return Math.max(...node.children.map(c => getMaxDepth(c, current + node.branchLength)));
}

function formatDistance(value: number): string {
  if (value === 0) return '0';
  if (value < 0.001) return value.toExponential(1);
  return value.toFixed(3);
}

function drawTree() {
  const canvas = canvasRef.value;
  if (!canvas || !props.tree) return;

  const width = 600;
  const height = 420;
  canvas.width = width;
  canvas.height = height;

  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  ctx.fillStyle = '#111827';
  ctx.fillRect(0, 0, width, height);

  const totalLeaves = countLeaves(props.tree);
  const maxDepth = getMaxDepth(props.tree) || 1;

  const leftMargin = 30;
  const rightMargin = 180;
  const topMargin = 40;
  const bottomMargin = 60;
  const treeWidth = width - leftMargin - rightMargin;
  const treeHeight = height - topMargin - bottomMargin;

  let leafIndex = 0;
  const xScale = treeWidth / maxDepth;
  const yStep = treeHeight / Math.max(totalLeaves - 1, 1);

  function drawBranchLengthLabel(startX: number, endX: number, y: number, length: number) {
    if (length <= 0) return;
    const midX = (startX + endX) / 2;
    ctx.fillStyle = '#fbbf24';
    ctx.font = '10px monospace';
    ctx.textAlign = 'center';
    ctx.fillText(formatDistance(length), midX, y - 6);
  }

  function drawNode(node: PhyloNode, x: number, y: number): { x: number; y: number } {
    const nodeX = x + node.branchLength * xScale;

    if (node.children.length === 0) {
      const leafY = topMargin + leafIndex * yStep;
      leafIndex++;

      ctx.strokeStyle = '#6ee7b7';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(x, leafY);
      ctx.lineTo(nodeX, leafY);
      ctx.stroke();

      drawBranchLengthLabel(x, nodeX, leafY, node.branchLength);

      ctx.fillStyle = '#d1d5db';
      ctx.font = '11px sans-serif';
      ctx.textAlign = 'left';
      const label = node.name.length > 20 ? node.name.substring(0, 18) + '...' : node.name;
      ctx.fillText(label, nodeX + 8, leafY + 4);

      ctx.fillStyle = '#06b6d4';
      ctx.beginPath();
      ctx.arc(nodeX, leafY, 3, 0, Math.PI * 2);
      ctx.fill();

      return { x: nodeX, y: leafY };
    }

    const childPositions = node.children.map(child => drawNode(child, nodeX, y));

    const minY = Math.min(...childPositions.map(p => p.y));
    const maxY = Math.max(...childPositions.map(p => p.y));
    const nodeY = (minY + maxY) / 2;

    ctx.strokeStyle = '#6ee7b7';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(nodeX, minY);
    ctx.lineTo(nodeX, maxY);
    ctx.stroke();

    if (x > 0 || node.branchLength > 0) {
      ctx.beginPath();
      ctx.moveTo(x, nodeY);
      ctx.lineTo(nodeX, nodeY);
      ctx.stroke();

      drawBranchLengthLabel(x, nodeX, nodeY, node.branchLength);
    }

    ctx.fillStyle = '#a78bfa';
    ctx.beginPath();
    ctx.arc(nodeX, nodeY, 2.5, 0, Math.PI * 2);
    ctx.fill();

    return { x: nodeX, y: nodeY };
  }

  drawNode(props.tree, leftMargin, topMargin);

  function drawScaleBar() {
    const barY = height - 25;
    const barStartX = leftMargin;

    let targetBarDist = maxDepth / 5;
    const magnitudes = [0.001, 0.005, 0.01, 0.05, 0.1, 0.2, 0.5, 1];
    for (const mag of magnitudes) {
      if (mag <= targetBarDist) {
        targetBarDist = mag;
      }
    }
    if (targetBarDist > maxDepth) targetBarDist = maxDepth / 2;

    const barWidth = targetBarDist * xScale;
    const barEndX = barStartX + barWidth;

    ctx.strokeStyle = '#e5e7eb';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(barStartX, barY);
    ctx.lineTo(barEndX, barY);
    ctx.stroke();

    ctx.beginPath();
    ctx.moveTo(barStartX, barY - 5);
    ctx.lineTo(barStartX, barY + 5);
    ctx.stroke();

    ctx.beginPath();
    ctx.moveTo(barEndX, barY - 5);
    ctx.lineTo(barEndX, barY + 5);
    ctx.stroke();

    ctx.fillStyle = '#e5e7eb';
    ctx.font = '11px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText(formatDistance(targetBarDist), (barStartX + barEndX) / 2, barY + 20);

    ctx.textAlign = 'left';
    ctx.fillText('距离标尺', barEndX + 12, barY + 4);
  }

  drawScaleBar();

  ctx.fillStyle = '#9ca3af';
  ctx.font = 'bold 12px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText('进化树 (Neighbor-Joining)', width / 2, 18);

  ctx.font = '10px sans-serif';
  ctx.fillStyle = '#6b7280';
  ctx.textAlign = 'right';
  ctx.fillText('分支长度表示遗传距离', width - 10, 18);
}

onMounted(() => {
  drawTree();
});

watch(() => props.tree, () => {
  nextTick(() => drawTree());
}, { deep: true });
</script>

<template>
  <div class="bg-gray-900 rounded-lg overflow-hidden">
    <div class="px-4 py-2 bg-gray-800 border-b border-gray-700 flex items-center justify-between">
      <h3 class="text-sm font-semibold text-gray-300">系统发育树</h3>
      <span class="text-xs text-gray-500">Neighbor-Joining 法</span>
    </div>
    <div class="p-4 flex justify-center">
      <canvas
        v-if="tree"
        ref="canvasRef"
        width="600"
        height="420"
        class="border border-gray-700 rounded"
      ></canvas>
      <div v-else class="flex items-center justify-center w-full h-64 text-gray-600 text-sm">
        请先加载序列并构建进化树
      </div>
    </div>
  </div>
</template>
