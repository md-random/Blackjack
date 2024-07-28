<template>
  <div class="hand-log">
    <h2>Hand Log</h2>
    <ul>
      <li v-for="(log, index) in logs" :key="index">{{ log }}</li>
    </ul>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface HandLogEntry {
  bet: number
  playerHand: string
  playerScore: number
  dealerHand: string
  dealerScore: number
  result: string
  bankroll: number
  winAmount: number
}

const logs = ref<string[]>([])

const updateHandLog = (entry: HandLogEntry) => {
  const logEntry = `Bet: $${entry.bet} | Player: ${entry.playerHand} | PlScore: ${entry.playerScore} | Dealer: ${entry.dealerHand} | DeScore: ${entry.dealerScore} | ${entry.result} | Bankroll: $${entry.bankroll}`
  logs.value.unshift(logEntry)
}

const resetLog = () => {
  logs.value = []
}

defineExpose({ updateHandLog, resetLog })
</script>

<style scoped>
.hand-log {
  max-height: 300px;
  overflow-y: auto;
  background-color: rgba(0, 0, 0, 0.5);
  font-size: 12px;
}

.hand-log ul {
  list-style-type: none;
  padding: 0;
}

.hand-log li {
  margin-bottom: 5px;
}
</style>
