<script setup>
import { ref } from "vue"

const form = ref({
  name: "",
  email: "",
  game: "",
  description: ""
})

const nameRules = [
    value => {
      if (value) return true
      return 'You must enter a name'
    },
]

const emailRules = [
  value => {
    if (value) return true
    return 'You must enter an email'
  },
]

const gameRules = [
  value => {
    if (value) return true
    return 'You must enter a game'
  },
]

const descRules = [
  value => {
    if (value) return true
    return 'You must enter a description'
  },
]

function submitForm() {
  location.reload()
}
</script>

<template>
  <h1 class="text-center text-white mt-10 text-h3" >Request a Game!</h1>
  <v-container class="border-xl mt-5 bg-grey-darken-4">
    <v-form>
      <v-text-field
        v-model="form.name"
        label="Name"
        class="text-white mt-5"
        :rules="nameRules"
      ></v-text-field>
      <v-text-field
          v-model="form.email"
          label="Email"
          class="text-white mt-5"
          :rules="emailRules"

      ></v-text-field>
      <v-text-field
          v-model="form.game"
          label="Game"
          class="text-white mt-5"
          :rules="gameRules"
      ></v-text-field>
      <v-textarea
          v-model="form.description"
          placeholder="Short description of the game..."
          rows="2"
          variant="underlined"
          density="compact"
          :rules="descRules"
      ></v-textarea>
      <v-btn color="grey-darken-2" class="mt-5" @click="submitForm">
        Submit
      </v-btn>
    </v-form>
  </v-container>

</template>

<style scoped>

</style>