<script setup lang="ts">
const { profile, isLoading } = useAuth();
</script>

<template>
  <div class="min-h-screen flex flex-col">
    <Header :profile="profile" :is-loading="isLoading" />

    <main class="flex-1" style="margin-top: 4rem">
      <slot />
      <ScrollTop />
    </main>

    <Footer />
  </div>
</template>
