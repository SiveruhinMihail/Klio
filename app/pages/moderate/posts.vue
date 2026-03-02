<template>
  <div class="container mx-auto px-4 py-6 max-w-6xl">
    <div class="flex justify-between items-center mb-6">
      <h1 class="text-3xl font-bold text-gray-800">Модерация постов</h1>
      <NuxtLink
        to="/moderate"
        class="text-primary hover:text-accent flex items-center gap-1"
      >
        <ArrowLeftIcon class="w-5 h-5" />
        Назад в панель
      </NuxtLink>
    </div>

    <div v-if="loading" class="text-center py-10 text-gray-500">
      Загрузка...
    </div>

    <div
      v-else-if="pendingPosts.length === 0"
      class="text-center py-10 text-gray-500"
    >
      Нет постов на модерацию
    </div>

    <div v-else class="space-y-4">
      <div
        v-for="post in pendingPosts"
        :key="post.id"
        class="bg-white border border-primary/10 rounded-lg p-4 shadow-sm hover:shadow-md transition"
      >
        <div class="flex flex-col md:flex-row justify-between gap-4">
          <div class="flex-1">
            <!-- Заголовок поста (ссылка) -->
            <NuxtLink
              :to="`/post/${post.id}`"
              class="text-xl font-semibold text-gray-800 hover:text-accent mb-2 block"
            >
              {{ post.title }}
            </NuxtLink>

            <!-- Краткое описание -->
            <p class="text-gray-600 mb-3 line-clamp-3">
              {{ post.description }}
            </p>

            <!-- Информация об авторе (аватар + ссылка + кнопка жалобы) -->
            <div class="flex items-center gap-3 text-sm text-gray-500">
              <span class="font-medium">Автор:</span>
              <NuxtLink
                v-if="post.author"
                :to="`/profile/${post.author.auth_uid}`"
                class="flex items-center gap-1 hover:text-accent"
              >
                <img
                  :src="post.author.avatar || defaultAvatar"
                  class="w-5 h-5 rounded-full object-cover"
                  alt=""
                  @error="handleImageError"
                />
                <span>{{ post.author.use || post.author.username }}</span>
              </NuxtLink>
              <span v-else class="text-gray-400">(удалён)</span>

              <!-- Кнопка жалобы на автора (для модераторов) -->
              <button
                v-if="post.author"
                @click="openReportModal(post.author.id)"
                class="ml-2 p-1 text-gray-400 hover:text-primary rounded-full hover:bg-gray-100 transition"
                title="Пожаловаться на автора"
              >
                <FlagIcon class="w-4 h-4" />
              </button>

              <span class="text-gray-300 mx-1">•</span>
              <span>{{ formatDate(post.created_at) }}</span>
            </div>
          </div>

          <!-- Кнопки действий (одобрить/отклонить) -->
          <div class="flex gap-2 items-start">
            <button
              @click="approve(post.id)"
              class="px-4 py-2 bg-accent text-white rounded-lg hover:bg-accent-dark transition shadow-sm"
            >
              Одобрить
            </button>
            <button
              @click="reject(post.id)"
              class="px-4 py-2 border border-red-600 text-red-600 rounded-lg hover:bg-red-50 transition"
            >
              Отклонить
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Модалка жалобы (используем существующий ReportModal) -->
    <ReportModal
      v-if="reportTarget"
      :is-open="true"
      :target-type="reportTarget.type"
      :target-id="reportTarget.id"
      @close="reportTarget = null"
      @submitted="handleReportSubmitted"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
import { FlagIcon, ArrowLeftIcon } from "@heroicons/vue/24/outline";
import { useAuth } from "~/composables/useAuth";
import ReportModal from "~/components/ReportModal.vue";

const supabase = useSupabaseClient();
const { isModerator, isAdmin } = useAuth();

const defaultAvatar =
  "https://phlyzwfqtpddvgrprngo.supabase.co/storage/v1/object/public/avatars/default.jpg";

const pendingPosts = ref<any[]>([]);
const loading = ref(true);

// Для жалобы
const reportTarget = ref<{ type: "user"; id: number } | null>(null);

function openReportModal(userId: number) {
  reportTarget.value = { type: "user", id: userId };
}

function handleReportSubmitted() {
  reportTarget.value = null;
  // можно показать уведомление, но пока просто закрываем
}

function handleImageError(e: Event) {
  (e.target as HTMLImageElement).src = defaultAvatar;
}

async function loadPosts() {
  if (!isModerator.value && !isAdmin.value) return navigateTo("/");
  loading.value = true;
  try {
    const { data, error } = await supabase
      .from("post")
      .select(
        `
        id,
        title,
        description,
        created_at,
        moderation_status,
        author:author_id (
          id,
          use,
          username,
          avatar,
          auth_uid
        )
      `,
      )
      .eq("moderation_status", "pending")
      .order("created_at", { ascending: false });
    if (error) throw error;
    pendingPosts.value = data || [];
  } catch (e) {
    console.error("Error loading pending posts:", e);
  } finally {
    loading.value = false;
  }
}

const approve = async (id: number) => {
  try {
    await $fetch("/api/moderation/approve-post", {
      method: "POST",
      body: { postId: id },
    });
    pendingPosts.value = pendingPosts.value.filter((p) => p.id !== id);
  } catch (e) {
    console.error(e);
    alert("Ошибка при одобрении");
  }
};

const reject = async (id: number) => {
  try {
    await $fetch("/api/moderation/reject-post", {
      method: "POST",
      body: { postId: id },
    });
    pendingPosts.value = pendingPosts.value.filter((p) => p.id !== id);
  } catch (e) {
    console.error(e);
    alert("Ошибка при отклонении");
  }
};

function formatDate(date: string) {
  return new Date(date).toLocaleDateString("ru-RU", {
    day: "numeric",
    month: "short",
    year: "numeric",
  });
}

onMounted(loadPosts);
</script>
