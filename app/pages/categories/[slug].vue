<template>
  <div class="container mx-auto px-4 py-6">
    <h1 class="text-3xl font-bold mb-6 text-gray-800">{{ category?.name }}</h1>
    <div v-if="loading" class="text-center">Загрузка...</div>
    <div v-else-if="posts.length === 0" class="text-center text-gray-500">
      Нет постов в этой категории
    </div>
    <div v-else>
      <div
        class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4"
      >
        <PostCard
          v-for="post in posts"
          :key="post.id"
          :post="post"
          @like="() => handleLike(post)"
          @favorite="() => handleFavorite(post)"
        />
      </div>
      <div v-if="hasMore" class="text-center mt-4">
        <button
          @click="loadMore"
          class="px-6 py-2 bg-secondary text-white rounded-lg hover:bg-secondary/80 transition shadow-sm"
        >
          Загрузить ещё
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { Database } from "~/types/supabase";
import { useAuth } from "~/composables/useAuth";
import { useFavorites } from "~/composables/useFavorites";
import PostCard from "~/components/PostCard.vue";

const route = useRoute();
const supabase = useSupabaseClient<Database>();
const { getPostsByCategory, getCategoryBySlug } = usePosts();
const { isAuthenticated, userId } = useAuth();
const { toggleFavorite } = useFavorites();

const category = ref<any>(null);
const posts = ref<any[]>([]);
const loading = ref(true);
const page = ref(0);
const hasMore = ref(true);

async function enrichPostsWithUserData(posts: any[]) {
  if (!isAuthenticated.value || !userId.value || !posts.length) return posts;
  const postIds = posts.map((p) => p.id);
  const [likesResult, favoritesResult] = await Promise.all([
    supabase
      .from("like_to_post")
      .select("post_id")
      .eq("user_id", userId.value)
      .in("post_id", postIds),
    supabase
      .from("favorites")
      .select("post_id")
      .eq("user_id", userId.value)
      .in("post_id", postIds),
  ]);
  const likedSet = new Set(likesResult.data?.map((l) => l.post_id) || []);
  const favoritedSet = new Set(
    favoritesResult.data?.map((f) => f.post_id) || [],
  );
  return posts.map((post) => ({
    ...post,
    isLiked: likedSet.has(post.id),
    isFavorited: favoritedSet.has(post.id),
    rating: post.rating || 0,
  }));
}

async function loadCategory() {
  try {
    category.value = await getCategoryBySlug(route.params.slug as string);
    if (!category.value) return navigateTo("/404");
    await resetAndLoad();
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
}

async function resetAndLoad() {
  page.value = 0;
  posts.value = [];
  hasMore.value = true;
  await loadPosts(0);
}

async function loadPosts(offset: number) {
  const newPosts = await getPostsByCategory(category.value.id, 20, offset);
  const enriched = await enrichPostsWithUserData(newPosts);
  if (enriched.length < 20) hasMore.value = false;
  posts.value = offset === 0 ? enriched : [...posts.value, ...enriched];
}

async function loadMore() {
  page.value++;
  await loadPosts(page.value * 20);
}

async function handleLike(post: any) {
  // оптимистичное обновление как на главной
  if (!isAuthenticated.value || !userId.value) return;
  const wasLiked = post.isLiked;
  post.isLiked = !wasLiked;
  if (post.likes?.[0]) post.likes[0].count += wasLiked ? -1 : 1;
  // обновляем в массиве
  const index = posts.value.findIndex((p) => p.id === post.id);
  if (index !== -1) posts.value[index] = { ...post };
  try {
    if (wasLiked) {
      await supabase
        .from("like_to_post")
        .delete()
        .eq("post_id", post.id)
        .eq("user_id", userId.value);
    } else {
      await supabase
        .from("like_to_post")
        .insert({ post_id: post.id, user_id: userId.value });
    }
  } catch (e) {
    post.isLiked = wasLiked;
    if (post.likes?.[0]) post.likes[0].count += wasLiked ? 1 : -1;
    if (index !== -1) posts.value[index] = { ...post };
    console.error(e);
  }
}

async function handleFavorite(post: any) {
  if (!isAuthenticated.value || !userId.value) return;
  const uid = userId.value; // теперь TypeScript знает, что это number
  const wasFavorited = post.isFavorited;
  post.isFavorited = !wasFavorited;
  const index = posts.value.findIndex((p) => p.id === post.id);
  if (index !== -1) posts.value[index] = { ...post };
  try {
    if (wasFavorited) {
      await supabase
        .from("favorites")
        .delete()
        .eq("post_id", post.id)
        .eq("user_id", uid);
    } else {
      await supabase
        .from("favorites")
        .insert({ post_id: post.id, user_id: uid });
    }
  } catch (e) {
    post.isFavorited = wasFavorited;
    if (index !== -1) posts.value[index] = { ...post };
    console.error(e);
  }
}

onMounted(loadCategory);
</script>
