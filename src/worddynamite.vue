<script setup>
import {computed, onUnmounted, ref} from "vue";

import promptsTxt from "./data/prompts.txt?raw";
import pokemonTxt from "./data/allpokemon.txt?raw";
import wordsTxt from "./data/wordbombdict.txt?raw";

// CONSTANTS
const STARTING_LIVES = 2;
const STARTING_LETTER_COUNT = 1;
const STARTING_BOMB_TIME = 10000;
const MIN_BOMB_TIME = 2500;
const STREAK_SPEEDUP = 0.99;
const HARD_BOMB_TIME = 7000;
const HARD_MIN_BOMB_TIME = 1800;
const HARD_STREAK_SPEEDUP = 0.94;
const EXPERT_MIN_PROMPT_LENGTH = 3;
const MIN_EXACT_LENGTH = 4;
const MAX_EXACT_LENGTH = 12;
const TIMER_TICK = 50;

const prompts = parseLines(promptsTxt);
const normalWords = parseLines(wordsTxt);

// store each mode's valid words so different modes can use different dictionaries later
const dictionaries = {
  normal: new Set(normalWords),
  hard: new Set(normalWords),
  expert: new Set(normalWords),
  exactLength: new Set(normalWords),
  blanks: new Set(normalWords),
  speedrun: new Set(normalWords),
  forbidden: new Set(normalWords),
  shiritori: new Set(normalWords),
  pokemon: new Set(parsePokemonLines(pokemonTxt)),
  /* TODO:
      add more modes
   */
};

// mode config, makes it easier to change lives/timer speed without editing game logic
const modeSettings = {
  normal: {
    label: "Normal",
    lives: STARTING_LIVES,
    bombTime: STARTING_BOMB_TIME,
    minBombTime: MIN_BOMB_TIME,
    speedup: STREAK_SPEEDUP,
  },
  hard: {
    label: "Hard",
    lives: STARTING_LIVES,
    bombTime: HARD_BOMB_TIME,
    minBombTime: HARD_MIN_BOMB_TIME,
    speedup: HARD_STREAK_SPEEDUP,
  },
  expert: {
    label: "Expert",
    lives: STARTING_LIVES,
    bombTime: STARTING_BOMB_TIME,
    minBombTime: MIN_BOMB_TIME,
    speedup: STREAK_SPEEDUP,
    minPromptLength: EXPERT_MIN_PROMPT_LENGTH,
  },
  exactLength: {
    label: "Exact Length",
    lives: STARTING_LIVES,
    bombTime: STARTING_BOMB_TIME,
    minBombTime: MIN_BOMB_TIME,
    speedup: STREAK_SPEEDUP,
  },
  blanks: {
    label: "Blanks",
    lives: STARTING_LIVES,
    bombTime: STARTING_BOMB_TIME,
    minBombTime: MIN_BOMB_TIME,
    speedup: STREAK_SPEEDUP,
  },
  speedrun: {
    label: "Speedrun",
    lives: 1,
    bombTime: STARTING_BOMB_TIME,
    minBombTime: MIN_BOMB_TIME,
    speedup: STREAK_SPEEDUP,
  },
  forbidden: {
    label: "Forbidden Letter",
    lives: STARTING_LIVES,
    bombTime: STARTING_BOMB_TIME,
    minBombTime: MIN_BOMB_TIME,
    speedup: STREAK_SPEEDUP,
  },
  shiritori: {
    label: "Shiritori",
    lives: STARTING_LIVES,
    bombTime: STARTING_BOMB_TIME,
    minBombTime: MIN_BOMB_TIME,
    speedup: STREAK_SPEEDUP,
  },
  pokemon: {
    label: "Pokemon",
    lives: STARTING_LIVES,
    bombTime: STARTING_BOMB_TIME,
    minBombTime: MIN_BOMB_TIME,
    speedup: STREAK_SPEEDUP,
  },
};

// Game Variables
const alphabet = "ABCDEFGHIJKLMNOPQRSTUVWY".split("");

const mode = ref(null);

const prompt = ref("");
const forbiddenLetter = ref("");
const exactLength = ref(0);
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
const elapsedTime = ref(0);
let timerId = null;
let timerStartedAt = 0;
let timerDuration = STARTING_BOMB_TIME;
let gameStartedAt = 0;

// keep track of used words
const usedWordList = computed(() => {
  const words = Array.from(usedWords.value);
  return words.slice().reverse();
});

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
  const remainingPercent = (remainingTime.value / timerDuration) * 100;
  return Math.max(0, Math.min(100, remainingPercent));
});

// round up seconds to nearest whole number rather than showing time in ms
const secondsRem = computed(() => {
  return Math.ceil(remainingTime.value / 1000);
});

const elapsedTimeLabel = computed(() => {
  return formatElapsedTime(elapsedTime.value);
});

// Blanks mode should say "Match" because the user is matching a pattern, not including text
const promptActionLabel = computed(() => {
  if (mode.value === "blanks") {
    return "Match";
  }

  return "Include";
});

// get curr mode settings, fallback to normal so the game still has values before mode is picked
const activeModeSettings = computed(() => {
  const settings = modeSettings[mode.value];

  if (settings) {
    return settings;
  }

  return modeSettings.normal;
});

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
  const pokemonNames = parseLines(text);
  const pokemonNamesAndAlternates = pokemonNames.flatMap((name) => {
  const lettersOnlyName = name.replace(/[^A-Z]/g, ""); // MR. MIME = MRMIME | both are valid inputs

  if (lettersOnlyName && lettersOnlyName !== name) {
    return [name, lettersOnlyName];
  }
  return [name];
});

  const uniquePokemonNames = new Set(pokemonNamesAndAlternates);

  return Array.from(uniquePokemonNames);
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
  if (mode.value === "blanks") {
    // Blanks mode uses its own prompt since it needs to add the underscore somewhere
    chooseBlanksPattern();
    return;
  }

  // Expert mode filters out shorter prompts so the prompt is harder to fit in a word
  const minPromptLength = activeModeSettings.value.minPromptLength || 0;
  let promptPool = prompts;

  if (minPromptLength) {
    promptPool = prompts.filter((entry) => entry.length >= minPromptLength);
  }

  let choices = prompts;

  if (promptPool.length) {
    choices = promptPool;
  }

  const choiceIndex = Math.floor(Math.random() * choices.length);

  prompt.value = choices[choiceIndex];
}

// choose where the blank should go in blanks mode
function chooseBlanksPattern() {
  const sourcePromptIndex = Math.floor(Math.random() * prompts.length);
  const sourcePrompt = prompts[sourcePromptIndex];
  let choices = ["before", "after"];

  // can only replace a real letter if the prompt has more than one letter
  if (sourcePrompt.length > 1) {
    choices = ["replace", "before", "after"];
  }

  const blankModeIndex = Math.floor(Math.random() * choices.length);
  const blankMode = choices[blankModeIndex];

  if (blankMode === "before") {
    prompt.value = `_${sourcePrompt}`;
    return;
  }

  if (blankMode === "after") {
    prompt.value = `${sourcePrompt}_`;
    return;
  }

  const blankIndex = Math.floor(Math.random() * sourcePrompt.length);

  // replace one letter with _ so the player has to find a word that fits
  prompt.value = sourcePrompt
    .split("")
    .map((letter, index) => {
      if (index === blankIndex) {
        return "_";
      }

      return letter;
    })
    .join("");
}

// choose a letter that is not already in the prompt so forbidden mode is still possible
function chooseForbiddenLetter() {
  if (mode.value !== "forbidden") {
    forbiddenLetter.value = "";
    return;
  }

  const promptLetters = new Set(prompt.value.replace(/[^A-Z]/g, ""));
  const choices = alphabet.filter((letter) => !promptLetters.has(letter));
  const choiceIndex = Math.floor(Math.random() * choices.length);

  forbiddenLetter.value = choices[choiceIndex];
}

/* Implement exact length mode:
  - find words that include the prompt
  - keep only reasonable word lengths
  - choose one of those lengths for the player to match
 */
function chooseExactLength() {
  if (mode.value !== "exactLength") {
    exactLength.value = 0;
    return;
  }

  const matchingWords = normalWords.filter((word) => {
    const includesPrompt = word.includes(prompt.value);
    const isLongEnough = word.length >= MIN_EXACT_LENGTH;
    const isShortEnough = word.length <= MAX_EXACT_LENGTH;

    return includesPrompt && isLongEnough && isShortEnough;
  });

  const matchingLengths = matchingWords.map((word) => {
    return word.length;
  });

  const possibleLengths = Array.from(new Set(matchingLengths));
  let choices = possibleLengths;

  if (!possibleLengths.length) {
    choices = Array.from({ length: MAX_EXACT_LENGTH - MIN_EXACT_LENGTH + 1 }, (_, index) => {
      return MIN_EXACT_LENGTH + index;
    });
  }

  const choiceIndex = Math.floor(Math.random() * choices.length);

  exactLength.value = choices[choiceIndex];
}

// choose every extra constraint that can change each round
function chooseRoundConstraints() {
  choosePrompt();
  chooseForbiddenLetter();
  chooseExactLength();
}

/* Implement choosing the next prompt for modes that allow it
  Shiritori takes the user's input and makes the last letter of the input the new prompt
 */
function chooseNextPrompt(userInput) {
  if (mode.value !== "shiritori") {
    chooseRoundConstraints();
    return;
  }

  const letters = userInput.replace(/[^A-Z]/g, ""); // gets rid of weird chars and symbols
  prompt.value = letters.slice(-1); // make last letter the new prompt
  chooseForbiddenLetter();
  chooseExactLength();
}

// How long curr bomb should last
function getBombDuration() {
  const settings = activeModeSettings.value;
  const speedMultiplier = settings.speedup ** streak.value;
  const streakDuration = settings.bombTime * speedMultiplier;
  return Math.max(settings.minBombTime, streakDuration);
}

// Get mode labels
function getModeLabel(selectedMode) {
  const settings = modeSettings[selectedMode];

  if (settings) {
    return settings.label;
  }

  return "Normal";
}

function formatElapsedTime(milliseconds) {
  const totalSeconds = Math.floor(milliseconds / 1000);
  const minutes = Math.floor(totalSeconds / 60);
  const seconds = totalSeconds % 60;

  return `${minutes}:${String(seconds).padStart(2, "0")}`;
}

// keep the survived time display updated while the bomb timer is running
function updateElapsedTime() {
  if (!gameStartedAt) {
    elapsedTime.value = 0;
    return;
  }

  elapsedTime.value = Date.now() - gameStartedAt;
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
    const millisecondsSinceTimerStarted = Date.now() - timerStartedAt;
    const nextRemainingTime = timerDuration - millisecondsSinceTimerStarted;

    remainingTime.value = Math.max(0, nextRemainingTime); // calculates how much time passed since bomb started
    updateElapsedTime();

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

  lives.value = activeModeSettings.value.lives;
  streak.value = 0;

  letterCount.value = STARTING_LETTER_COUNT;
  usedWords.value = new Set();
  lettersUsed.value = createLetterTracker(STARTING_LETTER_COUNT);

  input.value = "";
  gameOver.value = false;
  elapsedTime.value = 0;
  gameStartedAt = Date.now();
  messageType.value = "neutral"; // show curr msg as normal msg, neither success nor error
  message.value = `${getModeLabel(selectedMode)} mode. ${getModeInstruction(selectedMode)}`;

  chooseRoundConstraints();
  startBombTimer();
}

// Mode instruction
function getModeInstruction(selectedMode) {
  if (selectedMode === "exactLength") {
    return "Type a word that includes the prompt and has the exact required length.";
  }

  if (selectedMode === "blanks") {
    return "Type a word that includes the blank pattern.";
  }

  if (selectedMode === "speedrun") {
    return "Clear the alphabet once before the bomb gets you.";
  }

  if (selectedMode === "forbidden") {
    return "Type a word that includes the prompt and avoids the forbidden letter.";
  }

  if (selectedMode === "expert") {
    return "Type a word that includes the longer prompt.";
  }

  return "Type a word that includes the prompt.";
}

// Implement valid word logic, handle user errors but do not penalize user
function isValidWord(userInput) {
  // Blanks mode does not check includes because the underscore can be any letter
  if (mode.value === "blanks" && !matchesBlanksPattern(userInput)) {
    return [false, `Your word must match "${prompt.value}".`];
  }

  if (mode.value !== "blanks" && !userInput.includes(prompt.value)) {
    return [false, `Your word must include "${prompt.value}".`];
  }

  if (mode.value === "exactLength" && userInput.length !== exactLength.value) {
    return [false, `Exact Length mode requires ${exactLength.value} letters.`];
  }

  if (!dictionaries[mode.value].has(userInput)) {
    return [false, "That word is not in the dictionary."];
  }

  if (mode.value === "forbidden" && userInput.includes(forbiddenLetter.value)) {
    return [false, `Forbidden Letter mode bans "${forbiddenLetter.value}" this round.`];
  }

  if (usedWords.value.has(userInput)) {
    return [false, "No repeats. You already used that word."];
  }

  return [true, "Valid word."];
}

// make _ work like any capital letter when checking the user's word
function matchesBlanksPattern(userInput) {
  const pattern = prompt.value.replaceAll("_", "[A-Z]");
  const patternRegex = new RegExp(pattern);

  return patternRegex.test(userInput);
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
    // Speedrun ends after one alphabet clear instead of giving a bonus life
    if (mode.value === "speedrun") {
      lettersUsed.value = nextLetters;
      gameOver.value = true;
      updateElapsedTime();
      stopBombTimer();
      message.value = `Speedrun complete. You cleared the alphabet in ${elapsedTimeLabel.value}.`;
      messageType.value = "success";
      return true;
    }

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
  remainingTime.value = activeModeSettings.value.bombTime;

  // end the game if no more lives are remaining
  if (lives.value <= 0) {
    gameOver.value = true;
    updateElapsedTime();
    stopBombTimer();
    message.value = `Game over. You survived ${elapsedTimeLabel.value}. Start a new round to try again.`;
    messageType.value = "error";
    return;
  }

  chooseRoundConstraints();
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

  // speedrun can end inside markUsedLetters, so do NOT start a new prompt/timer after winning
  if (gameOver.value) {
    return;
  }

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
  <div class="wordDynamite" :class="{ playing: mode }">
    <section class="gameHeader" :class="{ playing: mode }">
      <div>
        <p class="siteTitle">rocksrocksrocks.net</p>
        <h1> Word Dynamite</h1>
      </div>

      <div class="stats">
        <div>
          <span>Lives</span>
          <strong>{{ mode ? lives : "--" }}</strong>
        </div>
        <div>
          <span>Streak</span>
          <strong>{{ mode ? streak : "--" }}</strong>
        </div>
        <div>
          <span>Letters Left</span>
          <strong>{{ mode ? lettersRem : "--" }}</strong>
        </div>
        <div>
          <span>Time</span>
          <strong>{{ mode ? secondsRem : "--" }}</strong>
        </div>
        <div>
          <span>Survived</span>
          <strong>{{ mode ? elapsedTimeLabel : "--" }}</strong>
        </div>
      </div>
    </section>

    <section v-if="!mode" class="modePanel">
      <button class="modeButton" type="button" @click="startGame('normal')">
        Normal
      </button>
      <button class="modeButton" type="button" @click="startGame('hard')">
        Hard
      </button>
      <button class="modeButton" type="button" @click="startGame('expert')">
        Expert
      </button>
      <button class="modeButton" type="button" @click="startGame('shiritori')">
        Shiritori
      </button>
      <button class="modeButton" type="button" @click="startGame('blanks')">
        Blanks
      </button>
      <button class="modeButton" type="button" @click="startGame('pokemon')">
        Pokemon
      </button>
      <button class="modeButton" type="button" @click="startGame('exactLength')">
        Exact Length
      </button>
      <button class="modeButton" type="button" @click="startGame('speedrun')">
        Speedrun
      </button>
      <button class="modeButton" type="button" @click="startGame('forbidden')">
        Forbidden Letter
      </button>
    </section>

    <section v-if="!mode" class="infoPanel">
      <h2>What Is Word Dynamite?</h2>
      <p>
        Word Dynamite is a timed word game where each answer must satisfy the current prompt before
        the bomb runs out. Valid words reset the timer, build your streak, and mark off letters in
        the alphabet tracker. Clear every letter to earn a bonus life and make the next alphabet
        clear harder.
      </p>

      <h2>Information about Modes</h2>
      <div class="modeInfoGrid">
        <div>
          <h3>Normal</h3>
          <p>Type any dictionary word that includes the prompt.</p>
        </div>
        <div>
          <h3>Hard</h3>
          <p>Normal rules, but the bomb starts shorter and speeds up faster.</p>
        </div>
        <div>
          <h3>Expert</h3>
          <p>Normal rules, but prompts are longer and harder to fit into words.</p>
        </div>
        <div>
          <h3>Shiritori</h3>
          <p>Each new prompt is the final letter of your last accepted word.</p>
        </div>
        <div>
          <h3>Blanks</h3>
          <p>Match a prompt pattern with one missing letter, such as <span>I_G</span> or <span>ING_</span>.</p>
        </div>
        <div>
          <h3>Pokemon</h3>
          <p>Use Pokemon names instead of the normal word dictionary.</p>
        </div>
        <div>
          <h3>Exact Length</h3>
          <p>Type a word that includes the prompt and has the shown number of letters (this mode is very hard).</p>
        </div>
        <div>
          <h3>Speedrun</h3>
          <p>Start with one life and win by clearing the alphabet once as fast as possible.</p>
        </div>
        <div>
          <h3>Forbidden Letter</h3>
          <p>Include the prompt while avoiding the forbidden letter shown each round.</p>
        </div>
      </div>
    </section>

    <div v-else class="gameArea">
      <section class="playPanel">
        <div class="bombMeter" aria-label="Bomb timer">
          <div :style="{ width: `${bombProg}%` }"></div>
        </div>

        <div class="promptBox">
          <span>{{ promptActionLabel }}</span>
          <strong>{{ prompt }}</strong>
          <p v-if="mode === 'exactLength'" class="constraintHint">
            Length: {{ exactLength }}
          </p>
          <p v-if="mode === 'forbidden'" class="forbiddenHint">
            Forbidden: {{ forbiddenLetter }}
          </p>
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
    </div>

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
  max-width: 1120px;
  padding: 28px 20px 48px;
}

.wordDynamite.playing {
  align-items: start;
  display: grid;
  gap: 16px;
  grid-template-columns: minmax(0, 1fr) 156px;
}

.gameHeader {
  align-items: end;
  display: flex;
  gap: 24px;
  justify-content: space-between;
}

.gameHeader.playing {
  align-items: start;
  grid-column: 1;
  justify-content: space-between;
}

.siteTitle {
  color: #fa3434;
  font-size: 14px;
  font-weight: 700;
  margin: 0 0 6px;
  text-transform: uppercase;
}

h1 {
  font-size: 54px;
  line-height: 54px;
  margin: 0;
}

.stats {
  display: grid;
  gap: 10px;
  grid-template-columns: repeat(5, 82px);
}

.modeButton {
  color: black;
}

.stats div,
.modePanel,
.infoPanel,
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
  grid-template-columns: repeat(7, 140px);
  margin-top: 24px;
  padding: 12px;
}

.infoPanel {
  margin-top: 24px;
  padding: 22px;
}

.infoPanel h2 {
  color: #fa3434;
  font-size: 22px;
  line-height: 26px;
  margin: 0 0 10px;
}

.infoPanel h2:not(:first-child) {
  margin-top: 24px;
}

.infoPanel p {
  color: #d0d0d0;
  line-height: 24px;
  margin: 0;
}

.modeInfoGrid {
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(3, 1fr);
}

.modeInfoGrid div {
  background: #191919;
  border: 1px solid #444444;
  border-radius: 6px;
  padding: 14px;
}

.modeInfoGrid h3 {
  color: #ffffff;
  font-size: 16px;
  margin: 0 0 6px;
}

.modeInfoGrid span {
  color: #fa3434;
  font-weight: 800;
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

.gameArea {
  display: contents;
}

.playPanel {
  grid-column: 1;
  display: grid;
  gap: 16px;
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

.constraintHint,
.forbiddenHint {
  color: #ffcc66;
  font-size: 18px;
  font-weight: 800;
  margin: 8px 0 0;
  text-transform: uppercase;
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
  gap: 6px;
  align-self: stretch;
  grid-column: 2;
  grid-row: 1 / 3;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(8, 1fr);
  justify-content: center;
  overflow: hidden;
  padding: 10px;
}

.letterTile {
  background: #151515;
  border: 1px solid #555;
  border-radius: 6px;
  min-height: 36px;
  padding: 4px;
  text-align: center;
}

.letterTile span,
.letterTile strong {
  display: block;
}

.letterTile span {
  font-size: 13px;
}

.letterTile strong {
  color: #fa3434;
  font-size: 15px;
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
  .wordDynamite.playing {
    grid-template-columns: minmax(0, 1fr) 138px;
  }

  .lettersPanel {
    grid-template-columns: repeat(3, 38px);
  }
}

@media (max-width: 860px) {
  .wordDynamite.playing {
    grid-template-columns: 1fr;
  }

  .gameArea {
    display: grid;
    gap: 16px;
  }

  .lettersPanel {
    grid-column: auto;
    grid-row: auto;
    grid-template-columns: repeat(13, minmax(0, 1fr));
    overflow: visible;
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

  .modeInfoGrid {
    grid-template-columns: 1fr;
  }

  .promptBox strong {
    font-size: 48px;
    line-height: 48px;
  }

  .lettersPanel {
    grid-template-columns: repeat(7, minmax(0, 1fr));
  }
}
</style>
