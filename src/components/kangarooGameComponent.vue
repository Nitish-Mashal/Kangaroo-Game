<template>
  <div class="min-h-screen w-full bg-white flex items-center justify-center p-4 relative">

    <!-- Game Icon -->
    <div class="absolute top-4 left-4">
      <img src="/kangaroo.png" alt="Game Icon" class="w-12 h-12">
    </div>

    <div class="w-full max-w-2xl p-4 sm:p-6">

      <!-- Base Word -->
      <div class="text-center mb-6">
        <div class="text-3xl sm:text-4xl font-bold tracking-widest uppercase text-indigo-700">
          {{ baseWord }}
        </div>
      </div>

      <!-- Input -->
      <input v-model="inputWord" type="text"
        class="w-full rounded-md border border-gray-300 focus:ring-2 focus:ring-indigo-400 px-4 py-2 text-sm mb-4"
        placeholder="Form words using above letters" />

      <!-- Submit Button (Centered) -->
      <div class="flex justify-center mb-4">
        <button @click="submitWord"
          class="px-6 py-2 rounded-md text-white font-semibold bg-pink-500 hover:bg-pink-600 transition text-sm">
          Submit
        </button>
      </div>

      <!-- Score -->
      <div class="text-indigo-700 font-bold mb-4 text-center">
        {{ correctCount }} / {{ totalCount }}
      </div>

      <!-- Status Message -->
      <div v-if="statusMessage" class="mb-4 p-2 rounded text-center font-medium" :class="statusClass">
        {{ statusMessage }}
      </div>

      <!-- Words List -->
      <ul class="space-y-1 text-sm">
        <li v-for="item in submittedWords" :key="item.word" class="flex items-center gap-2"
          :class="item.valid ? 'text-green-600' : 'text-red-500'">
          <span class="uppercase">{{ item.word }}</span>

          <!-- Meaning Icon (only for valid words) -->
          <button v-if="item.valid" @click="fetchMeaning(item.word)" class="text-yellow-500 hover:text-yellow-600"
            title="Show meaning">
            💡
          </button>
        </li>
      </ul>

      <!-- Meaning Box -->
      <div v-if="wordMeaning" class="mt-4 bg-indigo-50 border border-indigo-300 text-indigo-800 p-3 rounded relative">
        <button class="absolute top-1 right-2 text-indigo-600 hover:text-red-500 text-sm font-bold"
          @click="closeMeaning">
          ✖
        </button>
        <p class="text-sm pr-6">
          <strong>{{ meaningWord }}</strong>: {{ wordMeaning }}
        </p>
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
      submittedWords: [],
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
        const res = await axios.get(
          `https://aqada.online/gameplays/kangaroo/get-question?kangaroo=${this.kangarooId}`
        );
        this.baseWord = res.data.parent_word.toLowerCase();
        this.totalCount = res.data.parent_word_count;
      } catch {
        this.statusMessage = 'Failed to load base word.';
        this.statusClass = 'text-red-600';
      }
    },

    async submitWord() {
      const word = this.inputWord.trim().toLowerCase();

      if (word.length < 3) {
        this.statusMessage = 'Word must be at least 3 letters.';
        this.statusClass = 'text-red-600';
        return;
      }

      if (this.submittedWords.some(w => w.word === word)) {
        this.statusMessage = 'Word already submitted.';
        this.statusClass = 'text-red-600';
        return;
      }

      this.inputWord = '';

      try {
        const res = await axios.post(
          'https://aqada.online/gameplays/kangaroo/validate-answer',
          { kangaroo: this.kangarooId, answer: word }
        );

        const isValid = res.data === 'OK';

        this.submittedWords.push({
          word,
          valid: isValid,
        });

        if (isValid) {
          this.correctCount++;
          this.statusMessage = 'Correct word!';
          this.statusClass = 'text-green-600';
        } else {
          this.statusMessage = 'Invalid word.';
          this.statusClass = 'text-red-600';
        }

      } catch {
        this.statusMessage = 'Error submitting answer.';
        this.statusClass = 'text-red-600';
      }
    },

    async fetchMeaning(word) {
      this.meaningWord = word;
      this.wordMeaning = 'Loading...';

      try {
        const res = await axios.get(
          `https://api.dictionaryapi.dev/api/v2/entries/en/${word}`
        );
        this.wordMeaning =
          res.data[0]?.meanings[0]?.definitions[0]?.definition ||
          'No definition found.';
      } catch {
        this.wordMeaning = 'Definition not found.';
      }
    },

    closeMeaning() {
      this.wordMeaning = '';
      this.meaningWord = '';
    },
  },
};
</script>
