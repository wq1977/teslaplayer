<script setup>
import { onMounted } from 'vue'
onMounted(() => {
  document.body.removeChild(document.getElementById('init'))
})
</script>
<template>
  <router-view />
</template>;