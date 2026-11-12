<template>
  <!-- <g class="layer-marker" v-if="direction === 'horizontal'">
    <text :x="canvas.x + 5" :y="y - 5" text-anchor="start" :fill="color">{{
      label
    }}</text>
    <line
      :x1="canvas.x"
      :y1="y"
      :x2="canvas.x + canvas.width"
      :y2="y"
      :stroke="color"
      :stroke-opacity="strokeOpacity"
      :stroke-width="strokeWidth"
      :stroke-dasharray="strokeDasharray"
    />
  </g> -->
  <g class="layer-marker">
    <text
      :x="y + 10"
      :y="canvas.y + 5"
      text-anchor="start"
      :fill="color"
      style="writing-mode: tb"
      >{{ label }}</text
    >
    <line
      :x1="y"
      :y1="canvas.y"
      :x2="y"
      :y2="canvas.y + canvas.height"
      :stroke="color"
      :stroke-opacity="strokeOpacity"
      :stroke-width="strokeWidth"
      :stroke-dasharray="strokeDasharray"
    />
  </g>
</template>

<script setup lang="ts">
import { useChart } from "vue3-charts";
import { ref, defineComponent, watch } from "vue";

export interface MarkerProps {
  value: number;
  label: string;
  color: string;
  strokeWidth: number;
  strokeDashArray: number[];
}

const props = withDefaults(defineProps<MarkerProps>(), {
  value: 0,
  label: "",
  color: "red"
});
const chart = useChart();
const y = ref(0);
const direction = ref<Direction>("horizontal");
const canvas = ref<Canvas>(chart.canvas);

function update() {
  y.value = chart.scales.primary.at(props.value);
  canvas.value = chart.canvas;
  direction.value = chart.config.direction;
}

watch(chart.updates, update, { immediate: true });
</script>
