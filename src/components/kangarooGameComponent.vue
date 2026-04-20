<template>
  <div class="min-h-screen w-full bg-white flex items-center justify-center p-4 relative">

    <!-- Game Icon -->
    <div class="absolute top-4 left-4">
      <img src="/kangaroo.png" class="w-12 h-12">
    </div>

    <div class="w-full max-w-2xl p-4 sm:p-6">

      <!-- Base Word -->
      <div class="text-center mb-6">
        <div class="text-3xl font-bold uppercase text-indigo-700">
          {{ baseWord }}
        </div>
      </div>

      <!-- Status -->
      <div v-if="statusMessage" class="mb-4 text-center" :class="statusClass">
        {{ statusMessage }}
      </div>

      <!-- Input -->
      <input v-model="inputWord" :disabled="isGameCompleted" type="text" class="w-full border px-4 py-2 mb-4"
        placeholder="Form words" />

      <!-- Submit -->
      <div class="flex justify-center mb-4">
        <button @click="submitWord" :disabled="isGameCompleted"
          class="px-6 py-2 bg-pink-500 text-white rounded disabled:opacity-50">
          Submit
        </button>
      </div>

      <!-- Score -->
      <div class="text-center font-bold mb-4">
        {{ correctCount }} / {{ totalCount }}
      </div>

      <!-- Timer -->
      <!-- <div class="text-center text-gray-600 mb-4">
        ⏱️ {{ elapsedSeconds }} sec
      </div> -->

      <!-- Words -->
      <div class="grid grid-cols-2 gap-4 mt-4">

        <!-- Correct Words -->
        <div>
          <ul>
            <li v-for="item in correctWords" :key="item.word" class="text-green-600">
              <div class="items-center">
                <span>{{ item.word }}</span>

                <button @click="fetchMeaning(item.word)">
                  💡
                </button>
              </div>

              <!-- Meaning -->
              <div v-if="meanings[item.word]?.show" class="text-sm text-gray-600 mt-1">
                {{ meanings[item.word].text }}
              </div>

              <!-- Loading -->
              <div v-if="loadingWord === item.word" class="text-xs text-gray-400">
                Loading...
              </div>
            </li>
          </ul>
        </div>

        <!-- Wrong Words -->
        <div>
          <ul>
            <li v-for="item in wrongWords" :key="item.word" class="text-red-500 items-center">
              <span>{{ item.word }}</span>

              <button @click="fetchMeaning(item.word)">
                💡
              </button>

              <!-- Meaning -->
              <div v-if="meanings[item.word]?.show" class="text-sm text-gray-600 mt-1">
                {{ meanings[item.word].text }}
              </div>
            </li>
          </ul>
        </div>

      </div>
      <!-- Meaning Popup -->
      <div v-if="showMeaningPopup" class="fixed inset-0 bg-black/40 flex items-center justify-center z-50">
        <div class="bg-white w-full max-w-md mx-4 rounded-lg shadow-lg p-4 relative">

          <!-- Close Button -->
          <button @click="showMeaningPopup = false" class="absolute top-2 right-2 text-gray-500 hover:text-black">
            ✖
          </button>

          <!-- Title -->
          <h2 class="text-lg font-bold mb-2 text-indigo-600">
            {{ selectedWord }}
          </h2>

          <!-- Loading -->
          <div v-if="isMeaningLoading" class="text-gray-500">
            Loading...
          </div>

          <!-- Meaning Scrollable -->
          <div v-else class="text-sm text-gray-700 max-h-60 overflow-y-auto whitespace-pre-line">
            {{ selectedMeaning }}
          </div>

        </div>
      </div>

    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, computed, onUnmounted } from "vue";
import { useRoute } from "vue-router";
import axios from "axios";

const route = useRoute();

/* ---------------- HELPERS ---------------- */
const getKangarooId = () => {
  return route.query.kangaroo || route.query.game_id;
};

const getUserId = () => {
  return route.query.user || localStorage.getItem("currentUserId");
};

/* ---------------- STATE ---------------- */
const baseWord = ref("");
const inputWord = ref("");
const submittedWords = ref([]);
const correctCount = ref(0);
const totalCount = ref(0);
const statusMessage = ref("");
const statusClass = ref("");
const isGameCompleted = ref(false);
const meanings = ref({});
const loadingWord = ref(null);
const showMeaningPopup = ref(false);
const selectedWord = ref("");
const selectedMeaning = ref("");
const isMeaningLoading = ref(false);

/* ---------------- TIMER ---------------- */
const startTime = ref(null);
const elapsedSeconds = ref(0);
let timerInterval = null;

let statusTimeout = null;

const showStatus = (message, className) => {
  statusMessage.value = message;
  statusClass.value = className;

  clearTimeout(statusTimeout);
  statusTimeout = setTimeout(() => {
    statusMessage.value = "";
  }, 2000); // 2 seconds
};

const correctWords = computed(() =>
  submittedWords.value.filter(w => w.valid)
);

const wrongWords = computed(() =>
  submittedWords.value.filter(w => !w.valid)
);

const startTimer = () => {
  const kangarooId = getKangarooId();
  if (!kangarooId) return;

  const key = `kangaroo_start_time_${kangarooId}`;
  const stored = localStorage.getItem(key);

  if (stored) {
    startTime.value = parseInt(stored);
  } else {
    startTime.value = Date.now();
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
  localStorage.removeItem(`kangaroo_start_time_${kangarooId}`);

  baseWord.value = "";
  inputWord.value = "";
  submittedWords.value = [];
  correctCount.value = 0;
  totalCount.value = 0;
  elapsedSeconds.value = 0;
  statusMessage.value = "";
  isGameCompleted.value = false;
};

/* ---------------- FETCH ---------------- */
const fetchQuestion = async () => {
  const kangarooId = getKangarooId();

  if (!kangarooId) {
    statusMessage.value = "Invalid game";
    return;
  }

  try {
    const res = await axios.get(
      "https://aqada.online/gameplays/kangaroo/get-question",
      { params: { kangaroo: kangarooId } }
    );

    baseWord.value = res.data.parent_word.toLowerCase();
    totalCount.value = res.data.parent_word_count;

    startTimer();

  } catch (err) {
    console.error(err);
    statusMessage.value = "Failed to load";
    statusClass.value = "text-red-500";
  }
};

/* ---------------- Word Meaning API ---------------- */
const fetchMeaning = async (word) => {
  selectedWord.value = word;
  selectedMeaning.value = "";
  showMeaningPopup.value = true;
  isMeaningLoading.value = true;

  try {
    const res = await axios.get(
      `https://api.dictionaryapi.dev/api/v2/entries/en/${word}`
    );

    const meaningsList =
      res.data?.[0]?.meanings?.[0]?.definitions || [];

    selectedMeaning.value = meaningsList.length
      ? meaningsList.map((d, i) => `${i + 1}. ${d.definition}`).join("\n\n")
      : "No meaning found";

  } catch (err) {
    selectedMeaning.value = "Meaning not found";
  } finally {
    isMeaningLoading.value = false;
  }
};

/* ---------------- COMPLETE ---------------- */
const completeGame = async () => {
  const kangarooId = getKangarooId();
  const userId = getUserId();
  const gameId = route.query.game_id || kangarooId;

  try {
    const formData = new URLSearchParams();

    formData.append("game_id", gameId);

    if (userId) {
      formData.append("user", userId);
    }

    formData.append(
      "params",
      JSON.stringify({
        seconds: elapsedSeconds.value
      })
    );

    await axios.post(
      "https://aqada.online/games/game-completed",
      formData,
      {
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
        },
      }
    );

    statusMessage.value = "🎉 Game Completed!";
    statusClass.value = "text-green-600 font-bold";
    isGameCompleted.value = true;

    clearInterval(timerInterval);
    localStorage.removeItem(`kangaroo_start_time_${kangarooId}`);

  } catch (err) {
    console.error("❌ Complete API failed", err);
  }
};

/* ---------------- DUPLICATE CHECK ---------------- */
const isAlreadySubmitted = (word) => {
  return submittedWords.value.some(
    (item) => item.word === word && item.valid
  );
};

/* ---------------- SUBMIT ---------------- */
const submitWord = async () => {
  const word = inputWord.value.trim().toLowerCase();
  const kangarooId = getKangarooId();

  if (!word || !kangarooId || isGameCompleted.value) return;

  // Prevent duplicate valid words
  if (isAlreadySubmitted(word)) {
    showStatus("Already submitted", "text-yellow-500");
    inputWord.value = "";
    return;
  }

  inputWord.value = "";

  try {
    const res = await axios.post(
      "https://aqada.online/gameplays/kangaroo/validate-answer",
      { kangaroo: kangarooId, answer: word }
    );

    const isValid = res.data === "OK";

    submittedWords.value.push({ word, valid: isValid });

    if (isValid) {
      correctCount.value++;
      showStatus("Correct Word ✅", "text-green-600");

      if (correctCount.value === totalCount.value) {
        await completeGame();
      }
    } else {
      showStatus("Invalid Word ❌", "text-red-500");
    }

  } catch (err) {
    console.error(err);
    statusMessage.value = "Error submitting word";
    statusClass.value = "text-red-500";
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

/* ---------------- LIFECYCLE ---------------- */
onMounted(fetchQuestion);

onUnmounted(() => {
  clearInterval(timerInterval);
});
</script>