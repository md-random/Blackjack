<template>
  <div class="blackjack">
    <div class="version">Version 3.0</div>
    <div class="game-container">
      <div v-if="!gameStarted" class="game-setup">
        <h2>Game Setup</h2>
        <div>
          <label for="numDecks">Number of Decks (1-6): </label>
          <input
            id="numDecks"
            v-model.number="numberOfDecks"
            type="number"
            min="1"
            max="6"
          />
        </div>
        <div>
          <label for="startingBankroll"
            >Starting Bankroll ($100-$100,000):
          </label>
          <input
            id="startingBankroll"
            v-model.number="startingBankroll"
            type="number"
            min="100"
            max="100000"
          />
        </div>
        <button @click="startGame">Start Game</button>
      </div>

      <template v-else>
        <div class="top-row">
          <!-- Box 5: Count Info -->
          <div class="box count-info">
            <div class="count-info-text">
              <p>Cards left in deck: {{ deck.length }}</p>
              <p>Running Count: {{ runningCount }}</p>
              <p>True Count: {{ trueCount }}</p>
              <p>Play Style: {{ aggressionMeter }}</p>
            </div>

            <div class="aggression-meter">
              <div class="meter">
                <div
                  class="meter-bar"
                  :class="aggressionMeter.toLowerCase()"
                  :style="{ width: `${(trueCount + 2) * 20}%` }"
                ></div>
              </div>
            </div>
          </div>

          <!-- Box 1: Game Info and Betting -->
          <div class="box game-info">
            <h1>Blackjack</h1>
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
          </div>

          <!-- Box 6: Hand Log -->
          <div class="box hand-log">
            <h2>Hand Log</h2>
            <ul>
              <li v-for="(log, index) in handLog" :key="index">{{ log }}</li>
            </ul>
          </div>
        </div>

        <div class="middle-row">
          <!-- Box 2: Player's Hand -->
          <div class="box player-hand">
            <template v-if="splitHands.length === 0">
              <Hand
                :cards="playerHand"
                :player="`Player's Hand: ${playerScore}`"
                :isActive="playerTurn"
                :isWinner="winner === 'player'"
              />
            </template>
            <template v-else>
              <div v-for="(hand, index) in splitHands" :key="index">
                <Hand
                  :cards="hand"
                  :player="`Player's Hand ${index + 1}: ${calculateHandValue(
                    hand
                  )}`"
                  :isActive="playerTurn && currentHand === index"
                  :isWinner="winner === 'player'"
                />
                <p v-if="index === currentHand && !gameOver">
                  Currently playing this hand
                </p>
              </div>
            </template>
          </div>

          <!-- Box 3: Dealer's Hand -->
          <div class="box dealer-hand">
            <Hand
              :cards="visibleDealerHand"
              :player="`Dealer's Hand: ${dealerVisibleScore}`"
              :isActive="!playerTurn"
              :isWinner="winner === 'dealer'"
            />
          </div>
        </div>

        <!-- Box 4: Game Actions and Messages -->
        <div class="box game-actions">
          <p>{{ message }}</p>
          <div class="action-buttons">
            <button @click="hit" :disabled="!canHit">Hit</button>
            <button @click="stand" :disabled="!canHit">Stand</button>
            <button @click="double" :disabled="!canDouble">Double</button>
            <button @click="split" :disabled="!canSplit">Split</button>
          </div>

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
      </template>
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
const dealerHiddenCard = ref<Card | null>(null)
const message = ref('')
const gameOver = ref(false)
const playerTurn = ref(true)
const splitHands = ref<Card[][]>([])
const currentHand = ref(0)
const gameStarted = ref(false)
const numberOfDecks = ref(1)
const startingBankroll = ref(1000)
const playerMoney = ref(startingBankroll.value)
const currentBet = ref(0)
const betAmount = ref(10)
const handInProgress = ref(false)
const playerWinnings = ref(0)
const playerBroke = ref(false)
const winner = ref<'player' | 'dealer' | 'tie' | null>(null)
const runningCount = ref(0)
const handLog = ref<string[]>([])

const trueCount = computed(() => {
  const decksRemaining = Math.ceil(deck.value.length / 52)
  return decksRemaining
    ? Math.round((runningCount.value / decksRemaining) * 100) / 100
    : 0
})

const aggressionMeter = computed(() => {
  if (trueCount.value <= -2) return 'Aggressive'
  if (trueCount.value >= 1) return 'Conservative'
  return 'Neutral'
})

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
  runningCount.value = 0
}

const shuffleDeck = () => {
  for (let i = deck.value.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[deck.value[i], deck.value[j]] = [deck.value[j], deck.value[i]]
  }
}

const dealCard = (hand: Card[], isDealerHidden = false) => {
  if (deck.value.length === 0) {
    initializeDeck()
  }
  const card = deck.value.pop()
  if (card) {
    hand.push(card)
    if (!isDealerHidden) {
      updateRunningCount(card)
    }
  }
}

const updateRunningCount = (card: Card) => {
  if (['2', '3', '4', '5', '6'].includes(card.rank)) {
    runningCount.value += 1
  } else if (['10', 'J', 'Q', 'K', 'A'].includes(card.rank)) {
    runningCount.value -= 1
  }
}

const revealDealerHiddenCard = () => {
  if (dealerHiddenCard.value) {
    dealerHand.value.push(dealerHiddenCard.value)
    updateRunningCount(dealerHiddenCard.value)
    dealerHiddenCard.value = null
  }
}

const startGame = () => {
  if (numberOfDecks.value < 1 || numberOfDecks.value > 6) {
    alert('Please choose between 1 and 6 decks.')
    return
  }
  if (startingBankroll.value < 100 || startingBankroll.value > 100000) {
    alert('Please enter a starting bankroll between $100 and $100,000.')
    return
  }
  playerMoney.value = startingBankroll.value
  gameStarted.value = true
  initializeDeck()
}

const updateHandLog = (
  bet: number,
  playerHand: string,
  playerSc: number,
  dealerHand: string,
  dealerSc: number,
  result: string,
  bankroll: number
) => {
  const resultText =
    result === 'Player wins'
      ? 'Player wins'
      : result === 'Player loses'
      ? 'Player loses'
      : 'Tie'
  const displayBet = currentBet.value > bet ? bet * 2 : bet // Display doubled bet if applicable
  handLog.value.push(
    `Bet: $${displayBet} | Player: ${playerHand} | PlScore: ${playerSc} | Dealer: ${dealerHand} | DeScore: ${dealerSc} | ${resultText} | Bankroll: $${bankroll}`
  )
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
  playerWinnings.value = 0
  handInProgress.value = true
  winner.value = null
  dealInitialHands()
}

const dealInitialHands = () => {
  playerHand.value = []
  dealerHand.value = []
  dealerHiddenCard.value = null
  splitHands.value = []
  currentHand.value = 0
  message.value = ''
  gameOver.value = false
  playerTurn.value = true

  dealCard(playerHand.value)
  dealCard(dealerHand.value)
  dealCard(playerHand.value)
  dealCard(dealerHand.value, true)

  dealerHiddenCard.value = dealerHand.value.pop()

  const dealerScore = calculateHandValue(dealerHand.value)
  const playerScore = calculateHandValue(playerHand.value)

  if (dealerScore === 21 && playerScore === 21) {
    revealDealerHiddenCard()
    message.value = "Both Dealer and Player have Blackjack! It's a push."
    playerMoney.value += currentBet.value

    playerWinnings.value = 0
    winner.value = 'tie'
    endGame()
  } else if (dealerScore === 21) {
    revealDealerHiddenCard()
    message.value = 'Dealer has Blackjack! Dealer wins.'
    playerWinnings.value -= currentBet.value
    winner.value = 'dealer'
    endGame()
  } else if (playerScore === 21) {
    revealDealerHiddenCard()
    message.value = 'Player has Blackjack!'
    const blackjackPayout = Math.floor(currentBet.value * 1.5)
    playerMoney.value += currentBet.value + blackjackPayout
    playerWinnings.value = Number(currentBet.value) + blackjackPayout
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
    if (
      splitHands.value.length > 0 &&
      checkForBlackjackAfterSplit(currentPlayerHand)
    ) {
      if (currentHand.value < splitHands.value.length - 1) {
        nextHand()
      } else {
        stand()
      }
    } else if (playerScore > 21) {
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
    revealDealerHiddenCard()
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
    playerMoney.value -= currentBet.value // Deduct the additional bet
    currentBet.value *= 2 // Double the current bet
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
  } else {
    alert('Cannot split!')
  }
}

const nextHand = () => {
  currentHand.value++
  playerHand.value = splitHands.value[currentHand.value]
}

const compareHands = (playerScore: number, dealerScore: number) => {
  let result = ''
  let winAmount = 0
  console.log('currentBet.value', currentBet.value)
  const originalBet = currentBet.value // Store the original bet amount
  console.log('originalBet', originalBet)
  if (playerScore > 21) {
    result = 'Player loses'
    winAmount -= currentBet.value
  } else if (dealerScore > 21) {
    console.log('Player wins dealer Bust')
    result = 'Player wins'
    winAmount += currentBet.value * 2
  } else if (dealerScore > playerScore) {
    result = 'Player loses'
    //winAmount -= currentBet.value
  } else if (dealerScore < playerScore) {
    console.log('Player wins Draw')
    result = 'Player wins'
    winAmount += currentBet.value * 2
  } else {
    console.log('Tie')
    result = 'Tie'
    winAmount += currentBet.value
  }

  playerMoney.value += winAmount

  const playerHandString = playerHand.value
    .map((card) => `${card.rank}${card.suit[0]}`)
    .join(' ')
  const dealerHandString = dealerHand.value
    .concat(dealerHiddenCard.value ? [dealerHiddenCard.value] : [])
    .map((card) => `${card.rank}${card.suit[0]}`)
    .join(' ')

  updateHandLog(
    originalBet, // Use the original bet amount for logging
    playerHandString,
    playerScore,
    dealerHandString,
    dealerScore,
    result,
    Number(playerMoney.value)
  )

  playerWinnings.value = winAmount
  endGame()
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
  playerMoney.value = startingBankroll.value
  currentBet.value = 0
  betAmount.value = 10
  playerWinnings.value = 0
  playerBroke.value = false
  gameOver.value = false
  handInProgress.value = false
  message.value = ''
  playerHand.value = []
  dealerHand.value = []
  dealerHiddenCard.value = null
  splitHands.value = []
  currentHand.value = 0
  playerTurn.value = true
  winner.value = null
  handLog.value = []
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

const checkForBlackjackAfterSplit = (hand: Card[]) => {
  if (hand.length === 2 && calculateHandValue(hand) === 21) {
    message.value += `Hand ${currentHand.value + 1} has Blackjack! `
    const blackjackPayout = Math.floor(currentBet.value * 1.5)
    playerMoney.value += currentBet.value + blackjackPayout
    playerWinnings.value += blackjackPayout
    return true
  }
  return false
}

const visibleDealerHand = computed(() => {
  if (
    !playerTurn.value ||
    gameOver.value ||
    calculateHandValue(dealerHand.value) === 21 ||
    calculateHandValue(playerHand.value) === 21
  ) {
    return dealerHand.value.concat(
      dealerHiddenCard.value ? [dealerHiddenCard.value] : []
    )
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
    return calculateHandValue(
      dealerHand.value.concat(
        dealerHiddenCard.value ? [dealerHiddenCard.value] : []
      )
    )
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

const playerScore = computed(() => calculateHandValue(playerHand.value))
const dealerScore = computed(() =>
  calculateHandValue(
    dealerHand.value.concat(
      dealerHiddenCard.value ? [dealerHiddenCard.value] : []
    )
  )
)

// Initialize the deck when the component is created
initializeDeck()
</script>

<style scoped>
.blackjack {
  font-family: Arial, sans-serif;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  background-color: #2c7a2c; /* Green felt background */
  color: #ffffff;
}

.version {
  position: absolute;
  top: 10px;
  right: 10px;
  font-size: 14px;
  color: #4169e1; /* Royal Blue */
  font-weight: bold;
}

.game-container {
  display: flex;
  flex-direction: column;
  width: 100%;
  height: 100%;
}

.top-row {
  display: flex;
  justify-content: space-between;
  width: 100%;
}

.middle-row {
  display: flex;
  justify-content: space-around;
  width: 100%;
  margin: 20px 0;
}

.box {
  padding: 20px;
  margin: 10px;
  border-radius: 10px;
}
.count-info {
  text-align: start;
}

.count-info,
.game-info,
.hand-log {
  flex: 1;
  max-width: 30%;
}

.player-hand,
.dealer-hand {
  flex: 1;
  display: flex;
  justify-content: center;
}

.game-actions {
  width: 100%;
  text-align: center;
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

.result {
  font-weight: bold;
  margin-top: 10px;
}

.win {
  color: #2ecc71;
}

.loss {
  color: #e74c3c;
}

.hand-log {
  max-height: 300px;
  overflow-y: auto;
  background-color: rgba(0, 0, 0, 0.5);
}

.hand-log ul {
  list-style-type: none;
  padding: 0;
}

.hand-log li {
  margin-bottom: 5px;
}

@media (max-width: 1024px) {
  .top-row,
  .middle-row {
    flex-direction: column;
    align-items: center;
  }

  .count-info,
  .game-info,
  .hand-log {
    max-width: 90%;
  }
}

.count-info-text {
  font-size: 18px;
  padding: 40px;
}

@media (max-width: 576px) {
  .box {
    padding: 10px;
  }

  .action-buttons button {
    padding: 8px 16px;
    font-size: 14px;
  }
}

.game-setup {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100vh;
}

.game-setup h2 {
  margin-bottom: 20px;
}

.game-setup div {
  margin-bottom: 10px;
}

.game-setup input {
  margin-left: 10px;
}

.game-setup button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.game-setup button:hover {
  background-color: #2980b9;
}

.meter {
  width: 100%;
  height: 20px;
  background-color: #e0e0e0;
  border-radius: 10px;
  overflow: hidden;
}

.meter-bar {
  height: 100%;
  transition: width 0.5s ease-in-out;
}

.meter-bar.aggressive {
  background-color: #e74c3c;
}

.meter-bar.neutral {
  background-color: #f1c40f;
}

.meter-bar.conservative {
  background-color: #2ecc71;
}
</style>
