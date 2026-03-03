<template>
  <div class="container mx-auto px-4 py-6 max-w-4xl">
    <h1 class="text-3xl font-bold mb-6 text-gray-900 dark:text-white">
      Чаты сообществ
    </h1>

    <div
      v-if="loading"
      class="text-center py-10 text-gray-500 dark:text-gray-400"
    >
      Загрузка...
    </div>

    <div
      v-else-if="communities.length === 0"
      class="text-center py-10 text-gray-500 dark:text-gray-400"
    >
      Вы не состоите ни в одном сообществе.
    </div>

    <div v-else class="space-y-4">
      <div
        v-for="community in communities"
        :key="community.id"
        class="border border-primary/10 dark:border-gray-700 rounded-lg bg-white dark:bg-gray-800 overflow-hidden"
      >
        <!-- Шапка сообщества (кликабельная) -->
        <div
          class="flex items-center gap-3 p-4 cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-700 transition"
          @click="toggleCommunity(community.id)"
        >
          <img
            :src="getAvatarUrl(community.avatar, 40)"
            class="w-10 h-10 rounded-full object-cover"
            alt=""
          />
          <div class="flex-1 min-w-0">
            <h2
              class="font-semibold text-gray-900 dark:text-white truncate flex items-center gap-1"
            >
              {{ community.name }}
              <CheckBadgeIcon
                v-if="community.patent"
                class="w-4 h-4 text-accent shrink-0"
              />
            </h2>
            <p class="text-sm text-gray-500 dark:text-gray-400">
              {{ community.rating }} ⭐ •
              {{ community.membersCount }} участников
            </p>
          </div>
          <ChevronUpIcon
            v-if="expanded[community.id]"
            class="w-5 h-5 text-gray-500 dark:text-gray-400"
          />
          <ChevronDownIcon
            v-else
            class="w-5 h-5 text-gray-500 dark:text-gray-400"
          />
        </div>

        <!-- Развёрнутый чат -->
        <div
          v-if="expanded[community.id]"
          class="border-t border-primary/10 dark:border-gray-700 p-4"
        >
          <div
            class="h-64 overflow-y-auto mb-3 p-2 bg-gray-50 dark:bg-gray-900 rounded"
            :data-chat-id="community.id"
          >
            <div
              v-if="loadingMessages[community.id]"
              class="text-center text-gray-500"
            >
              Загрузка сообщений...
            </div>
            <div
              v-else-if="!messages[community.id]?.length"
              class="text-center text-gray-500"
            >
              Пока нет сообщений
            </div>
            <div v-else>
              <div
                v-for="msg in messages[community.id]"
                :key="msg.id"
                class="mb-2"
              >
                <span
                  class="font-semibold text-sm text-gray-800 dark:text-white"
                  >{{ msg.user?.username }}:</span
                >
                <span class="text-sm text-gray-700 dark:text-gray-300">{{
                  msg.text
                }}</span>
                <span class="text-xs text-gray-400 dark:text-gray-500 ml-2">{{
                  formatTime(msg.created_at)
                }}</span>
              </div>
            </div>
          </div>
          <div class="flex gap-2">
            <input
              v-model="newMessage[community.id]"
              type="text"
              class="flex-1 px-3 py-2 bg-white dark:bg-gray-800 border border-primary/20 dark:border-gray-700 rounded-lg focus:border-accent focus:ring-1 focus:ring-accent outline-none text-sm"
              placeholder="Сообщение..."
              @keyup.enter="sendMessage(community.id)"
            />
            <button
              @click="sendMessage(community.id)"
              class="px-4 py-2 bg-accent hover:bg-accent-dark dark:bg-accent-600 dark:hover:bg-accent-700 text-white rounded-lg text-sm"
            >
              Отправить
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick } from "vue";
import {
  CheckBadgeIcon,
  ChevronUpIcon,
  ChevronDownIcon,
} from "@heroicons/vue/24/outline";
import { useAuth } from "~/composables/useAuth";
import { useCommunity } from "~/composables/useCommunity";
import { getAvatarUrl } from "~/utils/image";

const { userId } = useAuth();
const {
  getUserCommunities,
  getMessages,
  sendMessage: sendMsg,
  getMembers,
} = useCommunity();

const loading = ref(true);
const communities = ref<any[]>([]);
const expanded = ref<Record<number, boolean>>({});
const messages = ref<Record<number, any[]>>({});
const loadingMessages = ref<Record<number, boolean>>({});
const newMessage = ref<Record<number, string>>({});

async function loadCommunities() {
  if (!userId.value) return;
  try {
    const comms = await getUserCommunities(userId.value);
    communities.value = comms;
    // Загружаем количество участников для каждого
    await Promise.all(
      comms.map(async (c) => {
        const members = await getMembers(c.id);
        (c as any).membersCount = members.length;
      }),
    );
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
}

async function toggleCommunity(id: number) {
  expanded.value[id] = !expanded.value[id];
  if (expanded.value[id] && !messages.value[id]) {
    await loadMessages(id);
  }
}

async function loadMessages(id: number) {
  loadingMessages.value[id] = true;
  try {
    const msgs = await getMessages(id, 20);
    messages.value[id] = msgs;
  } catch (e) {
    console.error(e);
  } finally {
    loadingMessages.value[id] = false;
  }
}

async function sendMessage(id: number) {
  const text = newMessage.value[id]?.trim();
  if (!text) return;
  try {
    const msg = await sendMsg(id, text);
    if (!messages.value[id]) messages.value[id] = [];
    messages.value[id].push(msg);
    newMessage.value[id] = "";
    await nextTick();
    const container = document.querySelector(`[data-chat-id="${id}"]`);
    if (container) container.scrollTop = container.scrollHeight;
  } catch (e) {
    console.error(e);
  }
}

function formatTime(date: string) {
  return new Date(date).toLocaleTimeString("ru-RU", {
    hour: "2-digit",
    minute: "2-digit",
  });
}

onMounted(loadCommunities);
</script>
