<script setup>
import { ref, computed } from 'vue'
import home from './home.vue'
import about from './about.vue'
import worddynamite from './worddynamite.vue'
import wordlegame from './wordlegame.vue'
import wavelength from './wavelength.vue'
import request from './request.vue'

const routes = {
  '/' : home,
  '/about' : about,
  '/request' : request,
  '/worddynamite' : worddynamite,
  '/wordlegame' : wordlegame,
  '/wavelength' : wavelength
}

const currPath = ref(window.location.hash)
const drawer = ref(false)

window.addEventListener('hashchange', () => {
  currPath.value = window.location.hash
})

const currView = computed(() => {
  return routes[currPath.value.slice(1) || '/']
})
</script>

<template>
  <v-app class="bg-color-custom">
    <v-navigation-drawer class="text-blue-grey-lighten-5" v-model="drawer" color="grey-darken-4">
      <v-list-item
          prepend-icon="mdi-home"
          href="#/"
          title="Home"
          @click="drawer = !drawer"
      >
      </v-list-item>
      <v-list-item
          prepend-icon="mdi-information-variant-circle-outline"
          href="#/about"
          title="About Us"
          @click="drawer = !drawer"
      >
      </v-list-item>
      <v-list-item
          prepend-icon="mdi-account-box-outline"
          href="#/request"
          title="Request a Game"
          @click="drawer = !drawer"
      >
      </v-list-item>
    </v-navigation-drawer>
    <v-app-bar color="grey-darken-4" class="text-blue-grey-lighten-5 text-h2 mainHeader" height="100">
      <v-app-bar-nav-icon @click="drawer = !drawer"></v-app-bar-nav-icon>
      <v-app-bar-title class="text-h2 mainHeader">Word Games</v-app-bar-title>
    </v-app-bar>
    <v-main>
      <component :is="currView"></component>
    </v-main>
    <v-footer app class="footer-color-custom" border="md">Copyright 2026</v-footer>
  </v-app>
</template>

<style scoped>
.mainHeader {
  font-family: "Changa One", sans-serif;
  font-weight: 400;
  font-style: normal;
  line-height: 200;
}
.bg-color-custom {
  background-color: #212121
}
.footer-color-custom {
  background-color: #191919
}
</style>
