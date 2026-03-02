<template>
  <div>
    <!-- Переключатель статусов -->
    <div class="flex gap-2 mb-4">
      <button
        v-for="status in statuses"
        :key="status.value"
        @click="selectedStatus = status.value"
        :class="[
          'px-3 py-1 rounded',
          selectedStatus === status.value
            ? 'bg-accent text-white'
            : 'bg-gray-200 hover:bg-gray-300',
        ]"
      >
        {{ status.label }}
      </button>
    </div>

    <div v-if="loading" class="text-center py-4">Загрузка...</div>
    <div v-else-if="posts.length === 0" class="text-center py-4 text-gray-500">
      Нет постов
    </div>
    <div v-else class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      <PostCard
        v-for="post in posts"
        :key="post.id"
        :post="post"
        @like="handleLike"
        @favorite="handleFavorite"
      >
        <!-- Слот для дополнительных действий -->
        <template
          #actions
          v-if="isOwner && post.moderation_status === 'rejected'"
        >
          <NuxtLink
            :to="`/edit-post/${post.id}`"
            class="text-accent hover:underline text-sm"
          >
            Редактировать
          </NuxtLink>
        </template>
      </PostCard>
    </div>

    <!-- Пагинация -->
    <div v-if="hasMore" class="text-center mt-4">
      <button @click="loadMore" class="text-accent hover:underline">
        Загрузить ещё
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted } from "vue";
import PostCard from "./PostCard.vue";

const props = defineProps<{
  userId: number | undefined; // разрешаем undefined
  isOwner: boolean;
}>();

const supabase = useSupabaseClient();
const { toggleFavorite } = useFavorites();

const statuses = [
  { value: "approved", label: "Опубликованные" },
  { value: "pending", label: "На проверке" },
  { value: "rejected", label: "Отклонённые" },
];

const selectedStatus = ref("approved");
const posts = ref<any[]>([]);
const loading = ref(false);
const page = ref(0);
const limit = 10;
const hasMore = ref(true);

// Загружаем посты только если userId определён
async function loadPosts(reset = false) {
  if (!props.userId) return; // защита
  if (reset) {
    page.value = 0;
    posts.value = [];
    hasMore.value = true;
  }
  if (!hasMore.value) return;
  loading.value = true;
  try {
    const from = page.value * limit;
    const to = from + limit - 1;
    const { data, error } = await supabase
      .from("post")
      .select(
        `
        *,
        post_images (*),
        likes:like_to_post(count),
        categories:post_categories(category:category_id(id, name, slug))
      `,
      )
      .eq("author_id", props.userId)
      .eq("moderation_status", selectedStatus.value)
      .order("created_at", { ascending: false })
      .range(from, to);
    if (error) throw error;
    if (data) {
      const enriched = await enrichPosts(data);
      posts.value.push(...enriched);
      hasMore.value = data.length === limit;
      page.value++;
    }
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
}

// Обогащение постов (лайки, избранное)
async function enrichPosts(postsData: any[]) {
  if (!props.isOwner) return postsData; // для чужих не нужно обогащать
  const postIds = postsData.map((p) => p.id);
  let likedSet = new Set<number>();
  let favoritedSet = new Set<number>();

  if (props.isOwner) {
    const { data: likes } = await supabase
      .from("like_to_post")
      .select("post_id")
      .eq("user_id", props.userId as number)
      .in("post_id", postIds);
    likedSet = new Set(likes?.map((l) => l.post_id) || []);

    const { data: favorites } = await supabase
      .from("favorites")
      .select("post_id")
      .eq("user_id", props.userId as number)
      .in("post_id", postIds);
    favoritedSet = new Set(favorites?.map((f) => f.post_id) || []);
  }

  return postsData.map((post) => ({
    ...post,
    rating: post.likes?.[0]?.count || 0,
    isLiked: likedSet.has(post.id),
    isFavorited: favoritedSet.has(post.id),
  }));
}

// Следим за изменением статуса и перезагружаем
watch(selectedStatus, () => {
  if (props.userId) loadPosts(true);
});

// Загружаем при монтировании, если userId есть
onMounted(() => {
  if (props.userId) loadPosts(true);
});

// Следим за изменением userId (на случай, если он появится позже)
watch(
  () => props.userId,
  (newId) => {
    if (newId) loadPosts(true);
  },
  { immediate: true },
);

function loadMore() {
  if (props.userId) loadPosts();
}

// Обработка лайка и избранного (аналогично другим компонентам)
async function handleLike(post: any) {
  /* ... */
}
async function handleFavorite(post: any) {
  /* ... */
}
</script>
