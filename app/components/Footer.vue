<template>
  <footer class="bg-gray-50 border-t border-primary/20 py-8 mt-12">
    <div class="container mx-auto px-4">
      <div class="flex flex-col md:flex-row justify-between items-center gap-4">
        <!-- Логотип и название -->
        <div class="flex items-center gap-2">
          <NuxtLink to="/" class="flex items-center gap-2">
            <img
              src="https://phlyzwfqtpddvgrprngo.supabase.co/storage/v1/object/public/avatars/logo.svg"
              class="w-8 h-8 rounded-full"
              alt="Klio"
            />
            <span class="text-lg font-medium text-gray-800">Klio</span>
          </NuxtLink>
        </div>

        <!-- Навигация -->
        <nav class="flex flex-wrap justify-center gap-x-6 gap-y-2 text-sm">
          <NuxtLink
            to="/"
            class="text-gray-600 hover:text-accent transition-colors"
            >Главная</NuxtLink
          >
          <NuxtLink
            to="/categories"
            class="text-gray-600 hover:text-accent transition-colors"
            >Категории</NuxtLink
          >
          <NuxtLink
            to="/communities"
            class="text-gray-600 hover:text-accent transition-colors"
            >Сообщества</NuxtLink
          >
          <NuxtLink
            to="/create"
            class="text-gray-600 hover:text-accent transition-colors"
            >Создать пост</NuxtLink
          >
        </nav>

        <!-- Копирайт -->
        <div class="text-sm text-gray-500">
          © {{ new Date().getFullYear() }} Klio
        </div>
      </div>
    </div>
  </footer>
</template>
