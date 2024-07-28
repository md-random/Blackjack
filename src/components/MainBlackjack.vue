<template>
  <div class="blackjack">
    <div class="version">Version 3.6</div>
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
            <div>
              <CardCounting
                :cardsLeft="deck.length"
                :runningCount="runningCount"
              />
            </div>

            <div class="aggression-meter">
              <div class="meter">
                <div
                  class="meter-bar"
                  :class="aggressionMeterClass"
                  :style="{ width: `${Math.abs(trueCount) * 20}%` }"
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
          </div>

          <!-- Box 6: Hand Log -->
          <!-- Box 6: Hand Log -->
          <HandLog
            ref="handLogRef"
            @logUpdated="onLogUpdated"
            @logReset="onLogReset"
          />
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
import { ref, computed, onUpdated, Ref, nextTick } from 'vue'
import Hand from './children/Hand.vue'
import HandLog from './children/HandLog.vue'
import CardCounting from './children/CardCounting.vue'

interface Card {
  suit: string
  rank: string
}

const deck = ref<Card[]>([])
const playerHand = ref<Card[]>([])
const dealerHand = ref<Card[]>([])
const dealerHiddenCard = ref<Card | null | undefined>(null)
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
const handLogRef = ref<InstanceType<typeof HandLog> | null>(null)
const handNumber = ref(0)

const trueCount = computed(() => {
  const decksRemaining = Math.ceil(deck.value.length / 52)
  return decksRemaining
    ? Math.round((runningCount.value / decksRemaining) * 100) / 100
    : 0
})

const aggressionMeter = computed(() => {
  if (trueCount.value > 0) return 'Aggressive'
  if (trueCount.value < 0) return 'Conservative'
  return 'Neutral'
})

const aggressionMeterClass = computed(() => {
  if (trueCount.value > 0) return 'aggressive'
  if (trueCount.value < 0) return 'conservative'
  return 'neutral'
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

const onLogUpdated = (logEntry: string) => {
  handLog.value.unshift(logEntry)
  // You can perform any other actions needed when the log is updated
}

const onLogReset = () => {
  handLog.value = []
  // You can perform any other actions needed when the log is reset
}

const placeBet = () => {
  if (playerMoney.value <= 0) {
    playerBroke.value = true
    return
  }

  validateBet()

  if (betAmount.value < 0.5 || betAmount.value > playerMoney.value) {
    alert(
      `Please enter a bet between $0.50 and $${playerMoney.value}, in increments of $0.50.`
    )
    return
  }

  currentBet.value = betAmount.value
  playerMoney.value -= currentBet.value
  playerWinnings.value = 0
  handInProgress.value = true
  winner.value = null
  dealInitialHands()
}

const validateBet = () => {
  const bet = betAmount.value
  const isValid =
    bet >= 0.5 && bet <= playerMoney.value && (bet * 100) % 50 === 0
  if (!isValid) {
    betAmount.value = Math.floor(bet * 2) / 2 // Reset to the nearest valid value
  }
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

  const dealerVisibleCard = dealerHand.value[0]
  const playerScore = calculateHandValue(playerHand.value)

  // Check for dealer blackjack if the visible card is an Ace or a 10-value card
  if (['A', '10', 'J', 'Q', 'K'].includes(dealerVisibleCard.rank)) {
    const dealerScore = calculateHandValue([
      ...dealerHand.value,
      ...(dealerHiddenCard.value ? [dealerHiddenCard.value] : []),
    ])
    if (dealerScore === 21) {
      revealDealerHiddenCard()
      if (playerScore === 21) {
        message.value = "Both Dealer and Player have Blackjack! It's a push."
        winLoss('tie', currentBet.value)
        winner.value = 'tie'
      } else {
        message.value = 'Dealer has Blackjack! Dealer wins.'
        winLoss('loss', currentBet.value)
        winner.value = 'dealer'
      }
      // Create hand log for this hand
      const playerHandString = playerHand.value
        .map((card) => `${card.rank}${card.suit[0]}`)
        .join(' ')
      const dealerHandString = dealerHand.value
        .concat(dealerHiddenCard.value ? [dealerHiddenCard.value] : [])
        .map((card) => `${card.rank}${card.suit[0]}`)
        .join(' ')
      handLogRef.value?.updateHandLog({
        bet: currentBet.value,
        playerHand: playerHandString,
        playerScore,
        dealerHand: dealerHandString,
        dealerScore,
        result: winner.value === 'tie' ? 'Tie' : 'Player loses',
        bankroll: playerMoney.value,
        winAmount: winner.value === 'tie' ? 0 : -currentBet.value,
      })
      endGame()
      return
    }
  }

  // Check for player blackjack
  if (playerScore === 21) {
    revealDealerHiddenCard()
    message.value = 'Player has Blackjack!'
    const blackjackPayout = currentBet.value * 1.5
    const winAmount = winLoss('blackjack', currentBet.value)
    winner.value = 'player'
    // Create hand log for this hand
    const playerHandString = playerHand.value
      .map((card) => `${card.rank}${card.suit[0]}`)
      .join(' ')
    const dealerHandString = dealerHand.value
      .concat(dealerHiddenCard.value ? [dealerHiddenCard.value] : [])
      .map((card) => `${card.rank}${card.suit[0]}`)
      .join(' ')
    handLogRef.value?.updateHandLog({
      bet: currentBet.value,
      playerHand: playerHandString,
      playerScore,
      dealerHand: dealerHandString,
      dealerScore: calculateHandValue(dealerHand.value),
      result: 'Player wins',
      bankroll: playerMoney.value,
      winAmount,
    })
    endGame()
  }
}

const winLoss = (result: 'win' | 'loss' | 'tie' | 'blackjack', bet: number) => {
  let winAmount = 0

  switch (result) {
    case 'win':
      winAmount = bet
      playerMoney.value += bet * 2 // Player gets back their bet plus the winnings
      break
    case 'loss':
      winAmount = -bet
      // Player already lost the bet amount when placing the bet
      break
    case 'tie':
      winAmount = 0
      playerMoney.value += bet // Player gets back their bet
      break
    case 'blackjack':
      winAmount = bet * 1.5
      playerMoney.value += bet + winAmount // Player gets back their bet plus 1.5x winnings
      break
  }

  playerWinnings.value += winAmount
  return winAmount
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
          stand() // Changed from endGame() to stand()
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
  if (currentHand.value < splitHands.value.length) {
    playerHand.value = splitHands.value[currentHand.value]
  } else {
    stand()
  }
}

const compareHands = (
  playerScore: number,
  dealerScore: number,
  handIndex?: number
) => {
  let result = ''
  let winAmount = 0
  const originalBet = currentBet.value

  if (playerScore > 21) {
    result = 'loses'
    winAmount = winLoss('loss', currentBet.value)
  } else if (dealerScore > 21) {
    result = 'wins'
    winAmount = winLoss('win', currentBet.value)
  } else if (dealerScore > playerScore) {
    result = 'loses'
    winAmount = winLoss('loss', currentBet.value)
  } else if (dealerScore < playerScore) {
    result = 'wins'
    winAmount = winLoss('win', currentBet.value)
  } else {
    result = 'Tie'
    winAmount = winLoss('tie', currentBet.value)
  }

  const playerHandString =
    handIndex !== undefined
      ? splitHands.value[handIndex]
          ?.map((card) => `${card.rank}${card.suit[0]}`)
          .join(' ')
      : playerHand.value.map((card) => `${card.rank}${card.suit[0]}`).join(' ')
  const dealerHandString = dealerHand.value
    .concat(dealerHiddenCard.value ? [dealerHiddenCard.value] : [])
    .map((card) => `${card.rank}${card.suit[0]}`)
    .join(' ')

  handLogRef.value?.updateHandLog({
    handNumber: 0, // This will be managed inside HandLog component
    bet: originalBet,
    playerHand: playerHandString,
    playerScore,
    dealerHand: dealerHandString,
    dealerScore,
    result,
    bankroll: Number(playerMoney.value),
    winAmount,
  })

  if (handIndex === undefined || handIndex === splitHands.value.length - 1) {
    endGame()
  }
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
  handLogRef.value?.resetLog()
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
    const winAmount = winLoss('blackjack', currentBet.value)
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

.count-info,
.game-info,
.hand-log {
  flex: 1;
  max-width: 40%;
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
</style>
