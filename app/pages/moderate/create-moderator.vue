<!-- pages/moderate/create-moderator.vue -->
<template>
  <div class="container mx-auto px-4 py-6 max-w-2xl">
    <h1 class="text-3xl font-bold mb-6 text-gray-800">Назначить модератора</h1>

    <!-- Шаг 1: поиск пользователя -->
    <div class="mb-6">
      <label class="block text-sm font-medium text-gray-700 mb-1"
        >Email пользователя</label
      >
      <div class="flex gap-2">
        <input
          v-model="searchEmail"
          type="email"
          class="flex-1 px-4 py-2 bg-white border border-primary/20 rounded-lg focus:border-accent focus:ring-1 focus:ring-accent outline-none transition"
          placeholder="user@example.com"
          @keyup.enter="searchUser"
        />
        <button
          @click="searchUser"
          :disabled="searching"
          class="px-4 py-2 bg-accent text-white rounded-lg hover:bg-accent-dark transition disabled:opacity-50"
        >
          <span v-if="!searching">Найти</span>
          <span v-else class="flex items-center gap-2">
            <span
              class="w-4 h-4 border-2 border-white border-t-transparent rounded-full animate-spin"
            ></span>
            Поиск...
          </span>
        </button>
      </div>
    </div>

    <!-- Шаг 2: результат поиска (карточка пользователя) -->
    <div
      v-if="foundUser"
      class="mb-6 p-4 border border-primary/10 rounded-lg bg-white shadow-sm"
    >
      <div class="flex items-center gap-4">
        <img
          :src="foundUser.avatar || '/default-avatar.png'"
          class="w-16 h-16 rounded-full object-cover border border-primary/20"
          alt=""
        />
        <div class="flex-1">
          <h2 class="text-xl font-semibold text-gray-800">
            {{ foundUser.use || foundUser.username }}
          </h2>
          <p class="text-sm text-gray-500">{{ foundUser.email }}</p>
          <p class="text-sm text-gray-500">
            Текущая роль:
            <span
              :class="{
                'text-green-600': foundUser.role === 'admin',
                'text-blue-600': foundUser.role === 'moderator',
                'text-gray-600': foundUser.role === 'user',
              }"
            >
              {{
                foundUser.role === "admin"
                  ? "Администратор"
                  : foundUser.role === "moderator"
                    ? "Модератор"
                    : "Пользователь"
              }}
            </span>
          </p>
        </div>
      </div>
    </div>

    <!-- Сообщение, если пользователь не найден -->
    <div
      v-else-if="searched && !foundUser"
      class="mb-6 text-center text-gray-500"
    >
      Пользователь с таким email не найден
    </div>

    <!-- Кнопка назначения (активна только если найден подходящий пользователь) -->
    <div
      v-if="foundUser"
      class="flex justify-end gap-3 pt-4 border-t border-primary/10"
    >
      <button
        type="button"
        @click="cancel"
        class="px-5 py-2 border border-primary/30 rounded-lg text-primary hover:bg-primary/5 transition"
      >
        Отмена
      </button>
      <button
        @click="assignModerator"
        :disabled="assigning || !canAssign"
        class="px-5 py-2 bg-accent text-white rounded-lg hover:bg-accent-dark transition disabled:opacity-50"
      >
        {{ assigning ? "Назначение..." : "Назначить модератором" }}
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from "vue";
import { useAuth } from "~/composables/useAuth";

const { isAdmin } = useAuth();
const supabase = useSupabaseClient();
const router = useRouter();

if (!isAdmin.value) {
  navigateTo("/");
}

const searchEmail = ref("");
const searching = ref(false);
const searched = ref(false);
const foundUser = ref<any>(null);
const assigning = ref(false);

const canAssign = computed(() => {
  return (
    foundUser.value &&
    foundUser.value.role !== "admin" &&
    foundUser.value.role !== "moderator"
  );
});

async function searchUser() {
  if (!searchEmail.value) return;
  searching.value = true;
  searched.value = true;
  try {
    const { data, error } = await supabase
      .from("user")
      .select("id, use, username, email, avatar, role")
      .eq("email", searchEmail.value)
      .maybeSingle();
    if (error) throw error;
    foundUser.value = data || null;
  } catch (e) {
    console.error(e);
    alert("Ошибка при поиске пользователя");
    foundUser.value = null;
  } finally {
    searching.value = false;
  }
}

async function assignModerator() {
  if (!foundUser.value) return;
  assigning.value = true;
  try {
    await $fetch('/api/moderation/assign-moderator', {
      method: 'POST',
      body: { userId: foundUser.value.id }
    });
    alert('Модератор назначен успешно');
    await navigateTo('/moderate');
  } catch (e: any) {
    console.error(e);
    alert(e.message || 'Ошибка');
  } finally {
    assigning.value = false;
  }
}
function cancel() {
  router.back();
}
</script>
