<template>
  <g class="layer-labels">
    <text
      v-for="(p, i) in bars.filter((p, i) => p.props.data[dataKeys[1]] > 0)"
      :key="i"
      :x="p.x + p.width / 2"
      :y="p.y - 5"
      text-anchor="middle"
      font-size="12"
      :fill="labelStyle?.fill"
    >
      {{ p.props.data[dataKeys[1]]?.toLocaleString().split(".")[0] }}
    </text>
  </g>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { useBars, useChart } from "vue3-charts";

export default defineComponent({
  name: "Labels",
  props: {
    dataKeys: {
      type: Object as () => [string, string],
      required: true
    },
    labelStyle: {
      type: Object,
      required: false
    }
  },
  setup(props) {
    const { bars } = useBars(props.dataKeys, { gap: 5 });
    return { bars };
  }
});
</script>
