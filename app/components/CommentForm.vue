<template>
  <div>
    <!-- Индикатор ответа -->
    <div v-if="replyTo" class="mb-2 text-sm text-primary">
      Ответ <span class="font-medium">{{ replyTo }}</span>
      <button
        @click="$emit('cancel')"
        class="ml-2 text-gray-400 hover:text-primary transition"
      >
        ✕
      </button>
    </div>

    <!-- Основная строка: поле ввода + кнопки -->
    <div class="flex items-start gap-2">
      <!-- Поле ввода (адаптивная высота) -->
      <div class="flex-1">
        <textarea
          ref="textareaRef"
          v-model="text"
          rows="1"
          maxlength="5000"
          class="w-full px-4 py-2 bg-white border border-primary/20 rounded-lg resize-none overflow-hidden focus:outline-none focus:border-accent focus:ring-1 focus:ring-accent transition"
          :placeholder="
            replyTo ? 'Напишите ответ...' : 'Напишите комментарий...'
          "
          @input="adjustHeight"
          @keydown.enter.prevent="submitOnEnter"
        ></textarea>
      </div>

      <!-- Кнопки: скрепка и отправка (компактные, круглые) -->
      <div class="flex items-center gap-1 mt-1">
        <button
          @click="triggerFileSelect"
          class="p-2 text-gray-400 hover:text-accent rounded-full hover:bg-gray-100 transition"
          title="Прикрепить изображения"
          type="button"
        >
          <PaperClipIcon class="w-5 h-5" />
        </button>
        <button
          @click="submitComment"
          :disabled="!text.trim() || submitting"
          class="p-2 bg-accent text-white rounded-full hover:bg-accent/80 transition disabled:opacity-50 disabled:cursor-not-allowed"
          type="button"
        >
          <PaperAirplaneIcon v-if="!submitting" class="w-5 h-5" />
          <span
            v-else
            class="w-5 h-5 border-2 border-white border-t-transparent rounded-full animate-spin"
          ></span>
        </button>
      </div>
    </div>

    <!-- Скрытый input для файлов -->
    <input
      ref="fileInput"
      type="file"
      accept="image/jpeg,image/png,image/webp"
      multiple
      class="hidden"
      @change="onFileSelected"
    />

    <!-- Превью изображений -->
    <div v-if="imagePreviews.length" class="flex flex-wrap gap-2 mt-3">
      <div v-for="(preview, idx) in imagePreviews" :key="idx" class="relative">
        <img
          :src="preview"
          class="w-12 h-12 object-cover rounded border border-primary/20"
        />
        <button
          @click="removeImage(idx)"
          class="absolute -top-1.5 -right-1.5 bg-red-500 text-white rounded-full w-5 h-5 flex items-center justify-center text-xs hover:bg-red-600 shadow-sm"
          type="button"
        >
          ×
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick } from "vue";
import { PaperClipIcon, PaperAirplaneIcon } from "@heroicons/vue/24/outline";
import type { Database } from "~/types/supabase";

const props = defineProps<{
  postId: number;
  parentId?: number | null;
  replyTo?: string | null;
}>();

const emit = defineEmits<{
  (e: "comment-added", comment: any): void;
  (e: "cancel"): void;
}>();

const supabase = useSupabaseClient<Database>();
const { userId } = useAuth();
const { uploadFile, getPublicUrl } = useStorage();

const text = ref("");
const submitting = ref(false);
const selectedFiles = ref<File[]>([]);
const imagePreviews = ref<string[]>([]);
const fileInput = ref<HTMLInputElement | null>(null);
const textareaRef = ref<HTMLTextAreaElement | null>(null);

// Адаптивная высота textarea
function adjustHeight() {
  if (textareaRef.value) {
    textareaRef.value.style.height = "auto";
    textareaRef.value.style.height = textareaRef.value.scrollHeight + "px";
  }
}

// Вызов adjustHeight при изменении текста
watch(text, () => {
  nextTick(adjustHeight);
});

function triggerFileSelect() {
  fileInput.value?.click();
}

function onFileSelected(event: Event) {
  const files = (event.target as HTMLInputElement).files;
  if (!files) return;
  const newFiles = Array.from(files);
  selectedFiles.value = [...selectedFiles.value, ...newFiles];
  newFiles.forEach((file) => {
    const url = URL.createObjectURL(file);
    imagePreviews.value.push(url);
  });
  (event.target as HTMLInputElement).value = "";
}

function removeImage(index: number) {
  const url = imagePreviews.value[index];
  if (url) URL.revokeObjectURL(url);
  imagePreviews.value.splice(index, 1);
  selectedFiles.value.splice(index, 1);
}

function submitOnEnter(e: KeyboardEvent) {
  if (!e.shiftKey) {
    e.preventDefault();
    submitComment();
  }
}

async function submitComment() {
  if (!text.value.trim() || !userId.value) return;
  submitting.value = true;
  try {
    const { data: comment, error: commentError } = await supabase
      .from("comments")
      .insert({
        post_id: props.postId,
        user_id: userId.value,
        text: text.value.trim(),
        parent_id: props.parentId || null,
      })
      .select("*, user:user_id(*)")
      .single();
    if (commentError) throw commentError;

    if (selectedFiles.value.length) {
      const imageRecords: Database["public"]["Tables"]["comment_images"]["Insert"][] =
        [];
      for (let i = 0; i < selectedFiles.value.length; i++) {
        const file = selectedFiles.value[i];
        const uploadResult = await uploadFile(
          "comments",
          file as any,
          `comment-${comment.id}`,
          { upsert: false, optimize: true },
        );
        const url = getPublicUrl("comments", uploadResult.path);
        imageRecords.push({
          comment_id: comment.id,
          url,
          sort_order: i,
        });
      }
      const { error: imagesError } = await supabase
        .from("comment_images")
        .insert(imageRecords);
      if (imagesError) throw imagesError;
    }

    // Очистка
    text.value = "";
    imagePreviews.value.forEach((url) => {
      if (url) URL.revokeObjectURL(url);
    });
    imagePreviews.value = [];
    selectedFiles.value = [];
    emit("comment-added", comment);
  } catch (e) {
    console.error(e);
    alert("Ошибка при отправке комментария");
  } finally {
    submitting.value = false;
  }
}

// При монтировании устанавливаем начальную высоту
onMounted(() => {
  nextTick(adjustHeight);
});
</script>
