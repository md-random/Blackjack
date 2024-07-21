<template>
  <div
    class="hand"
    :class="{
      'active-player': isActive,
      'inactive-player': !isActive,
      winner: isWinner,
    }"
  >
    <h2>{{ player }}</h2>
    <div class="cards-container">
      <div
        v-for="(card, index) in visibleCards"
        :key="index"
        class="card"
        :class="[card.suit.toLowerCase()]"
        :data-value="card.rank"
      >
        {{ getSuitSymbol(card.suit) }}
      </div>
      <div v-if="hiddenCard" class="card hidden">
        <!-- You can add a back design here -->
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'

interface Card {
  suit: string
  rank: string
}

const props = defineProps<{
  cards: Card[]
  player: string
  hideSecondCard?: boolean
  isActive: boolean
  isWinner: boolean
}>()

const visibleCards = computed(() => {
  if (props.hideSecondCard && props.cards.length > 1) {
    return [props.cards[0]]
  }
  return props.cards
})

const hiddenCard = computed(() => {
  return props.hideSecondCard && props.cards.length > 1
})

const getSuitSymbol = (suit: string) => {
  switch (suit.toLowerCase()) {
    case 'hearts':
      return '♥'
    case 'diamonds':
      return '♦'
    case 'clubs':
      return '♣'
    case 'spades':
      return '♠'
    default:
      return ''
  }
}
</script>

<style scoped>
.hand {
  width: 100%;
  padding: 10px;
  margin-bottom: 10px;
}

.cards-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px;
}

.card {
  width: 100px;
  height: 140px;
  background-color: white;
  border-radius: 10px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: 'Arial', sans-serif;
  font-size: 48px; /* Increased size for suit */
  position: relative;
}

.card::before,
.card::after {
  content: attr(data-value);
  position: absolute;
  font-size: 24px; /* Increased size for rank */
  font-weight: bold;
}

.card::before {
  top: 5px;
  left: 5px;
}

.card::after {
  bottom: 5px;
  right: 5px;
  transform: rotate(180deg);
}

.card.hearts,
.card.diamonds {
  color: red;
}

.card.spades,
.card.clubs {
  color: black;
}

.card.hidden {
  background-color: #b0b0b0;
  background-image: repeating-linear-gradient(
    45deg,
    #606dbc,
    #606dbc 10px,
    #465298 10px,
    #465298 20px
  );
}

/* Responsive card sizing */
@media (max-width: 1200px) {
  .card {
    width: 90px;
    height: 126px;
    font-size: 42px;
  }
  .card::before,
  .card::after {
    font-size: 22px;
  }
}

@media (max-width: 992px) {
  .card {
    width: 80px;
    height: 112px;
    font-size: 36px;
  }
  .card::before,
  .card::after {
    font-size: 20px;
  }
}

@media (max-width: 768px) {
  .card {
    width: 70px;
    height: 98px;
    font-size: 32px;
  }
  .card::before,
  .card::after {
    font-size: 18px;
  }
}

@media (max-width: 576px) {
  .card {
    width: 60px;
    height: 84px;
    font-size: 28px;
  }
  .card::before,
  .card::after {
    font-size: 16px;
  }
}
</style>
