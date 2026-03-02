<template>
  <div class="container mx-auto px-4 py-6 max-w-5xl">
    <h1 class="text-3xl font-bold mb-6 text-gray-800">
      Редактировать сообщество
    </h1>
    <div v-if="loading" class="text-center py-10">Загрузка...</div>
    <div v-else>
      <div v-if="community.patent" class="mb-4 p-3 bg-yellow-50 border border-yellow-200 rounded text-yellow-800">
        ⚠️ Это сообщество верифицировано. Название изменять нельзя.
      </div>

    <form @submit.prevent="handleSubmit" class="space-y-6">
      <!-- Название -->
      <div>
          <label class="block text-sm font-medium text-gray-700 mb-1">Название</label>
          <input
            v-model="form.name"
            type="text"
            required
            :disabled="community.patent"
            class="w-full px-4 py-2 bg-white border border-primary/20 rounded-lg focus:border-accent focus:ring-1 focus:ring-accent outline-none transition disabled:bg-gray-100 disabled:cursor-not-allowed"
            placeholder="Название сообщества"
          />
        </div>

      <!-- Описание и предпросмотр -->
      <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-1"
            >Описание (Markdown)</label
          >
          <textarea
            v-model="form.description"
            rows="10"
            class="w-full px-4 py-2 bg-white border border-primary/20 rounded-lg focus:border-accent focus:ring-1 focus:ring-accent outline-none transition resize-y font-mono text-sm h-[300px]"
            placeholder="Описание сообщества. Поддерживается Markdown"
          ></textarea>
        </div>
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-1"
            >Предпросмотр</label
          >
          <div
            class="prose prose-sm max-w-none p-4 bg-gray-50 border border-primary/20 rounded-lg overflow-auto h-[300px]"
            v-html="renderedPreview"
          ></div>
        </div>
      </div>

      <!-- Аватар -->
      <div>
        <label class="block text-sm font-medium text-gray-700 mb-2"
          >Аватар сообщества</label
        >
        <div class="flex items-center gap-4">
          <button
            type="button"
            @click="triggerFileSelect"
            class="px-4 py-2 border border-primary/30 rounded-lg text-primary hover:bg-primary/5 transition flex items-center gap-2"
          >
            <PhotoIcon class="w-5 h-5" />
            {{ avatarFile ? "Заменить файл" : "Выбрать файл" }}
          </button>
          <span v-if="avatarFile" class="text-sm text-gray-500">{{
            avatarFile.name
          }}</span>
        </div>
        <input
          ref="fileInput"
          type="file"
          accept="image/jpeg,image/png,image/webp"
          class="hidden"
          @change="onFileSelected"
        />

        <!-- Превью аватара -->
        <div v-if="avatarPreview || community?.avatar" class="mt-3">
          <img
            :src="avatarPreview || community?.avatar"
            class="w-20 h-20 rounded-full object-cover border border-primary/20"
            alt="Аватар"
          />
          <button
            v-if="avatarPreview"
            type="button"
            @click="removeImage"
            class="mt-2 text-sm text-red-500 hover:text-red-600 transition"
          >
            Удалить
          </button>
        </div>
      </div>

      <!-- Кнопки -->
      <div class="flex justify-end gap-3 pt-4 border-t border-primary/10">
        <button
          type="button"
          @click="cancel"
          class="px-5 py-2 border border-primary/30 rounded-lg text-primary hover:bg-primary/5 transition"
        >
          Отмена
        </button>
        <button
          type="submit"
          :disabled="submitting || !form.name"
          class="px-5 py-2 bg-accent text-white rounded-lg hover:bg-accent-dark transition disabled:opacity-50 disabled:cursor-not-allowed"
        >
          {{ submitting ? "Сохранение..." : "Сохранить" }}
        </button>
      </div>
    </form>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue";
import { PhotoIcon } from "@heroicons/vue/24/outline";
import MarkdownIt from "markdown-it";
import type { Database } from "~/types/supabase";

const route = useRoute();
const router = useRouter();
const { userId } = useAuth();
const { getCommunity, updateCommunity } = useCommunity();
const { uploadFile, getPublicUrl } = useStorage();
const supabase = useSupabaseClient<Database>();

const md = new MarkdownIt();

const community = ref<any>(null);
const form = ref({ name: "", description: "" });
const avatarFile = ref<File | null>(null);
const avatarPreview = ref<string | null>(null);
const submitting = ref(false);
const loading = ref(true);
const fileInput = ref<HTMLInputElement | null>(null);



const renderedPreview = computed(() => md.render(form.value.description || ""));

// Загрузка данных сообщества
async function loadCommunity() {
  const id = Number(route.params.id);
  const comm = await getCommunity(id);
  community.value = comm;
  form.value.name = comm.name as string;
  form.value.description = comm.description || "";
  if (isNaN(id)) {
    router.back();
    return;
  }
  try {
    const comm = await getCommunity(id);
    community.value = comm;

    // Проверка прав: владелец или администратор
    const isOwner = comm.owner_id === userId.value;
    const { data } = await supabase
      .from("subscribers")
      .select("role")
      .eq("communities_id", id)
      .eq("user_id", userId.value as number)
      .maybeSingle();
    const isAdmin = data?.role === "admin";

    if (!isOwner && !isAdmin) {
      alert("У вас нет прав на редактирование этого сообщества");
      router.back();
      return;
    }

    form.value.name = comm.name as string;
    form.value.description = comm.description || "";
  } catch (e) {
    console.error("Ошибка загрузки сообщества:", e);
    router.back();
  } finally {
    loading.value = false;
  }
};

function triggerFileSelect() {
  fileInput.value?.click();
}

function onFileSelected(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (!file) return;
  avatarFile.value = file;
  if (avatarPreview.value) URL.revokeObjectURL(avatarPreview.value);
  avatarPreview.value = URL.createObjectURL(file);
  (event.target as HTMLInputElement).value = "";
}

function removeImage() {
  if (avatarPreview.value) {
    URL.revokeObjectURL(avatarPreview.value);
    avatarPreview.value = null;
  }
  avatarFile.value = null;
}

async function handleSubmit() {
  if (!community.value) return;
  submitting.value = true;
  try {
    const updates: any = {
      name: form.value.name,
      description: form.value.description,
    };
    if (avatarFile.value) {
      const uploadResult = await uploadFile(
        "avatars", // можно создать отдельный bucket, пока используем avatars
        avatarFile.value,
        `community-${community.value.id}`,
        { upsert: true, optimize: true },
      );
      updates.avatar = getPublicUrl("avatars", uploadResult.path);
    }
    await updateCommunity(community.value.id, updates);
    await navigateTo(`/communities/${community.value.id}`);
  } catch (e) {
    console.error(e);
    alert("Ошибка при сохранении");
  } finally {
    submitting.value = false;
  }
}

function cancel() {
  router.back();
}

onMounted(loadCommunity);

onUnmounted(() => {
  if (avatarPreview.value) URL.revokeObjectURL(avatarPreview.value);
});
</script>
