<template>
  <div class="hand-log">
    <h2>Hand Log</h2>
    <ul>
      <li v-for="(log, index) in logs" :key="index">{{ log }}</li>
    </ul>
  </div>
</template>

<script setup lang="ts">
const props = defineProps<{
  logs: string[]
}>()
</script>

<style scoped>
.hand-log {
  max-height: 300px;
  overflow-y: auto;
  background-color: rgba(0, 0, 0, 0.5);
  font-size: 12px;
}

.hand-log ul {
  list-style-type: none;
  padding: 0;
}

.hand-log li {
  margin-bottom: 5px;
}
</style>
