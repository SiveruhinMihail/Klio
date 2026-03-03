<template>
  <div class="h-48 sm:h-64 w-full">
    <ClientOnly>
      <canvas ref="canvas"></canvas>
      <template #fallback>
        <!-- Скелетон для графика -->
        <div
          class="h-full w-full bg-gray-100 dark:bg-gray-800 rounded animate-pulse"
        ></div>
      </template>
    </ClientOnly>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from "vue";
import { useCommunity } from "~/composables/useCommunity";

const props = defineProps({ community: { type: Object, required: true } });

const { getCommunityRatingHistory } = useCommunity();
const canvas = ref<HTMLCanvasElement | null>(null);
let chart: any = null;

async function loadHistory() {
  if (!props.community?.id) return;
  try {
    const history = await getCommunityRatingHistory(props.community.id, 30);
    await renderChart(history.labels, history.values);
  } catch (e) {
    console.error("Error loading community rating history:", e);
  }
}

async function renderChart(labels: string[], data: number[]) {
  if (!canvas.value) return;

  // Динамический импорт Chart.js только на клиенте
  const { default: Chart } = await import("chart.js/auto");

  if (chart) chart.destroy();

  const isDark = document.documentElement.classList.contains("dark");

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
          pointBackgroundColor: isDark ? "#60a5fa" : "#3498db",
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      scales: {
        y: {
          beginAtZero: false,
          grid: {
            color: isDark ? "#374151" : "#e5e7eb",
          },
          ticks: {
            color: isDark ? "#9ca3af" : "#6b7280",
          },
        },
        x: {
          ticks: {
            color: isDark ? "#9ca3af" : "#6b7280",
            maxTicksLimit: 6,
          },
          grid: {
            display: false,
          },
        },
      },
      plugins: {
        legend: { display: false },
      },
    },
  });
}

onMounted(loadHistory);

watch(() => props.community, loadHistory, { deep: true });

// Перерисовка при смене темы
const colorMode = useColorMode();
watch(
  () => colorMode.value,
  () => {
    if (canvas.value) loadHistory();
  },
);

onUnmounted(() => {
  if (chart) chart.destroy();
});
</script>
