<template>
  <div
    class="hand"
    :class="{
      'active-player': isActive && !isWinner,
      'inactive-player': !isActive && !isWinner,
      winner: isWinner,
    }"
  >
    <h2>{{ player }}'s Hand</h2>
    <div v-for="(card, index) in visibleCards" :key="index" class="card">
      {{ card.rank }} of {{ card.suit }}
    </div>
    <div v-if="hiddenCard" class="card hidden">Hidden Card</div>
  </div>
</template>

<script setup lang="ts">
import { defineProps, computed } from 'vue'

interface Card {
  suit: string
  rank: string
}

interface HandProps {
  cards: Card[]
  player: string
  hideSecondCard?: boolean
  isActive: boolean
  isWinner: boolean
}

const props = defineProps<HandProps>()

const visibleCards = computed(() => {
  if (props.hideSecondCard && props.cards.length > 1) {
    return [props.cards[0]]
  }
  return props.cards
})

const hiddenCard = computed(() => {
  return props.hideSecondCard && props.cards.length > 1
})
</script>

<style scoped>
.hand {
  margin-bottom: 20px;
  padding: 10px;
  border-radius: 25px; /* Pill shape */
  transition: all 0.3s ease;
  border: 3px solid transparent;
}

.active-player {
  border-color: red;
}

.inactive-player {
  border-color: blue;
}

.winner {
  border-color: gold;
  animation: pulse-gold 2s infinite;
}

@keyframes pulse-gold {
  0% {
    box-shadow: 0 0 0 0 rgba(255, 215, 0, 0.7);
  }
  70% {
    box-shadow: 0 0 0 10px rgba(255, 215, 0, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(255, 215, 0, 0);
  }
}

.card {
  border: 1px solid black;
  padding: 10px;
  margin: 5px;
  display: inline-block;
}

.hidden {
  background-color: #ddd;
}
</style>
