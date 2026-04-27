<script setup>
import { ref, onMounted } from "vue"

const API_URL = "/api/gamerequests"

const form = ref({
  name: "",
  email: "",
  game: "",
  description: ""
})

const requests = ref([])
const formRef = ref(null)
const successMessage = ref("")
const errorMessage = ref("")

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

async function fetchRequests() {
    try {
        const res = await fetch(API_URL)
        requests.value = await res.json()
    } catch (err) {
        console.error("Failed to fetch requests:", err)
    }
}

async function submitForm() {
  const { valid } = await formRef.value.validate()
  if (!valid) return

  try {
    const res = await fetch(API_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(form.value)
    })

    if (!res.ok) {
      return new Error("Submission failed")
    }

    successMessage.value = "Request submitted successfully!"
    errorMessage.value = ""

    form.value = { name: "", email: "", game: "", description: "" }
    formRef.value.reset()

    await fetchRequests()
  } catch (err) {
    errorMessage.value = "Something went wrong. Please try again."
    successMessage.value = ""
  }
}

onMounted(fetchRequests)
</script>

<template>
  <h1 class="text-center text-white mt-10 text-h3" >Request a Game!</h1>
  <v-container class="border-xl mt-5 bg-grey-darken-4">
    <v-form ref="formRef">
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
      <v-btn color="grey-darken-2" class="mt-5" @click.prevent="submitForm">
        Submit
      </v-btn>
    </v-form>
  </v-container>

</template>

<style scoped>

</style>