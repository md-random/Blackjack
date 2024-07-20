<template>
  <div class="blackjack">
    <h1>Blackjack</h1>
    <div v-if="!gameStarted">
      <p>How many decks would you like to play with? (1-6)</p>
      <input v-model.number="numberOfDecks" type="number" min="1" max="6" />
      <p>How much money would you like to play with? ($100 - $100,000)</p>
      <input
        v-model.number="playerMoney"
        type="number"
        min="100"
        max="100000"
      />
      <button @click="startGame">Start Game</button>
    </div>
    <div v-else>
      <p>Player Money: ${{ playerMoney }}</p>
      <p>Current Bet: ${{ currentBet }}</p>
      <div v-if="!handInProgress || gameOver">
        <p>Place your bet ($1 - ${{ playerMoney }})</p>
        <input
          v-model.number="betAmount"
          type="number"
          :min="1"
          :max="playerMoney"
        />
        <button @click="placeBet">Place Bet</button>
      </div>
      <template v-if="splitHands.length === 0">
        <Hand :cards="playerHand" player="Player" />
        <p>Player Score: {{ calculateHandValue(playerHand) }}</p>
      </template>
      <template v-else>
        <div v-for="(hand, index) in splitHands" :key="index">
          <Hand :cards="hand" :player="`Player Hand ${index + 1}`" />
          <p>
            Player Hand {{ index + 1 }} Score: {{ calculateHandValue(hand) }}
          </p>
          <p v-if="index === currentHand && !gameOver">
            Currently playing this hand
          </p>
        </div>
      </template>
      <Hand :cards="visibleDealerHand" player="Dealer" />
      <p>Dealer Score: {{ dealerVisibleScore }}</p>
      <p>{{ message }}</p>
      <p>Cards left in deck: {{ deck.length }}</p>
      <button @click="hit" :disabled="!canHit">Hit</button>
      <button @click="stand" :disabled="!canHit">Stand</button>
      <button @click="double" :disabled="!canDouble">Double</button>
      <button @click="split" :disabled="!canSplit">Split</button>

      <!-- New section to display wins/losses -->
      <div v-if="gameOver" class="result">
        <p v-if="playerWinnings > 0" class="win">
          Player wins ${{ playerWinnings }}
        </p>
        <p v-else-if="playerWinnings < 0" class="loss">
          Player loses ${{ -playerWinnings }}
        </p>
        <p v-else>It's a tie!</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import Hand from './children/Hand.vue'

interface Card {
  suit: string
  rank: string
}

const deck = ref<Card[]>([])
const playerHand = ref<Card[]>([])
const dealerHand = ref<Card[]>([])
const message = ref('')
const gameOver = ref(false)
const playerTurn = ref(true)
const canSplit = ref(false)
const splitHands = ref<Card[][]>([])
const currentHand = ref(0)
const dealerCheckedForBlackjack = ref(false)
const gameStarted = ref(false)
const numberOfDecks = ref(1)
const playerMoney = ref(1000)
const currentBet = ref(0)
const betAmount = ref(10)
const handInProgress = ref(false)
const playerWinnings = ref(0)

const suits = ['Hearts', 'Diamonds', 'Clubs', 'Spades']
const ranks = ['2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K', 'A']

const initializeDeck = () => {
  deck.value = []
  for (let i = 0; i < numberOfDecks.value; i++) {
    for (const suit of suits) {
      for (const rank of ranks) {
        deck.value.push({ suit, rank })
      }
    }
  }
  shuffleDeck()
}

const shuffleDeck = () => {
  for (let i = deck.value.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[deck.value[i], deck.value[j]] = [deck.value[j], deck.value[i]]
  }
}

const dealCard = (hand: Card[]) => {
  if (deck.value.length === 0) {
    initializeDeck()
  }
  const card = deck.value.pop()
  if (card) hand.push(card)
}

const startGame = () => {
  if (numberOfDecks.value < 1 || numberOfDecks.value > 6) {
    alert('Please choose between 1 and 6 decks.')
    return
  }
  if (playerMoney.value < 100 || playerMoney.value > 100000) {
    alert('Please enter an amount between $100 and $100,000.')
    return
  }
  gameStarted.value = true
  initializeDeck()
}

const placeBet = () => {
  if (betAmount.value < 1 || betAmount.value > playerMoney.value) {
    alert(`Please enter a bet between $1 and $${playerMoney.value}.`)
    return
  }
  currentBet.value = betAmount.value
  playerMoney.value -= currentBet.value
  handInProgress.value = true
  gameOver.value = false
  playerWinnings.value = 0 // Reset winnings for new hand
  dealInitialHands()
}

const dealInitialHands = () => {
  playerHand.value = []
  dealerHand.value = []
  splitHands.value = []
  currentHand.value = 0
  dealerCheckedForBlackjack.value = false
  message.value = ''
  gameOver.value = false
  playerTurn.value = true

  dealCard(playerHand.value)
  dealCard(dealerHand.value)
  dealCard(playerHand.value)
  dealCard(dealerHand.value)

  updateActions()

  if (isDealerBlackjack()) {
    message.value = 'Dealer has Blackjack! Dealer wins.'
    endGame()
  } else if (calculateHandValue(playerHand.value) === 21) {
    message.value = 'Player has Blackjack!'
    playerMoney.value += Math.floor(currentBet.value * 2.5)
    endGame()
  }
}

const isDealerBlackjack = () => {
  if (dealerHand.value.length < 2) return false

  const dealerFirstCard = dealerHand.value[0]
  const dealerSecondCard = dealerHand.value[1]
  const dealerFirstCardValue =
    dealerFirstCard.rank === 'A'
      ? 11
      : ['K', 'Q', 'J', '10'].includes(dealerFirstCard.rank)
      ? 10
      : parseInt(dealerFirstCard.rank)
  const dealerSecondCardValue =
    dealerSecondCard.rank === 'A'
      ? 11
      : ['K', 'Q', 'J', '10'].includes(dealerSecondCard.rank)
      ? 10
      : parseInt(dealerSecondCard.rank)

  if (
    dealerFirstCard.rank === 'A' ||
    ['K', 'Q', 'J', '10'].includes(dealerFirstCard.rank)
  ) {
    if (dealerFirstCardValue + dealerSecondCardValue === 21) {
      dealerCheckedForBlackjack.value = true
      return true
    }
  }
  return false
}

const hit = () => {
  if (playerTurn.value && !gameOver.value) {
    const currentPlayerHand =
      splitHands.value.length > 0
        ? splitHands.value[currentHand.value]
        : playerHand.value
    dealCard(currentPlayerHand)
    const playerScore = calculateHandValue(currentPlayerHand)

    if (playerScore > 21) {
      if (
        splitHands.value.length > 0 &&
        currentHand.value < splitHands.value.length - 1
      ) {
        message.value = `Hand ${currentHand.value + 1} busts!`
        nextHand()
      } else {
        message.value =
          splitHands.value.length > 0
            ? 'All hands bust! Dealer wins.'
            : 'Player busts! Dealer wins.'
        endGame()
      }
    } else if (playerScore === 21) {
      if (
        splitHands.value.length > 0 &&
        currentHand.value < splitHands.value.length - 1
      ) {
        message.value = `Hand ${currentHand.value + 1} has 21!`
        nextHand()
      } else {
        stand()
      }
    }
    updateActions()
  }
}

const stand = () => {
  if (
    splitHands.value.length > 0 &&
    currentHand.value < splitHands.value.length - 1
  ) {
    nextHand()
  } else {
    playerTurn.value = false
    dealerTurn()
  }
}

const dealerTurn = () => {
  while (calculateHandValue(dealerHand.value) < 17) {
    dealCard(dealerHand.value)
  }
  const dealerScore = calculateHandValue(dealerHand.value)

  if (splitHands.value.length > 0) {
    splitHands.value.forEach((hand, index) => {
      const playerScore = calculateHandValue(hand)
      compareHands(playerScore, dealerScore, index + 1)
    })
  } else {
    const playerScore = calculateHandValue(playerHand.value)
    compareHands(playerScore, dealerScore)
  }
  endGame()
}

const double = () => {
  if (canDouble.value) {
    playerMoney.value -= currentBet.value
    currentBet.value *= 2
    hit()
    if (!gameOver.value) {
      stand()
    }
  } else {
    alert('Cannot double down!')
  }
}

const split = () => {
  if (canSplit.value) {
    playerMoney.value -= currentBet.value
    splitHands.value = [[playerHand.value[0]], [playerHand.value[1]]]
    currentHand.value = 0
    dealCard(splitHands.value[0])
    dealCard(splitHands.value[1])
    playerHand.value = splitHands.value[0]
    updateActions()
  } else {
    alert('Cannot split!')
  }
}

const nextHand = () => {
  currentHand.value++
  playerHand.value = splitHands.value[currentHand.value]
  updateActions()
}

const compareHands = (
  playerScore: number,
  dealerScore: number,
  handNumber?: number
) => {
  const handMsg = handNumber ? `Hand ${handNumber}: ` : ''
  let winAmount = 0

  if (playerScore > 21) {
    message.value += `${handMsg}Player busts. Dealer wins. `
    winAmount = -currentBet.value
  } else if (dealerScore > 21) {
    message.value += `${handMsg}Dealer busts! Player wins. `
    winAmount = currentBet.value
  } else if (dealerScore > playerScore) {
    message.value += `${handMsg}Dealer wins! `
    winAmount = -currentBet.value
  } else if (dealerScore < playerScore) {
    message.value += `${handMsg}Player wins! `
    winAmount = currentBet.value
  } else {
    message.value += `${handMsg}It's a tie! `
    winAmount = 0
  }

  playerMoney.value += winAmount + currentBet.value
  playerWinnings.value += winAmount
}

const updateActions = () => {
  const currentPlayerHand =
    splitHands.value.length > 0
      ? splitHands.value[currentHand.value]
      : playerHand.value
  canSplit.value =
    currentPlayerHand.length === 2 &&
    currentPlayerHand[0].rank === currentPlayerHand[1].rank &&
    splitHands.value.length === 0 &&
    playerMoney.value >= currentBet.value
}

const endGame = () => {
  gameOver.value = true
  playerTurn.value = false
  handInProgress.value = false
}

const calculateHandValue = (hand: Card[]) => {
  if (!hand || hand.length === 0) return 0

  let value = 0
  let aces = 0
  for (const card of hand) {
    if (card.rank === 'A') {
      aces++
      value += 11
    } else if (['K', 'Q', 'J'].includes(card.rank)) {
      value += 10
    } else {
      value += parseInt(card.rank)
    }
  }
  while (value > 21 && aces > 0) {
    value -= 10
    aces--
  }
  return value
}

const visibleDealerHand = computed(() => {
  if (!playerTurn.value || gameOver.value || dealerCheckedForBlackjack.value) {
    return dealerHand.value
  }
  return dealerHand.value.slice(0, 1)
})

const dealerVisibleScore = computed(() => {
  if (!dealerHand.value || dealerHand.value.length === 0) return 0
  if (!playerTurn.value || gameOver.value || dealerCheckedForBlackjack.value) {
    return calculateHandValue(dealerHand.value)
  }
  return calculateHandValue([dealerHand.value[0]])
})

const canHit = computed(() => {
  if (!playerTurn.value || gameOver.value) return false
  const currentPlayerHand =
    splitHands.value.length > 0
      ? splitHands.value[currentHand.value]
      : playerHand.value
  return calculateHandValue(currentPlayerHand) < 21
})

const canDouble = computed(() => {
  if (!playerTurn.value || gameOver.value) return false
  const currentPlayerHand =
    splitHands.value.length > 0
      ? splitHands.value[currentHand.value]
      : playerHand.value
  return currentPlayerHand.length === 2 && playerMoney.value >= currentBet.value
})
</script>
