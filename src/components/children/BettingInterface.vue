<template>
  <div class="betting-interface">
    <div v-if="!handInProgress && !playerBroke" class="betting-input">
      <p>Place your bet ($0.50 - ${{ playerMoney }})</p>
      <input
        v-model.number="betAmount"
        type="number"
        :min="0.5"
        :max="playerMoney"
        step="0.50"
        @input="validateBet"
      />
      <button @click="placeBet">Place Bet</button>
    </div>
    <div class="action-buttons">
      <button @click="hit" :disabled="!canHit">Hit</button>
      <button @click="stand" :disabled="!canHit">Stand</button>
      <button @click="double" :disabled="!canDouble">Double</button>
      <button @click="split" :disabled="!canSplit">Split</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps<{
  playerMoney: number
  handInProgress: boolean
  playerBroke: boolean
  canHit: boolean
  canDouble: boolean
  canSplit: boolean
}>()

const emit = defineEmits([
  'placeBet',
  'hit',
  'stand',
  'double',
  'split',
  'updateBetAmount',
])

const betAmount = ref(10)

const validateBet = () => {
  const bet = betAmount.value
  const isValid =
    bet >= 0.5 && bet <= props.playerMoney && (bet * 100) % 50 === 0
  if (!isValid) {
    betAmount.value = Math.floor(bet * 2) / 2 // Reset to the nearest valid value
  }
  emit('updateBetAmount', betAmount.value)
}

const placeBet = () => emit('placeBet', betAmount.value)
const hit = () => emit('hit')
const stand = () => emit('stand')
const double = () => emit('double')
const split = () => emit('split')
</script>

<style scoped>
.betting-interface {
  margin-top: 20px;
}

.betting-input {
  margin-bottom: 10px;
}

.action-buttons {
  display: flex;
  justify-content: center;
  margin-bottom: 10px;
  flex-wrap: wrap;
}

.action-buttons button {
  padding: 10px 20px;
  font-size: 16px;
  margin: 5px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.action-buttons button:hover {
  background-color: #2980b9;
}

.action-buttons button:disabled {
  background-color: #95a5a6;
  cursor: not-allowed;
}
</style>
