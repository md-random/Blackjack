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
      <div v-if="!handInProgress && !playerBroke">
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
        <Hand
          :cards="playerHand"
          player="Player"
          :isActive="playerTurn"
          :isWinner="winner === 'player'"
        />
        <p>Player Score: {{ calculateHandValue(playerHand) }}</p>
      </template>
      <template v-else>
        <div v-for="(hand, index) in splitHands" :key="index">
          <Hand
            :cards="hand"
            :player="`Player Hand ${index + 1}`"
            :isActive="playerTurn && currentHand === index"
            :isWinner="winner === 'player'"
          />
          <p>
            Player Hand {{ index + 1 }} Score: {{ calculateHandValue(hand) }}
          </p>
          <p v-if="index === currentHand && !gameOver">
            Currently playing this hand
          </p>
        </div>
      </template>
      <Hand
        :cards="visibleDealerHand"
        player="Dealer"
        :isActive="!playerTurn"
        :isWinner="winner === 'dealer'"
      />
      <p v-if="!playerTurn || gameOver">
        Dealer Score: {{ dealerVisibleScore }}
      </p>
      <p>{{ message }}</p>
      <p>Cards left in deck: {{ deck.length }}</p>
      <button @click="hit" :disabled="!canHit">Hit</button>
      <button @click="stand" :disabled="!canHit">Stand</button>
      <button @click="double" :disabled="!canDouble">Double</button>
      <button @click="split" :disabled="!canSplit">Split</button>

      <!-- Display wins/losses -->
      <div v-if="gameOver" class="result">
        <p v-if="playerWinnings > 0" class="win">
          Player wins ${{ playerWinnings }}
        </p>
        <p v-else-if="playerWinnings < 0" class="loss">
          Player loses ${{ -playerWinnings }}
        </p>
        <p v-else>It's a tie!</p>
      </div>

      <!-- Display message and button if player is broke -->
      <div v-if="playerBroke" class="result">
        <p class="loss">You are broke!</p>
        <button @click="resetGame">Restart Game</button>
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
const splitHands = ref<Card[][]>([])
const currentHand = ref(0)
const gameStarted = ref(false)
const numberOfDecks = ref(1)
const playerMoney = ref(1000)
const currentBet = ref(0)
const betAmount = ref(10)
const handInProgress = ref(false)
const playerWinnings = ref(0)
const playerBroke = ref(false)
const winner = ref<'player' | 'dealer' | 'tie' | null>(null)

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
  if (playerMoney.value <= 0) {
    playerBroke.value = true
    return
  }
  if (betAmount.value < 1 || betAmount.value > playerMoney.value) {
    alert(`Please enter a bet between $1 and $${playerMoney.value}.`)
    return
  }
  currentBet.value = betAmount.value
  playerMoney.value -= currentBet.value
  playerWinnings.value = 0 // Reset winnings for new hand
  handInProgress.value = true
  winner.value = null // Reset the winner
  dealInitialHands()
}

const dealInitialHands = () => {
  playerHand.value = []
  dealerHand.value = []
  splitHands.value = []
  currentHand.value = 0
  message.value = ''
  gameOver.value = false
  playerTurn.value = true

  dealCard(playerHand.value)
  dealCard(dealerHand.value)
  dealCard(playerHand.value)
  dealCard(dealerHand.value)

  const dealerScore = calculateHandValue(dealerHand.value)
  const playerScore = calculateHandValue(playerHand.value)

  if (dealerScore === 21 && playerScore === 21) {
    message.value = "Both Dealer and Player have Blackjack! It's a push."
    playerMoney.value += currentBet.value // Return the bet
    playerWinnings.value = 0
    winner.value = 'tie'
    endGame()
  } else if (dealerScore === 21) {
    message.value = 'Dealer has Blackjack! Dealer wins.'
    playerWinnings.value = -currentBet.value
    winner.value = 'dealer'
    endGame()
  } else if (playerScore === 21) {
    message.value = 'Player has Blackjack!'
    const blackjackPayout = Math.floor(currentBet.value * 1.5)
    playerMoney.value += currentBet.value + blackjackPayout
    playerWinnings.value = blackjackPayout
    winner.value = 'player'
    endGame()
  }
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
      message.value = `Hand ${currentHand.value + 1} busts!`
      if (
        splitHands.value.length > 0 &&
        currentHand.value < splitHands.value.length - 1
      ) {
        nextHand()
      } else {
        playerWinnings.value -= currentBet.value
        if (splitHands.value.length > 0) {
          nextHand()
        } else {
          endGame()
        }
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
    playerMoney.value -= currentBet.value // Deduct the additional bet
    splitHands.value = [[playerHand.value[0]], [playerHand.value[1]]]
    currentHand.value = 0
    dealCard(splitHands.value[0])
    dealCard(splitHands.value[1])
    playerHand.value = splitHands.value[0]
  } else {
    alert('Cannot split!')
  }
}

const nextHand = () => {
  currentHand.value++
  playerHand.value = splitHands.value[currentHand.value]
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
    winner.value = 'dealer'
  } else if (dealerScore > 21) {
    message.value += `${handMsg}Dealer busts! Player wins. `
    winAmount = currentBet.value
    winner.value = 'player'
  } else if (dealerScore > playerScore) {
    message.value += `${handMsg}Dealer wins! `
    winAmount = -currentBet.value
    winner.value = 'dealer'
  } else if (dealerScore < playerScore) {
    message.value += `${handMsg}Player wins! `
    winAmount = currentBet.value
    winner.value = 'player'
  } else {
    message.value += `${handMsg}It's a tie! `
    winAmount = 0 // No change in money for a tie
    winner.value = 'tie'
  }

  playerWinnings.value += winAmount
  playerMoney.value += winAmount
}

const endGame = () => {
  gameOver.value = true
  playerTurn.value = false
  handInProgress.value = false
  if (playerMoney.value <= 0) {
    playerBroke.value = true
  }
}

const resetGame = () => {
  gameStarted.value = false
  playerMoney.value = 1000
  currentBet.value = 0
  betAmount.value = 10
  playerWinnings.value = 0
  playerBroke.value = false
  gameOver.value = false
  handInProgress.value = false
  message.value = ''
  playerHand.value = []
  dealerHand.value = []
  splitHands.value = []
  currentHand.value = 0
  playerTurn.value = true
  winner.value = null // Reset the winner
  initializeDeck()
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
  if (
    !playerTurn.value ||
    gameOver.value ||
    calculateHandValue(dealerHand.value) === 21 ||
    calculateHandValue(playerHand.value) === 21
  ) {
    return dealerHand.value
  }
  return dealerHand.value.slice(0, 1)
})

const dealerVisibleScore = computed(() => {
  if (!dealerHand.value || dealerHand.value.length === 0) return 0
  if (
    !playerTurn.value ||
    gameOver.value ||
    calculateHandValue(dealerHand.value) === 21 ||
    calculateHandValue(playerHand.value) === 21
  ) {
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

const canSplit = computed(() => {
  if (!playerTurn.value || gameOver.value) return false
  const currentPlayerHand =
    splitHands.value.length > 0
      ? splitHands.value[currentHand.value]
      : playerHand.value
  return (
    currentPlayerHand.length === 2 &&
    currentPlayerHand[0].rank === currentPlayerHand[1].rank &&
    splitHands.value.length === 0 &&
    playerMoney.value >= currentBet.value
  )
})
</script>
