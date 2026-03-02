<template>
  <div class="h-32 w-full">
    <canvas ref="canvas"></canvas>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from "vue";
import Chart from "chart.js/auto";
import { useCommunity } from "~/composables/useCommunity";

const props = defineProps({ community: { type: Object, required: true } }); // community должен содержать id

const { getCommunityRatingHistory } = useCommunity();
const canvas = ref<HTMLCanvasElement | null>(null);
let chart: Chart | null = null;

async function loadHistory() {
  if (!props.community?.id) return;
  try {
    const history = await getCommunityRatingHistory(props.community.id, 30);
    renderChart(history.labels, history.values);
  } catch (e) {
    console.error("Error loading community rating history:", e);
  }
}

function renderChart(labels: string[], data: number[]) {
  if (!canvas.value) return;
  if (chart) chart.destroy();
  chart = new Chart(canvas.value, {
    type: "line",
    data: {
      labels,
      datasets: [
        {
          label: "Рейтинг",
          data,
          borderColor: "#3498db",
          backgroundColor: "rgba(52,152,219,0.1)",
          tension: 0.2,
          pointRadius: 2,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      scales: { y: { beginAtZero: false } },
      plugins: { legend: { display: false } },
    },
  });
}

onMounted(loadHistory);
watch(() => props.community, loadHistory, { deep: true });
onUnmounted(() => {
  if (chart) chart.destroy();
});
</script>
