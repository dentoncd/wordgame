<script setup>
import { ref, watch } from 'vue'

const progress = ref(100)
let interval = null

const start = () => {
  const duration = 5000
  const stepTime = 50
  const steps = duration / stepTime
  const decrement = 100 / steps

  progress.value = 100

  clearInterval(interval)

  interval = setInterval(() => {
    progress.value -= decrement

    if (progress.value <= 0) {
      progress.value = 0
      clearInterval(interval)
    }
  }, stepTime)
}

const loading = ref(false)

watch(loading, val => {
  if (!val) return
  setTimeout(() => (loading.value = false), 3000)
})

</script>

<template>
  <h1 class="text-h3 font-weight-bold mt-6 text-center text-white">Word Dynamite</h1>

  <v-container>
    <h3 class="text-h5 text-grey-lighten-1 text-center mt-6">Enter a word containing the prompt...</h3>

    <v-container>
      <v-progress-linear color="white" :model-value="progress" class="progress-bar" height="20"></v-progress-linear>
    </v-container>
    <v-btn @click="start">Start</v-btn>


  </v-container>

  <v-container>
    <v-responsive
        class="mx-auto"
        max-width="350"
    >
      <v-text-field
          v-model="model"
          hide-details="auto"
          label=""
          color="white"
          bg-color="white"
          clearable
      ></v-text-field>
    </v-responsive>
  </v-container>



</template>

<style scoped>
.progress-bar .v-progress-linear__determinate {
  transition: width 5s linear;
}
</style>