<template>
  <div class="bg-gray-50 min-h-screen">
    <TimeZoneConverter />
  </div>
</template>
<script setup>
import TimeZoneConverter from '../components/TimeZoneConverter.vue';
</script>
<style scoped>
/* Custom scrollbar for categories if needed */
.no-scrollbar::-webkit-scrollbar {
  display: none;
}

.no-scrollbar {
  -ms-overflow-style: none;
  /* IE and Edge */
  scrollbar-width: none;
  /* Firefox */
}
</style>