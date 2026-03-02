<template>
  <div class="container mx-auto px-4 py-6 max-w-4xl">
    <div v-if="loading" class="text-center py-10">Загрузка...</div>
    <div v-else-if="!post" class="text-center py-10 text-gray-500">
      Пост не найден
    </div>
    <div v-else>
      <!-- Шапка поста -->
      <div class="mb-6">
        <h1 class="text-3xl font-bold text-gray-800 mb-2">{{ post.title }}</h1>

        <!-- Мета-информация: автор, дата, действия -->
        <div class="flex flex-wrap items-center justify-between gap-4 text-sm">
          <div class="flex items-center gap-3 text-gray-600">
            <NuxtLink
              :to="`/profile/${post.author?.auth_uid}`"
              class="flex items-center gap-2 hover:text-accent transition"
            >
              <img
                :src="post.author?.avatar || '/default-avatar.png'"
                class="w-6 h-6 rounded-full object-cover"
                alt=""
              />
              <span class="font-medium">{{ post.author?.use || "Автор" }}</span>
            </NuxtLink>
            <span>•</span>
            <span>{{ formatDate(post.created_at) }}</span>
          </div>

          <!-- Кнопки действий (только если пост одобрен) -->
          <div v-if="isPostActive" class="flex items-center gap-4">
            <button
              @click="toggleLike"
              :disabled="!isAuthenticated"
              class="flex items-center gap-1 text-gray-600 hover:text-red-500 transition"
            >
              <HeartIcon
                :class="[isLiked ? 'text-red-500 fill-red-500' : '']"
                class="w-5 h-5"
              />
              <span>{{ post.rating }}</span>
            </button>
            <button
              @click="toggleFavoritePost"
              :disabled="!isAuthenticated"
              class="flex items-center gap-1 text-gray-600 hover:text-yellow-500 transition"
            >
              <BookmarkIcon
                :class="[isFavorited ? 'text-yellow-500 fill-yellow-500' : '']"
                class="w-5 h-5"
              />
            </button>
            <button
              @click="openReportModal('post', post.id)"
              :disabled="!isAuthenticated || hasReportedPost"
              class="flex items-center gap-1 text-gray-600 hover:text-primary disabled:opacity-50"
              :title="
                hasReportedPost ? 'Вы уже отправили жалобу' : 'Пожаловаться'
              "
            >
              <FlagIcon
                :class="[hasReportedPost ? 'text-primary fill-primary' : '']"
                class="w-5 h-5"
              />
            </button>
          </div>

          <!-- Баннер для неодобренных постов -->
          <div v-else class="text-sm text-orange-600">
            {{
              post.moderation_status === "pending"
                ? "Пост на проверке. Взаимодействие недоступно."
                : "Пост отклонён модератором."
            }}
          </div>
        </div>

        <!-- Категории -->
        <div v-if="postCategories.length" class="flex flex-wrap gap-2 mt-4">
          <NuxtLink
            v-for="cat in postCategories"
            :key="cat.id"
            :to="`/categories/${cat.slug}`"
            class="px-3 py-1 bg-accent/20 text-accent-dark rounded-full text-sm hover:bg-accent/30 transition"
          >
            {{ cat.name }}
          </NuxtLink>
        </div>
      </div>

      <!-- Галерея изображений -->
      <div v-if="post.post_images?.length" class="mb-8">
        <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-2">
          <div
            v-for="(img, index) in post.post_images"
            :key="index"
            class="aspect-square bg-primary/5 rounded-lg overflow-hidden cursor-pointer hover:opacity-90 transition"
            @click="openImageViewer(postImages, index as number)"
          >
            <img :src="img.url" class="w-full h-full object-cover" alt="" />
          </div>
        </div>
      </div>

      <!-- Описание поста -->
      <div
        class="prose prose-lg max-w-none mb-8"
        v-html="renderedDescription"
      ></div>

      <!-- Комментарии -->
      <div class="border-t border-primary/10 pt-6" id="comments">
        <h2 class="text-2xl font-semibold mb-4 flex items-center gap-2">
          <ChatBubbleLeftIcon class="w-6 h-6 text-primary" />
          Комментарии ({{ totalComments }})
        </h2>

        <!-- Форма комментария (только если пост одобрен) -->
        <CommentForm
          v-if="isAuthenticated && isPostActive"
          :post-id="post.id"
          :parent-id="replyToCommentId"
          :reply-to="replyToUsername"
          @comment-added="handleNewComment"
          @cancel="cancelReply"
          class="mb-6"
        />
        <div
          v-else-if="!isAuthenticated && isPostActive"
          class="mb-6 text-center text-gray-500"
        >
          <NuxtLink to="/auth/login" class="text-accent hover:underline"
            >Войдите</NuxtLink
          >, чтобы оставить комментарий.
        </div>

        <!-- Список комментариев (отображаются всегда, но без интерактива, если пост не одобрен) -->
        <div
          v-if="commentTree.length === 0"
          class="text-gray-500 text-center py-4"
        >
          Пока нет комментариев. Будьте первым!
        </div>
        <div v-else class="space-y-4">
          <CommentItem
            v-for="comment in commentTree"
            :key="comment.id"
            :comment="comment"
            :is-authenticated="isAuthenticated"
            :interactive="isPostActive"
            @like="toggleCommentLike"
            @reply="startReply"
            @report="reportComment"
            @open-image="openImageViewer"
          />
        </div>

        <!-- Кнопка загрузить ещё комментарии -->
        <div v-if="hasMoreComments" class="text-center mt-4">
          <button @click="loadMoreComments" class="text-accent hover:underline">
            Загрузить ещё комментарии
          </button>
        </div>
      </div>
    </div>

    <!-- Модальные окна -->
    <ReportModal
      v-if="reportModalTarget"
      :is-open="true"
      :target-type="reportModalTarget.type"
      :target-id="reportModalTarget.id"
      @close="reportModalTarget = null"
      @submitted="handleReportSubmitted"
    />

    <ImageViewer
      v-if="showViewer"
      :images="viewerImages"
      :initial-index="viewerIndex"
      @close="showViewer = false"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, nextTick } from "vue";
import {
  HeartIcon,
  BookmarkIcon,
  FlagIcon,
  ChatBubbleLeftIcon,
} from "@heroicons/vue/24/outline";
import type { Database } from "~/types/supabase";
import MarkdownIt from "markdown-it";
import CommentForm from "~/components/CommentForm.vue";
import CommentItem from "~/components/CommentItem.vue";
import ImageViewer from "~/components/ImageViewer.vue";
import ReportModal from "~/components/ReportModal.vue";

const supabase = useSupabaseClient<Database>();
const route = useRoute();
const { isAuthenticated, userId } = useAuth();
const { toggleFavorite, isFavorite } = useFavorites();

const md = new MarkdownIt();

const loading = ref(true);
const post = ref<any>(null);
const comments = ref<any[]>([]);
const totalComments = ref(0);
const isLiked = ref(false);
const isFavorited = ref(false);
const replyToCommentId = ref<number | null>(null);
const replyToUsername = ref<string | null>(null);
const hasReportedPost = ref(false);
const postCategories = ref<any[]>([]);

const reportModalTarget = ref<{
  type: "post" | "comment" | "user";
  id: number;
} | null>(null);
const viewerImages = ref<string[]>([]);
const viewerIndex = ref(0);
const showViewer = ref(false);

const postImages = computed(
  () => post.value?.post_images?.map((img: any) => img.url) || [],
);
const renderedDescription = computed(() =>
  md.render(post.value?.description || ""),
);

const isPostActive = computed(
  () => post.value?.moderation_status === "approved",
);

const commentsLimit = 20;
const commentsOffset = ref(0);
const hasMoreComments = ref(true);
const commentTree = ref<any[]>([]);

// Построение дерева комментариев с сортировкой по лайкам и дате
function buildCommentTree() {
  const map = new Map();
  comments.value.forEach((c) => {
    map.set(c.id, { ...c, children: [] });
  });
  const roots: any[] = [];
  comments.value.forEach((c) => {
    const node = map.get(c.id);
    if (c.parent_id && map.has(c.parent_id)) {
      map.get(c.parent_id).children.push(node);
    } else if (!c.parent_id) {
      roots.push(node);
    }
  });
  const sortFn = (a: any, b: any) => {
    if (a.likes_count !== b.likes_count) return b.likes_count - a.likes_count;
    return new Date(b.created_at).getTime() - new Date(a.created_at).getTime();
  };
  roots.sort(sortFn);
  roots.forEach((r) => r.children.sort(sortFn));
  commentTree.value = roots;
}

// Загрузка поста
const loadPost = async () => {
  const id = route.params.id;
  const numericId = typeof id === "string" ? parseInt(id, 10) : -1;
  if (isNaN(numericId)) return;

  // Пост + автор + изображения + лайки
  const { data: postData, error: postError } = await supabase
    .from("post")
    .select(
      "*, author:user!author_id(*), post_images (*), likes:like_to_post(count)",
    )
    .eq("id", numericId)
    .single();
  if (postError) throw postError;
  post.value = postData;
  post.value.rating = postData.likes?.[0]?.count || 0;

  // Категории поста
  const { data: categoriesData } = await supabase
    .from("post_categories")
    .select("category_id, category:category_id(id, name, slug)")
    .eq("post_id", numericId);
  if (categoriesData) {
    postCategories.value = categoriesData.map((item) => item.category);
  }

  // Проверка жалобы на пост
  if (isAuthenticated.value && userId.value) {
    const { data: report } = await supabase
      .from("reports")
      .select("id")
      .eq("target_type", "post")
      .eq("target_id", numericId)
      .eq("reporter_id", userId.value)
      .maybeSingle();
    hasReportedPost.value = !!report;
  }

  // Общее количество комментариев
  const { count } = await supabase
    .from("comments")
    .select("*", { count: "exact", head: true })
    .eq("post_id", numericId);
  totalComments.value = count || 0;

  // Загрузка комментариев
  await loadComments(numericId, 0);

  // Проверка лайка и избранного (только если пост одобрен)
  if (isAuthenticated.value && userId.value && isPostActive.value) {
    const { data: likeData } = await supabase
      .from("like_to_post")
      .select("user_id")
      .eq("post_id", numericId)
      .eq("user_id", userId.value)
      .maybeSingle();
    isLiked.value = !!likeData;
    isFavorited.value = await isFavorite(numericId);
  } else {
    isLiked.value = false;
    isFavorited.value = false;
  }
};

// Загрузка комментариев с пагинацией
const loadComments = async (postId: number, offset: number) => {
  const { data, error } = await supabase
    .from("comments")
    .select(
      `
      *,
      user:user_id(id, username, use, avatar, auth_uid, communities:subscribers(community:communities_id(id, name, patent, rating))),
      images:comment_images(*)
    `,
    )
    .eq("post_id", postId)
    .order("likes_count", { ascending: false })
    .order("created_at", { ascending: false })
    .range(offset, offset + commentsLimit - 1);
  if (error) throw error;

  if (data) {
    let likedSet = new Set<number>();
    let reportedSet = new Set<number>();
    // Если пост активен и пользователь авторизован, подгружаем его лайки/репорты на комментарии
    if (isAuthenticated.value && userId.value && isPostActive.value) {
      const commentIds = data.map((c) => c.id);
      if (commentIds.length) {
        const { data: likes } = await supabase
          .from("like_to_comment")
          .select("comment_id")
          .eq("user_id", userId.value)
          .in("comment_id", commentIds);
        likedSet = new Set(likes?.map((l) => l.comment_id) || []);
        const { data: reports } = await supabase
          .from("reports")
          .select("target_id")
          .eq("target_type", "comment")
          .eq("reporter_id", userId.value)
          .in("target_id", commentIds);
        reportedSet = new Set(reports?.map((r) => r.target_id) || []);
      }
    }

    const enriched = data.map((c) => ({
      ...c,
      likes_count: c.likes_count || 0,
      liked: likedSet.has(c.id),
      reported: reportedSet.has(c.id),
      images: c.images || [],
    }));

    comments.value = [...comments.value, ...enriched];
    commentsOffset.value = offset + enriched.length;
    hasMoreComments.value = enriched.length === commentsLimit;
    buildCommentTree();
  }
};

const loadMoreComments = () => {
  if (post.value) loadComments(post.value.id, commentsOffset.value);
};

// Real-time подписка на новые комментарии
let subscription: any;
onMounted(async () => {
  try {
    await loadPost();
  } catch (e) {
    console.error("Error loading post:", e);
  } finally {
    loading.value = false;
  }

  const numericId = parseInt(route.params.id as string, 10);
  if (!isNaN(numericId)) {
    subscription = supabase
      .channel("comments-channel")
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "comments",
          filter: `post_id=eq.${numericId}`,
        },
        async (payload) => {
          const { data: newComment } = await supabase
            .from("comments")
            .select(
              "*, user:user_id(*), likes:like_to_comment(count), images:comment_images(*)",
            )
            .eq("id", payload.new.id)
            .single();
          if (newComment) {
            let liked = false;
            let reported = false;
            if (isAuthenticated.value && userId.value && isPostActive.value) {
              const { data: like } = await supabase
                .from("like_to_comment")
                .select("comment_id")
                .eq("comment_id", newComment.id)
                .eq("user_id", userId.value)
                .maybeSingle();
              liked = !!like;
              const { data: report } = await supabase
                .from("reports")
                .select("id")
                .eq("target_type", "comment")
                .eq("target_id", newComment.id)
                .eq("reporter_id", userId.value)
                .maybeSingle();
              reported = !!report;
            }
            const enriched = {
              ...newComment,
              likes_count: newComment.likes?.[0]?.count || 0,
              liked,
              reported,
              images: newComment.images || [],
            };
            comments.value.push(enriched);
            totalComments.value++;
            buildCommentTree();
          }
        },
      )
      .subscribe();
  }
});

onUnmounted(() => {
  if (subscription) subscription.unsubscribe();
});

function handleNewComment(comment: any) {
  comments.value.push(comment);
  totalComments.value++;
  buildCommentTree();
  replyToCommentId.value = null;
  replyToUsername.value = null;
}

async function toggleLike() {
  if (!isAuthenticated.value || !userId.value || !post.value) return;
  if (!isPostActive.value) return;
  const uid = userId.value;
  const wasLiked = isLiked.value;
  isLiked.value = !wasLiked;
  post.value.rating += wasLiked ? -1 : 1;
  try {
    if (wasLiked) {
      await supabase
        .from("like_to_post")
        .delete()
        .eq("post_id", post.value.id)
        .eq("user_id", uid);
    } else {
      await supabase
        .from("like_to_post")
        .insert({ post_id: post.value.id, user_id: uid });
    }
  } catch (e) {
    isLiked.value = wasLiked;
    post.value.rating += wasLiked ? 1 : -1;
    console.error(e);
  }
}

async function toggleFavoritePost() {
  if (!isAuthenticated.value || !post.value || !userId.value) return;
  if (!isPostActive.value) return;
  const uid = userId.value;
  const wasFavorited = isFavorited.value;
  isFavorited.value = !wasFavorited;
  try {
    if (wasFavorited) {
      await supabase
        .from("favorites")
        .delete()
        .eq("post_id", post.value.id)
        .eq("user_id", uid);
    } else {
      await supabase
        .from("favorites")
        .insert({ post_id: post.value.id, user_id: uid });
    }
  } catch (e) {
    isFavorited.value = wasFavorited;
    console.error(e);
  }
}

async function toggleCommentLike(comment: any) {
  if (!isAuthenticated.value || !userId.value) return;
  if (!isPostActive.value) return;
  const uid = userId.value;
  const wasLiked = comment.liked;
  comment.liked = !wasLiked;
  comment.likes_count += wasLiked ? -1 : 1;
  buildCommentTree();
  try {
    if (wasLiked) {
      await supabase
        .from("like_to_comment")
        .delete()
        .eq("comment_id", comment.id)
        .eq("user_id", uid);
    } else {
      await supabase
        .from("like_to_comment")
        .insert({ comment_id: comment.id, user_id: uid });
    }
  } catch (e) {
    comment.liked = wasLiked;
    comment.likes_count += wasLiked ? 1 : -1;
    buildCommentTree();
    console.error(e);
  }
}

function startReply(comment: any) {
  replyToCommentId.value = comment.id;
  replyToUsername.value = comment.user?.use || "Пользователь";
}

function cancelReply() {
  replyToCommentId.value = null;
  replyToUsername.value = null;
}

function openReportModal(type: "post" | "comment" | "user", id: number) {
  reportModalTarget.value = { type, id };
}

function reportComment(comment: any) {
  openReportModal("comment", comment.id);
}

function handleReportSubmitted() {
  if (!reportModalTarget.value) return;
  const { type, id } = reportModalTarget.value;
  if (type === "post") {
    hasReportedPost.value = true;
  } else if (type === "comment") {
    const markReported = (list: any[]) => {
      for (let c of list) {
        if (c.id === id) {
          c.reported = true;
          return true;
        }
        if (c.children && markReported(c.children)) return true;
      }
      return false;
    };
    markReported(commentTree.value);
    commentTree.value = [...commentTree.value];
  }
  reportModalTarget.value = null;
}

function openImageViewer(images: string[], index: number) {
  viewerImages.value = images;
  viewerIndex.value = index;
  showViewer.value = true;
}

function formatDate(date: string) {
  return new Date(date).toLocaleDateString("ru-RU", {
    day: "numeric",
    month: "long",
    year: "numeric",
    hour: "2-digit",
    minute: "2-digit",
  });
}
</script>
