<template>
  <div class="hand">
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
