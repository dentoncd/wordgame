<script setup>
import { computed, onUnmounted, ref } from "vue";

import promptsTxt from "../prompts.txt?raw";
import pokemonTxt from "../allpokemon.txt?raw";
import hyphTxt from "../hyphen-dict.txt?raw";
import wordsTxt from "../wordbombdict.txt?raw";

// CONSTANTS
const STARTING_LIVES = 2;
const STARTING_LETTER_COUNT = 1;
const STARTING_BOMB_TIME = 10000;
const MIN_BOMB_TIME = 2500;
const STREAK_SPEEDUP = 0.97;
const TIMER_TICK = 50;

const prompts = parseLines(promptsTxt);
const dictionaries = {
  normal: new Set(parseLines(wordsTxt)),
  shiritori: new Set(parseLines(wordsTxt)),
  pokemon: new Set(parsePokemonLines(pokemonTxt)),
  hyphenated: new Set(parseLines(hyphTxt)),
  /* TODO:
      add hard mode
      add expert mode
      probably scrap blanks mode
      add more modes
   */
};

// Game Variables
const alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ".split("");

const mode = ref(null);

const prompt = ref("");
const input = ref("");

const lives = ref(STARTING_LIVES);
const streak = ref(0);

const letterCount = ref(STARTING_LETTER_COUNT);
const usedWords = ref(new Set()); // don't allow repeating user input words
const lettersUsed = ref(createLetterTracker(STARTING_LETTER_COUNT));

const message = ref("Choose Normal mode to start.");
const messageType = ref("neutral");

const gameOver = ref(false);

// Timer
const remainingTime = ref(STARTING_BOMB_TIME);
let timerId = null;
let timerStartedAt = 0;
let timerDuration = STARTING_BOMB_TIME;

// keep track of used words
const usedWordList = computed(() => Array.from(usedWords.value).slice().reverse());

// get the remaining letters once a user types in a valid word
const lettersRem = computed(() => {
  let total = 0;

  for (const letter of alphabet) {
    total += lettersUsed.value[letter];
  }
  return total;
});

// clamp bomb progress, percentage of remaining bomb time
const bombProg = computed(() => {
  return Math.max(0, Math.min(100, (remainingTime.value / timerDuration) * 100));
});

// round up seconds to nearest whole number rather than showing time in ms
const secondsRem = computed(() => Math.ceil(remainingTime.value / 1000));

/* Implement a function that takes a takes any text file and:
    - splits it into lines
    - gets rid of spaces and empty lines
    - makes everything uppercase for consistency
 */
function parseLines(text) {
  return text
    .split(/\r?\n/)
    .map((line) => line.trim().toUpperCase())
    .filter(Boolean);
}

// Take the parseLines function and returns valid alternative if pokemon name has punctuation or weird chars
function parsePokemonLines(text) {
  return Array.from(
    new Set(parseLines(text).flatMap((name) => { // ex: ["MEGA BLASTOISE"] -> ["MEGA", "BLASTOISE"]
        const lettersOnlyName = name.replace(/[^A-Z]/g, ""); // MR. MIME = MRMIME | both are valid inputs
        return lettersOnlyName && lettersOnlyName !== name ? [name, lettersOnlyName] : [name];
      }),
    ),
  );
}

/* Implement bomb party style used letters feature where
  the game keeps track of the curr used letters and once it's all ran out, resets, count += 1
 */
function createLetterTracker(count) {
  const tracker = {};

  for (const letter of alphabet) {
    tracker[letter] = count;
  }
  return tracker;
}

// choose a random prompt
function choosePrompt() {
  prompt.value = prompts[Math.floor(Math.random() * prompts.length)];
}

/* Implement choosing the next prompt for modes that allow it
  Shiritori takes the user's input and makes the last letter of the input the new prompt
 */
function chooseNextPrompt(userInput) {
  if (mode.value !== "shiritori") {
    choosePrompt();
    return;
  }

  const letters = userInput.replace(/[^A-Z]/g, ""); // gets rid of weird chars and symbols
  prompt.value = letters.slice(-1); // make last letter the new prompt
}

// How long curr bomb should last
function getBombDuration() {
  return Math.max(MIN_BOMB_TIME, STARTING_BOMB_TIME * (STREAK_SPEEDUP ** streak.value));
}

// Get mode labels
function getModeLabel(selectedMode) {
  // TODO: add all remaining modes
  if (selectedMode === "hyphenated") {
    return "Hyphenated";
  }

  if (selectedMode === "pokemon") {
    return "Pokemon";
  }

  if (selectedMode === "shiritori") {
    return "Shiritori";
  }

  return "Normal";
}

// Stop bomb timer
function stopBombTimer() {
  if (timerId) {
    clearInterval(timerId);
    timerId = null;
  }
}

/* Implement starting the bomb
    - use set interval to have the bomb run async
 */
function startBombTimer() {
  stopBombTimer();
  timerDuration = getBombDuration();
  remainingTime.value = timerDuration;
  timerStartedAt = Date.now(); // real time when bomb starts (set interval has delays)

  timerId = setInterval(() => {
    remainingTime.value = Math.max(0, timerDuration - (Date.now() - timerStartedAt)); // calculates how much time passed since bomb started

    // if time is 0, handle whether game continues or end
    if (remainingTime.value <= 0) {
      handleBombTimeout();
    }
  }, TIMER_TICK);
}

/* Implement starting the game
  - Get the curr mode the user selects
  - Need lives and keep track of streak val
  - Keep track of letter count and used words
  - Also show if the game ends and any fancy error messages
 */
function startGame(selectedMode) {
  mode.value = selectedMode;

  lives.value = STARTING_LIVES;
  streak.value = 0;

  letterCount.value = STARTING_LETTER_COUNT;
  usedWords.value = new Set();
  lettersUsed.value = createLetterTracker(STARTING_LETTER_COUNT);

  input.value = "";
  gameOver.value = false;
  messageType.value = "neutral"; // show curr msg as normal msg, neither success nor error
  message.value = `${getModeLabel(selectedMode)} mode. Type a word that includes the prompt.`;

  choosePrompt();
  startBombTimer();
}

// Implement valid word logic, handle user errors but do not penalize user
function isValidWord(userInput) {
  if (!userInput.includes(prompt.value)) {
    return [false, `Your word must include "${prompt.value}".`];
  }

  if (!dictionaries[mode.value].has(userInput)) {
    return [false, "That word is not in the dictionary."];
  }

  if (mode.value === "hyphenated" && !userInput.includes("-")) {
    return [false, "Hyphenated mode requires a hyphenated word."];
  }

  if (usedWords.value.has(userInput)) {
    return [false, "No repeats. You already used that word."];
  }

  return [true, "Valid word."];
}

// Implement keeping track of the used letters and mark them
function markUsedLetters(userInput) {
  const nextLetters = { ...lettersUsed.value }; // make a copy of the curr letters state
  const uniqueLetters = new Set(userInput); // remove duplicates

  // get rid of letter if in user input and still needs to be used
  uniqueLetters.forEach((letter) => {
    if (letter in nextLetters && nextLetters[letter] > 0) {
      nextLetters[letter] -= 1;
    }
  });

  // get remaining obj counts
  const letterCounts = Object.values(nextLetters);
  let allLettersCleared = true; // assume alphabet is cleared

  // if count is not 0, then alphabet is not cleared
  for (const count of letterCounts) {
    if (count !== 0) {
      allLettersCleared = false;
      break;
    }
  }

  // give a life to the player and incr the letter count if all letters are cleared
  if (allLettersCleared) {
    lives.value += 1;
    letterCount.value += 1;

    lettersUsed.value = createLetterTracker(letterCount.value); // reset letter tracker using new count

    // show player they were successful
    message.value = `Alphabet cleared. Bonus life earned. Lives: ${lives.value}`;
    messageType.value = "success";
    return true;
  }

  lettersUsed.value = nextLetters;
  return false;
}

/* Implement the function that handles the event that a player loses a life
    - decr life, reset streak, clear input, reset bomb speed
 */
function loseLife(reason) {
  lives.value -= 1;
  streak.value = 0;
  input.value = "";
  remainingTime.value = STARTING_BOMB_TIME;

  // end the game if no more lives are remaining
  if (lives.value <= 0) {
    gameOver.value = true;
    stopBombTimer();
    message.value = "Game over. Start a new round to try again.";
    messageType.value = "error";
    return;
  }

  choosePrompt();
  message.value = reason;
  messageType.value = "error";
  startBombTimer();
}

// Implement handling if the bomb runs out of time
function handleBombTimeout() {
  stopBombTimer();
  loseLife("Time ran out. The bomb reset and your streak was lost.");
}

/* Implement the function that handles the user's input when submitting a valid or invalid word
    - Make sure there is an active game running
    - Clean up user input
    - Ignore empty input
    - Check valid word
    - If valid: save the word in usedWords, incr streak, update letters
    - Otherwise: error
    - Choose next prompt
    - Reset bomb but make it faster
 */
function submitWord() {
  if (!mode.value || gameOver.value) {
    return;
  }

  const userInput = input.value.trim().toUpperCase();

  if (!userInput) {
    message.value = "Type a word first.";
    messageType.value = "error";
    return;
  }

  const [valid, reason] = isValidWord(userInput);

  if (!valid) {
    message.value = `${userInput}: ${reason}`;
    messageType.value = "error";
    input.value = "";
    return;
  }

  usedWords.value = new Set([...usedWords.value, userInput]);
  streak.value += 1;
  const earnedAlphabetBonus = markUsedLetters(userInput);

  if (!earnedAlphabetBonus) {
    message.value = `${userInput} accepted. Length: ${userInput.length}`;
    messageType.value = "success";
  }

  input.value = "";
  chooseNextPrompt(userInput);
  startBombTimer();
}

function quitMode() {
  stopBombTimer();
  window.location.reload(); // reload the window to go back to the mode selection
}

onUnmounted(stopBombTimer); // stop timer when player leaves page
</script>

<template>
  <div class="wordDynamite">
    <section class="gameHeader">
      <div>
        <p class="gameTitle">rocksrocksrocks.net</p>
        <h1> Word Dynamite</h1>
      </div>

      <div class="stats">
        <div>
          <span>Lives</span>
          <strong>{{ lives }}</strong>
        </div>
        <div>
          <span>Streak</span>
          <strong>{{ streak }}</strong>
        </div>
        <div>
          <span>Letters Left</span>
          <strong>{{ lettersRem }}</strong>
        </div>
        <div>
          <span>Time</span>
          <strong>{{ secondsRem }}</strong>
        </div>
      </div>
    </section>

    <section v-if="!mode" class="modePanel">
      <button class="modeButton" type="button" @click="startGame('normal')">
        Normal
      </button>
      <button class="modeButton locked" type="button" disabled>Hard</button>
      <button class="modeButton" type="button" @click="startGame('shiritori')">
        Shiritori
      </button>
      <button class="modeButton locked" type="button" disabled>Blanks</button>
      <button class="modeButton" type="button" @click="startGame('pokemon')">
        Pokemon
      </button>
      <button class="modeButton" type="button" @click="startGame('hyphenated')">
        Hyphenated
      </button>
      <button class="modeButton locked" type="button" disabled>Expert</button>
    </section>

    <section v-else class="playPanel">
      <div class="bombMeter" aria-label="Bomb timer">
        <div :style="{ width: `${bombProg}%` }"></div>
      </div>

      <div class="promptBox">
        <span>Include</span>
        <strong>{{ prompt }}</strong>
      </div>

      <form class="wordForm" @submit.prevent="submitWord">
        <input
          v-model="input"
          :disabled="gameOver"
          autocomplete="off"
          placeholder="Type a word"
          autofocus
        >
        <button type="submit" :disabled="gameOver">Submit</button>
      </form>

      <p class="message" :class="messageType">{{ message }}</p>

      <div class="actions">
        <button class="backButton" type="button" @click="quitMode">
          Back
        </button>
        <button type="button" @click="startGame(mode)">
          {{ gameOver ? "Play Again" : "Restart" }}
        </button>
      </div>
    </section>

    <section class="lettersPanel">
      <div
        v-for="letter in alphabet"
        :key="letter"
        class="letterTile"
        :class="{ cleared: lettersUsed[letter] === 0 }"
      >
        <span>{{ letter }}</span>
        <strong>{{ lettersUsed[letter] }}</strong>
      </div>
    </section>

    <section v-if="usedWordList.length" class="usedPanel">
      <h2>Used Words: {{ usedWordList.length }}</h2>
      <div class="usedWords">
        <span v-for="word in usedWordList" :key="word">{{ word }}</span>
      </div>
    </section>
  </div>
</template>

<style scoped>
.wordDynamite {
  color: white;
  margin: 0 auto;
  max-width: 980px;
  padding: 28px 20px 48px;
}

.gameHeader {
  align-items: end;
  display: flex;
  gap: 24px;
  justify-content: space-between;
}

.gameTitle {
  color: #fa3434;
  font-size: 14px;
  font-weight: 700;
  margin: 0 0 6px;
  text-transform: uppercase;
}

h1 {
  font-size: 64px;
  line-height: 64px;
  margin: 0;
}

.stats {
  display: grid;
  gap: 10px;
  grid-template-columns: repeat(4, 82px);
}

.stats div,
.modePanel,
.playPanel,
.lettersPanel,
.usedPanel {
  background: #2b2b2b;
  border: 1px solid #4a4a4a;
  border-radius: 8px;
}

.stats div {
  padding: 12px;
  text-align: center;
}

.stats span,
.promptBox span {
  color: #bdbdbd;
  display: block;
  font-size: 12px;
  text-transform: uppercase;
}

.stats strong {
  color: #fa3434;
  display: block;
  font-size: 29px;
}

.modePanel {
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(4, 140px);
  margin-top: 24px;
  padding: 18px;
}

button {
  background: #fa3434;
  border: 0;
  border-radius: 6px;
  color: #171717;
  cursor: pointer;
  font-weight: 800;
  min-height: 44px;
  padding: 0 18px;
}

button:disabled,
.locked {
  background: #555555;
  color: #aaaaaa;
  cursor: not-allowed;
}

.playPanel {
  display: grid;
  gap: 16px;
  margin-top: 24px;
  padding: 20px;
}

.bombMeter {
  background: #111111;
  border: 1px solid #676767;
  border-radius: 999px;
  height: 18px;
  overflow: hidden;
}

.bombMeter div {
  background: linear-gradient(90deg, #45d483, #ffcc66, #ff5252);
  height: 18px;
  transition: width 50ms linear;
}

.promptBox {
  text-align: center;
}

.promptBox strong {
  color: #fa3434;
  display: block;
  font-size: 96px;
  line-height: 96px;
}

.wordForm {
  display: flex;
  gap: 10px;
}

input {
  background: #111;
  border: 1px solid #666;
  border-radius: 6px;
  color: white;
  flex: 1;
  font-size: 18px;
  min-height: 46px;
  padding: 0 14px;
  text-transform: uppercase;
}

.message {
  margin: 0;
  min-height: 24px;
}

.message.success {
  color: #7ee787;
}

.message.error {
  color: #ff7b72;
}

.message.neutral {
  color: #d0d0d0;
}

.actions {
  display: flex;
  gap: 10px;
  justify-content: space-between;
}

.backButton {
  background: #555;
  color: #f2f2f2;
}

.lettersPanel {
  box-sizing: border-box;
  display: grid;
  gap: 8px;
  grid-template-columns: repeat(13, 64px);
  justify-content: center;
  margin-top: 24px;
  padding: 14px;
}

.letterTile {
  background: #151515;
  border: 1px solid #555;
  border-radius: 6px;
  min-height: 54px;
  padding: 6px 4px;
  text-align: center;
}

.letterTile span,
.letterTile strong {
  display: block;
}

.letterTile.cleared {
  background: #183d24;
  border-color: #3fb950;
}

.usedPanel {
  margin-top: 24px;
  padding: 18px;
}

.usedPanel h2 {
  font-size: 16px;
  margin: 0 0 12px;
}

.usedWords {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.usedWords span {
  background: #191919;
  border: 1px solid #444;
  border-radius: 999px;
  padding: 4px 10px;
}

/* Screen responsiveness */
@media (max-width: 980px) {
  .lettersPanel {
    grid-template-columns: repeat(13, 52px);
  }
}

@media (max-width: 860px) {
  .lettersPanel {
    grid-template-columns: repeat(7, 64px);
  }
}

@media (max-width: 720px) {
  .gameHeader,
  .wordForm {
    align-items: stretch;
    flex-direction: column;
  }

  h1 {
    font-size: 32px;
    line-height: 32px;
  }

  .stats {
    grid-template-columns: repeat(2, 140px);
  }

  .modePanel {
    grid-template-columns: repeat(2, 140px);
  }

  .promptBox strong {
    font-size: 48px;
    line-height: 48px;
  }

  .lettersPanel {
    grid-template-columns: repeat(7, 36px);
  }
}
</style>
