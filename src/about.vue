<script setup>
import { computed, ref } from 'vue'

const selectedSection = ref('')

const chips = [
  {
    title: "History",
    body: "This website was created for my advanced web development class. That's it.",
    id: 'history'
  },
  {
    title: "Authors",
    body: "Cashawn Denton",
    id: 'authors'
  },
  {
    title: "Sponsors",
    body: "Nobody yet",
    id: 'sponsors'
  }

]

const selectedChip = computed(() => {
  for (const chip of chips) {
    if (chip.id === selectedSection.value) {
      return chip
    }
  }
  return undefined
})
</script>

<template>
  <h1 class="text-white text-h1 text-center mt-15">About Us</h1>
  <h3 class="text-grey-lighten-1 text-center mt-5 text-h5">Learn more about the history behind rocksrocksrocks.net</h3>

  <v-container class="justify-center d-flex">
    <v-chip
        v-for="chip in chips"
        :key="chip.id"
        class="text-white justify-center ma-5"
        clickable
        @click="selectedSection = chip.id"
    >
      {{ chip.title }}
    </v-chip>
  </v-container>

  <v-container v-if="selectedChip" class="d-flex justify-center">
    <v-card class="pa-5" width="700" color="grey-darken-3" border="xl" border-color="grey-darken-4">
      <v-card-title>{{ selectedChip.title }}</v-card-title>
      <v-card-text color="grey-darken-1">{{ selectedChip.body }}</v-card-text>
    </v-card>
  </v-container>

</template>

<style scoped>

</style>
