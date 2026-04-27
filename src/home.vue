<script setup>
  const games = [
    {
      title: "Word Dynamite",
      description: "Fast-paced word-based game.",
      image: "cartoon-dynamite-new-temp.png",
      href: "#/worddynamite",
    },
    {
      title: "Wordle Game",
      description: "Wordle, but the same?",
      image: "cartoon-wordle-temp.png",
      href: "#/wordlegame",
    },
    {
      title: "Wavelength",
      description: "Guessing game about finding common ground.",
      image: "cartoon-wavelength-temp.png",
      href: "#/wavelength",
    },
  ]
</script>

<template>
  <h1 class="text-h3 font-weight-bold mt-6 text-center text-white">Check out our games!</h1>
  <v-container class="mt-6">
    <v-card
      v-for="game in games"
      :key="game"
      class="mb-6"
      border="sm"
      color="grey-darken-3"
    >
      <v-row>
        <v-col cols="4">
          <v-img
              :src="game.image" height="200" cover></v-img>
        </v-col>
        <v-col cols="8">
          <v-card-title class="text-h6">{{ game.title }}</v-card-title>
          <v-card-text>{{ game.description }}</v-card-text>
          <v-card-actions>
            <v-btn
                variant="tonal"
                :href="game.href"
            >Play Game</v-btn>
          </v-card-actions>
        </v-col>
      </v-row>
    </v-card>
  </v-container>

</template>

<style scoped>

</style>