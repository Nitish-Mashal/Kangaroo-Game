<template>
  <div
    class="min-h-screen w-full bg-gradient-to-br from-indigo-200 via-blue-100 to-purple-200 flex items-center justify-center p-4 sm:p-6">
    <div class="w-full max-w-2xl p-4 sm:p-6 bg-white rounded-2xl shadow-2xl">

      <!-- Base Word Display -->
      <div
        class="bg-gradient-to-r from-indigo-500 to-purple-500 p-4 rounded-lg text-center border border-blue-300 mb-6 shadow-md">
        <div class="text-2xl sm:text-3xl font-bold text-white tracking-widest uppercase">{{ baseWord }}</div>
      </div>

      <!-- Instruction -->
      <div class="text-sm"><span class="font-bold">NOTE:</span> Type a word using letters from
        <span class="uppercase font-bold">{{ baseWord }}</span>
      </div>

      <!-- Input + Submit Section -->
      <div class="flex flex-col sm:flex-row gap-3 pt-3 pb-2">
        <input v-model="inputWord" type="text"
          class="flex-grow rounded-md border border-indigo-300 focus:ring-2 focus:ring-indigo-400 focus:border-indigo-500 px-4 py-2 placeholder-gray-500 text-sm"
          placeholder="Look at the above Note" />
        <button @click="submitWord"
          class="w-1/2 sm:w-1/3 px-3 py-2 rounded-md text-white font-semibold bg-pink-500 hover:bg-pink-600 transition duration-300 text-sm">
          Submit
        </button>
      </div>

      <!-- Score Count -->
      <div class="text-md text-indigo-700 mb-4 font-bold text-left pt-3">{{ correctCount }} / {{ totalCount }}</div>

      <!-- Feedback Message -->
      <div v-if="statusMessage" class="mt-3 p-2 rounded text-center font-medium transition duration-300 ease-in-out"
        :class="statusClass">
        {{ statusMessage }}
      </div>

      <!-- Valid Words Section -->
      <div class="mt-6">
        <h2 class="text-lg font-semibold text-indigo-700 mb-2">Your Valid Words:</h2>
        <ul class="list-disc pl-5 space-y-1 text-gray-800 text-sm">
          <li v-for="word in validWords" :key="word" class="flex items-center gap-2">
            <span>{{ word }}</span>
            <button @click="fetchMeaning(word)" class="text-yellow-500 hover:text-yellow-600" title="Show meaning">
              💡
            </button>
          </li>
        </ul>

        <!-- Meaning Tooltip with Close Button -->
        <div v-if="wordMeaning"
          class="mt-4 bg-indigo-100 border border-indigo-300 text-indigo-800 p-3 rounded relative">
          <button class="absolute top-1 right-2 text-indigo-600 hover:text-red-500 text-sm font-bold"
            @click="closeMeaning" title="Close">
            ✖
          </button>
          <p class="text-sm pr-6"><strong>{{ meaningWord }}</strong>: {{ wordMeaning }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      baseWord: '',
      inputWord: '',
      validWords: [],
      correctCount: 0,
      totalCount: 0,
      statusMessage: '',
      statusClass: '',
      kangarooId: '687ede9bb81f5f2a28c31f99',
      meaningWord: '',
      wordMeaning: '',
    };
  },
  created() {
    this.fetchQuestion();
  },
  methods: {
    async fetchQuestion() {
      try {
        const response = await axios.get(
          `http://147.93.106.108:5100/gameplays/kangaroo/get-question?kangaroo=${this.kangarooId}`
        );
        const data = response.data;
        this.baseWord = data.parent_word.toLowerCase();
        this.totalCount = data.parent_word_count;
      } catch (err) {
        this.statusMessage = 'Failed to load base word.';
        this.statusClass = 'bg-red-100 text-red-700 border border-red-300';
      }
    },

    async submitWord() {
      const word = this.inputWord.trim().toLowerCase();

      // Validation: word must be at least 3 letters
      if (word.length < 3) {
        this.statusMessage = 'Word should be at least 3 letters long.';
        this.statusClass = 'bg-yellow-100 text-yellow-800 border border-yellow-300';
        this.clearStatusMessage();
        return;
      }

      // Validation: already submitted
      if (this.validWords.includes(word)) {
        this.statusMessage = 'You already submitted this word.';
        this.statusClass = 'bg-yellow-100 text-yellow-800 border border-yellow-300';
        this.clearStatusMessage();
        return;
      }

      this.inputWord = '';

      try {
        const response = await axios.post(
          'http://147.93.106.108:5100/gameplays/kangaroo/validate-answer',
          {
            kangaroo: this.kangarooId,
            answer: word,
          }
        );

        const status = response.data;
        this.statusMessage = status;
        this.statusClass =
          status === 'OK'
            ? 'bg-green-100 text-green-800 border border-green-300'
            : 'bg-red-100 text-red-700 border border-red-300';

        if (status === 'OK') {
          this.validWords.push(word);
          this.correctCount = this.validWords.length;
        }

        this.clearStatusMessage();
      } catch (err) {
        this.statusMessage = 'Error submitting your answer.';
        this.statusClass = 'bg-red-100 text-red-700 border border-red-300';
        this.clearStatusMessage();
      }
    },

    clearStatusMessage() {
      setTimeout(() => {
        this.statusMessage = '';
        this.statusClass = '';
      }, 3000);
    },

    async fetchMeaning(word) {
      this.meaningWord = word;
      this.wordMeaning = 'Loading...';

      try {
        const response = await axios.get(`https://api.dictionaryapi.dev/api/v2/entries/en/${word}`);
        this.wordMeaning = response.data[0]?.meanings[0]?.definitions[0]?.definition || 'No definition found.';
      } catch (err) {
        this.wordMeaning = 'Definition not found or API limit exceeded.';
      }
    },

    closeMeaning() {
      this.meaningWord = '';
      this.wordMeaning = '';
    },
  },
};
</script>

<style scoped>
input::placeholder {
  font-size: 0.875rem;
  color: #888;
}
</style>