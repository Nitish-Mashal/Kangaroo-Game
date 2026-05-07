<template>
  <div class="min-h-screen w-full bg-white p-4 relative overflow-y-auto z-0">

    <!-- Game Icon -->
    <div class="absolute top-4 left-4 z-10 pointer-events-none">
      <img src="/kangaroo.png" class="w-12 h-12" />
    </div>

    <!-- Main Container -->
    <div class="w-full max-w-2xl mx-auto p-4 sm:p-6 pb-20">

      <!-- Base Word -->
      <div class="text-center mb-6 mt-10">
        <div class="text-3xl font-bold uppercase text-indigo-700">
          {{ baseWord }}
        </div>
      </div>

      <!-- Status -->
      <div v-if="statusMessage" class="mb-4 text-center font-medium" :class="statusClass">
        {{ statusMessage }}
      </div>

      <!-- Input -->
      <input ref="wordInput" v-model="inputWord" :disabled="isGameCompleted" type="text" autocomplete="off"
        autocorrect="off" autocapitalize="none" spellcheck="false"
        class="relative z-20 w-full border border-gray-300 px-4 py-3 mb-4 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-pink-400"
        placeholder="Use minimum 4 letters" @keyup.enter="submitWord" />

      <!-- Submit -->
      <div class="flex justify-center mb-4">
        <button @click="submitWord" :disabled="isGameCompleted"
          class="px-6 py-2 bg-pink-500 text-white rounded-lg disabled:opacity-50">
          Submit
        </button>
      </div>

      <!-- Score -->
      <div class="text-center font-bold mb-6">
        {{ correctCount }} / {{ totalCount }}
      </div>

      <!-- Words -->
      <div class="grid grid-cols-2 gap-4">

        <!-- Correct -->
        <div>
          <ul>
            <li v-for="item in correctWords" :key="item.word" class="text-green-600 mb-2">
              <div class="flex items-center gap-2">
                <span>{{ item.word }}</span>
                <button @click="fetchMeaning(item.word)">💡</button>
              </div>
            </li>
          </ul>
        </div>

        <!-- Wrong -->
        <div>
          <ul>
            <li v-for="item in wrongWords" :key="item.word" class="text-red-500 mb-2">
              <div class="flex items-center gap-2">
                <span>{{ item.word }}</span>
                <button @click="fetchMeaning(item.word)">💡</button>
              </div>
            </li>
          </ul>
        </div>

      </div>

    </div>

    <!-- Meaning Popup -->
    <div v-if="showMeaningPopup" class="fixed inset-0 bg-black/40 flex items-center justify-center z-50">
      <div class="bg-white w-full max-w-md mx-4 rounded-lg shadow-lg p-4 relative">
        <button @click="showMeaningPopup = false" class="absolute top-2 right-2 text-gray-500 hover:text-black">
          ✖
        </button>

        <h2 class="text-lg font-bold mb-2 text-indigo-600">
          {{ selectedWord }}
        </h2>

        <div v-if="isMeaningLoading" class="text-gray-500">
          Loading...
        </div>

        <div v-else class="text-sm text-gray-700 max-h-60 overflow-y-auto whitespace-pre-line">
          {{ selectedMeaning }}
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import {
  ref,
  onMounted,
  watch,
  computed,
  nextTick,
  onUnmounted
} from "vue";
import { useRoute } from "vue-router";
import axios from "axios";

const route = useRoute();
const wordInput = ref(null);

/* ---------------- HELPERS ---------------- */
const getKangarooId = () =>
  route.query.kangaroo || route.query.game_id;

const getUserId = () =>
  route.query.user || localStorage.getItem("userId");

const getCompletedKey = () =>
  `kangaroo_completed_${getKangarooId()}`;

/* ---------------- STATE ---------------- */
const baseWord = ref("");
const inputWord = ref("");
const submittedWords = ref([]);
const correctCount = ref(0);
const totalCount = ref(0);

const statusMessage = ref("");
const statusClass = ref("");

const isGameCompleted = ref(false);

const showMeaningPopup = ref(false);
const selectedWord = ref("");
const selectedMeaning = ref("");
const isMeaningLoading = ref(false);

/* ---------------- TIMER ---------------- */
const startTime = ref(null);
const elapsedSeconds = ref(0);

let timerInterval = null;
let statusTimeout = null;

/* ---------------- COMPUTED ---------------- */
const correctWords = computed(() =>
  submittedWords.value.filter((w) => w.valid)
);

const wrongWords = computed(() =>
  submittedWords.value.filter((w) => !w.valid)
);

/* ---------------- STATUS ---------------- */
const showStatus = (message, className) => {
  statusMessage.value = message;
  statusClass.value = className;

  clearTimeout(statusTimeout);

  statusTimeout = setTimeout(() => {
    statusMessage.value = "";
  }, 2000);
};

/* ---------------- FOCUS ---------------- */
const focusInput = async () => {
  await nextTick();
  if (wordInput.value && !isGameCompleted.value) {
    wordInput.value.focus();
  }
};

/* ---------------- TIMER ---------------- */
const startTimer = () => {
  const kangarooId = getKangarooId();
  if (!kangarooId) return;

  const key = `kangaroo_start_time_${kangarooId}`;
  const stored = localStorage.getItem(key);

  startTime.value = stored ? parseInt(stored) : Date.now();

  if (!stored) {
    localStorage.setItem(key, startTime.value);
  }

  timerInterval = setInterval(() => {
    elapsedSeconds.value = Math.floor(
      (Date.now() - startTime.value) / 1000
    );
  }, 1000);
};

/* ---------------- RESET ---------------- */
const resetGame = () => {
  const kangarooId = getKangarooId();

  clearInterval(timerInterval);

  baseWord.value = "";
  inputWord.value = "";
  submittedWords.value = [];
  correctCount.value = 0;
  totalCount.value = 0;
  elapsedSeconds.value = 0;
  statusMessage.value = "";

  isGameCompleted.value =
    localStorage.getItem(getCompletedKey()) === "true";
};

/* ---------------- FETCH MEANING ---------------- */
const fetchMeaning = async (word) => {
  selectedWord.value = word;
  selectedMeaning.value = "";
  showMeaningPopup.value = true;
  isMeaningLoading.value = true;

  try {
    const res = await axios.get(
      `https://api.dictionaryapi.dev/api/v2/entries/en/${word}`
    );

    const meanings = res.data?.[0]?.meanings || [];

    let output = "";

    meanings.forEach((meaning) => {
      output += `(${meaning.partOfSpeech})\n`;

      meaning.definitions.forEach((def, index) => {
        output += `${index + 1}. ${def.definition}\n`;
      });

      output += "\n";
    });

    selectedMeaning.value =
      output || "No meaning found.";

  } catch (err) {
    selectedMeaning.value =
      "Meaning not available.";
  } finally {
    isMeaningLoading.value = false;
  }
};

/* ---------------- FETCH ---------------- */
const fetchQuestion = async () => {
  const kangarooId = getKangarooId();
  if (!kangarooId) return;

  try {
    const res = await axios.get(
      "https://aqada.online/gameplays/kangaroo/get-question",
      {
        params: { kangaroo: kangarooId }
      }
    );

    baseWord.value =
      res.data.parent_word.toLowerCase();

    totalCount.value =
      res.data.parent_word_count;

    if (!isGameCompleted.value) {
      startTimer();
    }

    focusInput();

  } catch (err) {
    showStatus("Failed to load", "text-red-500");
  }
};

/* ---------------- COMPLETE GAME ---------------- */
const completeGame = async () => {
  const kangarooId = getKangarooId();
  const userId = getUserId();
  const gameId =
    route.query.game_id || kangarooId;

  try {
    const formData = new URLSearchParams();

    formData.append("game_id", gameId);
    if (userId) formData.append("user", userId);

    formData.append(
      "params",
      JSON.stringify({
        seconds: elapsedSeconds.value
      })
    );

    const res = await axios.post(
      "https://aqada.online/games/game-completed",
      formData
    );

    if (res.data === "OK" || res.status === 200) {
      isGameCompleted.value = true;

      /* ✅ LOCAL FLAG */
      localStorage.setItem(
        getCompletedKey(),
        "true"
      );

      /* ✅ PARENT SYNC (IMPORTANT) */
      localStorage.setItem(
        "completed_game_id",
        gameId
      );

      /* ✅ INSTANT UI */
      showStatus(
        "🎉 Game Completed!",
        "text-green-600"
      );

      clearInterval(timerInterval);

      localStorage.removeItem(
        `kangaroo_start_time_${kangarooId}`
      );
    }

  } catch (err) {
    console.error(err);
  }
};

/* ---------------- SUBMIT ---------------- */
const submitWord = async () => {
  const word = inputWord.value
    .trim()
    .toLowerCase();

  const kangarooId = getKangarooId();

  if (!word || !kangarooId || isGameCompleted.value) return;

  inputWord.value = "";

  try {
    const res = await axios.post(
      "https://aqada.online/gameplays/kangaroo/validate-answer",
      {
        kangaroo: kangarooId,
        answer: word
      }
    );

    const isValid = res.data === "OK";

    submittedWords.value.push({
      word,
      valid: isValid
    });

    if (isValid) {
      correctCount.value++;

      showStatus("Correct ✅", "text-green-600");

      if (correctCount.value === totalCount.value) {
        await completeGame(); // 🔥 instant trigger
      }
    } else {
      showStatus("Wrong ❌", "text-red-500");
    }

    focusInput();

  } catch (err) {
    console.error(err);
    focusInput();
  }
};

/* ---------------- WATCH ---------------- */
watch(
  () => [route.query.kangaroo, route.query.game_id],
  () => {
    resetGame();
    fetchQuestion();
  }
);

/* ---------------- MOUNT ---------------- */
onMounted(() => {
  resetGame();
  fetchQuestion();
});

onUnmounted(() => {
  clearInterval(timerInterval);
});
</script>