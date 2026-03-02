<template>
  <div class="container mx-auto px-4 py-6 max-w-6xl">
    <h1 class="text-3xl font-bold mb-6 text-gray-800">
      Запросы на смену названия
    </h1>

    <!-- Глобальный поиск по всем сообществам (для проверки уникальности) -->
    <div class="mb-6 max-w-md">
      <div class="relative">
        <input
          v-model="globalSearchQuery"
          type="text"
          placeholder="Поиск по всем сообществам..."
          class="w-full px-4 py-2 pl-10 bg-white border border-primary/20 rounded-lg focus:border-accent focus:ring-1 focus:ring-accent outline-none transition"
          @input="debouncedGlobalSearch"
        />
        <MagnifyingGlassIcon
          class="absolute left-3 top-1/2 transform -translate-y-1/2 w-5 h-5 text-primary/50"
        />
      </div>
      <!-- Результаты глобального поиска -->
      <div
        v-if="globalSearchResults.length"
        class="mt-2 bg-white border border-primary/20 rounded-lg p-2 max-h-60 overflow-y-auto shadow-lg"
      >
        <NuxtLink
          v-for="c in globalSearchResults"
          :key="c.id"
          :to="`/communities/${c.id}`"
          class="flex items-center gap-2 p-2 hover:bg-gray-50 rounded text-gray-700"
        >
          <img
            :src="c.avatar || defaultAvatar"
            class="w-6 h-6 rounded-full object-cover"
          />
          <span>{{ c.name }}</span>
          <CheckBadgeIcon v-if="c.patent" class="w-4 h-4 text-accent ml-auto" />
        </NuxtLink>
      </div>
    </div>

    <!-- Локальный поиск по заявкам -->
    <div class="mb-4 max-w-md">
      <div class="relative">
        <input
          v-model="localSearchQuery"
          type="text"
          placeholder="Фильтр по названию сообщества в заявках..."
          class="w-full px-4 py-2 pl-10 bg-white border border-primary/20 rounded-lg focus:border-accent focus:ring-1 focus:ring-accent outline-none transition"
        />
        <MagnifyingGlassIcon
          class="absolute left-3 top-1/2 transform -translate-y-1/2 w-5 h-5 text-primary/50"
        />
      </div>
    </div>

    <!-- Список заявок -->
    <div v-if="loading" class="text-center py-10 text-gray-500">
      Загрузка...
    </div>
    <div
      v-else-if="filteredRequests.length === 0"
      class="text-center py-10 text-gray-500"
    >
      Нет подходящих заявок.
    </div>
    <div v-else class="space-y-6">
      <div
        v-for="req in filteredRequests"
        :key="req.id"
        class="bg-white border border-primary/10 rounded-lg p-6 shadow-sm"
      >
        <div class="flex flex-col lg:flex-row gap-6">
          <!-- Карточка сообщества -->
          <div class="lg:w-80 flex-shrink-0">
            <CommunityCard :community="req.community" />
          </div>

          <!-- Детали запроса -->
          <div class="flex-1">
            <p class="text-sm text-gray-500">
              Запрошено: {{ formatDate(req.created_at) }}
            </p>
            <div class="mt-2">
              <p class="text-sm text-gray-600">
                Текущее название:
                <span class="font-medium">{{ req.community.name }}</span>
              </p>
              <p class="text-sm text-gray-600">
                Запрашиваемое:
                <span class="font-medium text-accent">{{
                  req.requested_name
                }}</span>
              </p>
            </div>
            <p class="text-sm text-gray-500 mt-2">
              Владелец:
              <NuxtLink
                :to="`/profile/${req.community.owner?.auth_uid}`"
                class="text-accent hover:underline"
              >
                {{ req.community.owner?.use || req.community.owner?.username }}
              </NuxtLink>
            </p>

            <!-- Поле для комментария при отклонении (если активно) -->
            <div v-if="rejectingId === req.id" class="mt-3">
              <textarea
                v-model="rejectComment"
                rows="2"
                class="w-full px-3 py-2 border border-primary/20 rounded-lg focus:border-accent focus:ring-1 focus:ring-accent outline-none"
                placeholder="Причина отклонения (необязательно)"
              ></textarea>
            </div>

            <div class="flex gap-3 mt-4">
              <button
                @click="approve(req.id)"
                class="px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 transition flex items-center gap-2"
              >
                <CheckIcon class="w-5 h-5" />
                Одобрить
              </button>
              <button
                v-if="rejectingId !== req.id"
                @click="rejectingId = req.id"
                class="px-4 py-2 border border-red-600 text-red-600 rounded-lg hover:bg-red-50 transition flex items-center gap-2"
              >
                <XMarkIcon class="w-5 h-5" />
                Отклонить
              </button>
              <template v-else>
                <button
                  @click="confirmReject(req.id)"
                  class="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 transition"
                >
                  Подтвердить
                </button>
                <button
                  @click="
                    rejectingId = null;
                    rejectComment = '';
                  "
                  class="px-4 py-2 border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50 transition"
                >
                  Отмена
                </button>
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import {
  MagnifyingGlassIcon,
  CheckIcon,
  XMarkIcon,
  CheckBadgeIcon,
} from "@heroicons/vue/24/outline";
import CommunityCard from "~/components/CommunityCard.vue";
import { useCommunity } from "~/composables/useCommunity";

const {
  getPendingNameChangeRequests,
  approveNameChangeRequest,
  rejectNameChangeRequest,
} = useCommunity();
const supabase = useSupabaseClient();
const defaultAvatar =
  "https://phlyzwfqtpddvgrprngo.supabase.co/storage/v1/object/public/avatars/default.jpg";

// Заявки
const requests = ref<any[]>([]);
const loading = ref(true);

// Локальный поиск по заявкам
const localSearchQuery = ref("");
const filteredRequests = computed(() => {
  const q = localSearchQuery.value.toLowerCase();
  if (!q) return requests.value;
  return requests.value.filter(
    (req) =>
      req.community.name?.toLowerCase().includes(q) ||
      req.requested_name?.toLowerCase().includes(q),
  );
});

// Глобальный поиск по всем сообществам
const globalSearchQuery = ref("");
const globalSearchResults = ref<any[]>([]);

const performGlobalSearch = async (query: string) => {
  if (!query) {
    globalSearchResults.value = [];
    return;
  }
  const { data, error } = await supabase
    .from("community")
    .select("id, name, avatar, patent")
    .ilike("name", `%${query}%`)
    .limit(10);
  if (error) {
    console.error("Global search error:", error);
    return;
  }
  globalSearchResults.value = data || [];
};

const debouncedGlobalSearch = debounce(() => {
  performGlobalSearch(globalSearchQuery.value);
}, 300);

// Состояние для отклонения
const rejectingId = ref<number | null>(null);
const rejectComment = ref("");

// Форматирование даты
function formatDate(date: string) {
  return new Date(date).toLocaleDateString("ru-RU", {
    day: "numeric",
    month: "long",
    year: "numeric",
  });
}

// Загрузка заявок
async function loadRequests() {
  try {
    const data = await getPendingNameChangeRequests();
    requests.value = data;
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
}

// Действия
async function approve(id: number) {
  await approveNameChangeRequest(id);
  await loadRequests();
}

async function confirmReject(id: number) {
  await rejectNameChangeRequest(id, rejectComment.value || undefined);
  rejectingId.value = null;
  rejectComment.value = "";
  await loadRequests();
}

// Debounce helper
function debounce<T extends (...args: any[]) => any>(fn: T, delay: number) {
  let timer: ReturnType<typeof setTimeout>;
  return (...args: Parameters<T>) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

onMounted(loadRequests);
</script>
