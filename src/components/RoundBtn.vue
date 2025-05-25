<template>
  <v-btn @mousedown.prevent @click="$emit('click')" class="mx-1" :icon="icon"
    size="x-large"></v-btn>
</template>

<script setup>

import { defineProps, defineEmits } from 'vue'
defineProps(['icon'])
defineEmits(['click'])
</script>
