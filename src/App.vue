<template>
  <div class="generator">
    <v-list style="background: white; margin-bottom: 20px;">
      <v-chip style="color: crimson; margin-right: 8px; font-size: 20px;">
        MISSES: {{ misses }}
      </v-chip>
      <v-chip style="color: darkgreen; font-size: 20px">BINGOS: {{ bingos }} </v-chip>
      <v-chip style="color: blue; margin-left: 8px; font-size: 20px;">
        Current chance: {{ (currentChance * 100).toFixed(1) }}%
      </v-chip>
      <v-chip style="color: dimgrey; margin-left: 8px; font-size: 20px;">
        Actual chance: {{ (actualChance * 100).toFixed(1) }}%
      </v-chip>
      <v-chip style="color: red; margin-left: 8px; font-size: 20px;">
        Longest miss streak: {{ longestMissStreak }}
      </v-chip>
    </v-list>
    <v-list class="values">
      <v-list-item
        v-for="(chip, index) in generated"
        class="chip"
        :class="[
          {'first-green': index === 0 && chip === 'BINGO'},
          {'first-crimson': index === 0 && chip === 'MISS'},
          {'green': index !== 0 && chip === 'BINGO'},
          {'crimson': index !== 0 && chip === 'MISS'},
        ]"
      >
        {{ chip }}
      </v-list-item>
    </v-list>
    <v-btn class="generate-btn" @click="generate">Generate</v-btn>
    <v-btn class="generate-btn" style="margin-top: 15px;" @click="executeMassiveTest">Execute Test</v-btn>
    <v-progress-linear style="margin-top: 15px; height: 8px; width: 80%" color="error" :model-value="testsProgress"></v-progress-linear>
  </div>
  <div style="display: flex; flex-direction: row; align-items: center;">
    <v-select
      style="background: black; max-width: 300px; margin: 20px;"
      label="Current mode"
      :items="modes"
      variant="solo-filled"
      v-model="currentMode"
      @update:modelValue="resetAll"
    ></v-select>
    <v-btn @click="resetAll">Reset</v-btn>
  </div>
</template>

<script lang="ts" setup>
import {ref, watch} from 'vue';

const modes = ['Adjust chances', 'Stable chances'] as const;
type Mode = typeof modes[number];

const currentMode = ref<Mode>('Adjust chances');
const generated = ref<string[]>([]);
const misses = ref(0);
const bingos = ref(0);
const showSnackbar = ref(false);
const currentChance = ref(0.1); // Current probability
const streak = ref(0); // Current miss streak
const actualChance = ref(0)
const longestMissStreak = ref(0)
const testsAmount = ref(1000)
const testsProgress = ref(0)

// Constants for Adjust chances mode (optimized for 10% target)
const BASE_CHANCE = 0.1;       // Target probability (10%)
const MAX_CHANCE = 0.3;        // Upper bound (30%)
const MIN_CHANCE = 0.05;       // Lower bound (5%)
const INCREMENT = 0.005;        // Base increment after miss (0.5%)
const DECREMENT_BASE = 0.2;   // Decrement when above base (20%)
const DECREMENT_MIN = 0.05;    // Decrement when below base (2%)

// Constant for Stable chances mode
const STABLE_CHANCE = 0.1;     // Fixed probability (10%)

watch([misses, bingos], () => {
  actualChance.value = bingos.value / (misses.value + bingos.value) || 0;
})

watch(streak, () => {
  if (longestMissStreak.value < streak.value) {
    longestMissStreak.value = streak.value;
  }
})

const executeMassiveTest = () => {
  resetAll();

  for(let i = 0; i < testsAmount.value; ++i) {
    setTimeout(() => {
      testsProgress.value = (i / testsAmount.value).toFixed(2) * 100

      generate()
    }, 0)
  }
}

/**
 * Get a cryptographically secure random number
 */
const getTrueRandomNumber = (min: number, max: number) => {
  const array = new Uint32Array(1);
  window.crypto.getRandomValues(array);
  return (array[0] % (max - min + 1)) + min;
}

/**
 * Resets all counters and generated values when mode changes
 */
const resetAll = () => {
  generated.value = [];
  misses.value = 0;
  bingos.value = 0;
  streak.value = 0;
  currentChance.value = currentMode.value === 'Adjust chances' ? BASE_CHANCE : STABLE_CHANCE;
  actualChance.value = 0
  longestMissStreak.value = 0
}

/**
 * Generates result in Adjust chances mode with dynamic probability adjustment
 */
const generateAdjustChances = () => {
  const rand = Math.random();

  if (rand <= currentChance.value) {
    // BINGO case
    bingos.value++;
    showSnackbar.value = true;
    generated.value.unshift('BINGO');

    // Adjust probability down
    if (currentChance.value > BASE_CHANCE) {
      // More aggressive decrease when above target
      currentChance.value = Math.max(currentChance.value - DECREMENT_BASE, BASE_CHANCE - 0.02);
    } else {
      // Gentle decrease when below target
      currentChance.value = Math.max(currentChance.value - DECREMENT_MIN, MIN_CHANCE);
    }
    streak.value = 0;
  } else {
    // MISS case
    misses.value++;
    generated.value.unshift('MISS');
    streak.value++;

    // Logarithmic increase to prevent runaway probability
    currentChance.value = Math.min(
      BASE_CHANCE + Math.log(streak.value + 1) * INCREMENT,
      MAX_CHANCE
    );
  }
}

/**
 * Generates result in Stable chances mode with fixed probability
 */
const generateStableChances = () => {
  const randomNumber = getTrueRandomNumber(1, 100);

  const rand = (randomNumber / 100).toFixed(2);

  if (rand <= STABLE_CHANCE) {
    bingos.value++;
    showSnackbar.value = true;
    generated.value.unshift('BINGO');
    streak.value = 0;
  } else {
    misses.value++;
    generated.value.unshift('MISS');
    streak.value++;
  }
}

/**
 * Main generate function that routes to the appropriate mode handler
 */
const generate = () => {
  if (currentMode.value === 'Adjust chances') {
    generateAdjustChances();
  } else {
    generateStableChances();
  }
}

// Initialize with default mode
resetAll();
</script>

<style>
.generator {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  background: white;
}

.values {
  width: 90%;
  height: 400px;
  margin-bottom: 10px;
  display: flex;
  flex-direction: column-reverse;
  flex-wrap: nowrap;
  overflow-x: auto;
  justify-content: flex-start;
  gap: 8px;
  background: white !important;
}

.chip {
  text-align: center;
  height: 20px;
  font-size: 20px;
  padding-top: 0 !important;
  border: solid 1px black;
  border-radius: 8px !important;
}

.green {
  background: limegreen !important;
}

.crimson {
  background: crimson !important;
}

.first-green {
  background: radial-gradient(limegreen, skyblue) !important;
}

.first-crimson {
  background: radial-gradient(crimson, skyblue) !important;
}

.generate-btn {
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 16px;
  border: none;
  padding: 10px;
  border-radius: 8px;
  background: lightgray;
  transition: 0.25s;
}
</style>
