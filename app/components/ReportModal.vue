<!-- components/ReportModal.vue -->

<script setup lang="ts">
import { ref, computed } from "vue";
import { useAuth } from "~/composables/useAuth";

const props = defineProps<{
  isOpen: boolean;
  targetType: "post" | "comment" | "user" | "community";
  targetId: number;
}>();

const emit = defineEmits<{
  (e: "close" | "submitted"): void;
}>();

const supabase = useSupabaseClient();
const { userId } = useAuth();
const reason = ref("");
const submitting = ref(false);

const targetTypeRu = computed(() => {
  switch (props.targetType) {
    case "post":
      return "пост";
    case "comment":
      return "комментарий";
    case "user":
      return "пользователя";
    case "community":
      return "сообщество";
    default:
      return ""; // или любое значение по умолчанию
  }
});

const close = () => {
  reason.value = "";
  emit("close");
};

const submit = async () => {
  if (!reason.value.trim() || !userId.value) return;
  submitting.value = true;
  try {
    await supabase.from("reports").insert({
      target_type: props.targetType,
      target_id: props.targetId,
      reporter_id: userId.value,
      reason: reason.value,
      status: "pending",
    });
    emit("submitted");
    close();
  } catch (e) {
    console.error(e);
    alert("Ошибка при отправке жалобы");
  } finally {
    submitting.value = false;
  }
};
</script>
<template>
  <div
    v-if="isOpen"
    class="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-50 dark:bg-opacity-70"
    @click.self="close"
  >
    <div class="bg-white dark:bg-gray-800 rounded-lg p-6 max-w-md w-full">
      <h3 class="text-lg font-medium text-gray-900 dark:text-white mb-4">
        Пожаловаться на {{ targetTypeRu }}
      </h3>
      <textarea
        v-model="reason"
        placeholder="Опишите причину жалобы..."
        class="w-full border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-700 text-gray-900 dark:text-white rounded p-2 mb-4"
        rows="4"
      ></textarea>
      <div class="flex justify-end gap-2">
        <button
          @click="close"
          class="px-4 py-2 text-gray-600 dark:text-gray-300 hover:text-gray-800 dark:hover:text-white transition"
        >
          Отмена
        </button>
        <button
          @click="submit"
          :disabled="!reason.trim() || submitting"
          class="px-4 py-2 bg-red-600 hover:bg-red-700 dark:bg-red-700 dark:hover:bg-red-800 text-white rounded disabled:opacity-50"
        >
          {{ submitting ? "Отправка..." : "Отправить" }}
        </button>
      </div>
    </div>
  </div>
</template>
