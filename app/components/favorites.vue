<script setup>
import { useUser } from "~/composables/useUser";
import { useAuth } from "~/composables/useAuth";

const { getUserFavorites } = useUser();
const { profile } = useAuth();
const loading = ref(true);
const posts = ref([]);

onMounted(async () => {
  if (!profile.value) {
    // Редирект на логин
    return navigateTo("/auth/login");
  }
  try {
    posts.value = await getUserFavorites(profile.value.id);
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
});

async function handleFavorite(post) {
  try {
    const isNowFavorited = await toggleFavorite(post.id);
    if (!isNowFavorited) {
      // Удаляем из списка
      posts.value = posts.value.filter((p) => p.id !== post.id);
    }
  } catch (e) {
    console.error(e);
  }
}
</script>
<template>
  <div class="container mx-auto px-4 py-6">
    <h1 class="text-3xl font-bold mb-6 text-gray-900 dark:text-white">
      Мои закладки
    </h1>
    <div v-if="loading" class="text-center py-10">Загрузка...</div>
    <div
      v-else-if="favorites.length === 0"
      class="text-center py-10 text-gray-500 dark:text-gray-400"
    >
      У вас пока нет избранных постов.
    </div>
    <div
      v-else
      class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4"
    >
      <PostCard
        v-for="post in favorites"
        :key="post.id"
        :post="post"
        @like="handleLike"
        @favorite="handleFavorite"
      />
    </div>
  </div>
</template>
