<script setup>
import { ref, computed } from 'vue'
import home from './home.vue'
import about from './about.vue'

const routes = {
  '/' : home,
  '/about' : about
}

const currPath = ref(window.location.hash)

window.addEventListener('hashchange', () => {
  currPath.value = window.location.hash
})

const currView = computed(() => {
  return routes[currPath.value.slice(1) || '/']
})
</script>

<template>
  <a href="#/">Home</a> |
  <a href="#/about">About</a> |

  <component :is="currView"></component>
</template>