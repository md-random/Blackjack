<template>
  <div class="blackjack">
    <div class="version">Version 4.0</div>
    <div class="grid-container">
      <div class="grid-item grid-item-1"></div>
      <div class="grid-item grid-item-2">
        <Hand
          :cards="visibleDealerHand"
          :player="`Dealer: ${dealerVisibleScore}`"
          :isActive="!playerTurn"
          :isWinner="winner === 'dealer'"
          class="hand dealer-hand"
        />
      </div>
      <div class="grid-item grid-item-3"></div>
      <div class="grid-item grid-item-4">
        <CardCounting :cardsLeft="deck.length" :runningCount="runningCount" />
      </div>
      <div class="grid-item grid-item-5">
        <div class="game-info">
          <h2>Blackjack</h2>
          <p>Money: ${{ playerMoney }}</p>
          <p>Bet: ${{ currentBet }}</p>
        </div>
        <BettingInterface
          :playerMoney="playerMoney"
          :handInProgress="handInProgress"
          :playerBroke="playerBroke"
          :canHit="canHit"
          :canDouble="canDouble"
          :canSplit="canSplit"
          @placeBet="placeBet"
          @hit="hit"
          @stand="stand"
          @double="double"
          @split="split"
          @updateBetAmount="updateBetAmount"
        />
        <p class="message">{{ message }}</p>
      </div>
      <div class="grid-item grid-item-6">
        <HandLog :logs="handLog" />
      </div>
      <div class="grid-item grid-item-7"></div>
      <div class="grid-item grid-item-8">
        <Hand
          :cards="playerHand"
          :player="`Player: ${playerScore}`"
          :isActive="playerTurn"
          :isWinner="winner === 'player'"
          class="hand player-hand"
        />
      </div>
      <div class="grid-item grid-item-9"></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onUpdated, Ref, nextTick } from 'vue'
import Hand from './children/Hand.vue'
import HandLog from './children/HandLog.vue'
import CardCounting from './children/CardCounting.vue'
import BettingInterface from './children/BettingInterface.vue'

interface Card {
  suit: string
  rank: string
}

interface HandLogEntry {
  handNumber: number
  bet: number
  playerHand: string
  playerScore: number
  dealerHand: string
  dealerScore: number
  result: string
  bankroll: number
  winAmount: number
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

const updateBetAmount = (amount: number) => {
  betAmount.value = amount
}

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
      // Increment handNumber and create hand log for this hand
      handNumber.value++
      const playerHandString = playerHand.value
        .map((card) => `${card.rank}${card.suit[0]}`)
        .join(' ')
      const dealerHandString = dealerHand.value
        .concat(dealerHiddenCard.value ? [dealerHiddenCard.value] : [])
        .map((card) => `${card.rank}${card.suit[0]}`)
        .join(' ')
      updateHandLog({
        handNumber: handNumber.value,
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
    // Increment handNumber and create hand log for this hand
    handNumber.value++
    const playerHandString = playerHand.value
      .map((card) => `${card.rank}${card.suit[0]}`)
      .join(' ')
    const dealerHandString = dealerHand.value
      .concat(dealerHiddenCard.value ? [dealerHiddenCard.value] : [])
      .map((card) => `${card.rank}${card.suit[0]}`)
      .join(' ')
    updateHandLog({
      handNumber: handNumber.value,
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

const updateHandLog = (entry: HandLogEntry) => {
  const resultText =
    entry.result === 'Tie'
      ? 'Tie'
      : `Player ${entry.result} $${Math.abs(entry.winAmount).toFixed(2)}`
  const displayBet = currentBet.value > entry.bet ? entry.bet * 2 : entry.bet // Display doubled bet if applicable
  handLog.value.unshift(
    `Hand #${entry.handNumber} | Bet: $${displayBet} | Player: ${entry.playerHand} | PlScore: ${entry.playerScore} | Dealer: ${entry.dealerHand} | DeScore: ${entry.dealerScore} | ${resultText} | Bankroll: $${entry.bankroll}`
  )
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
  const originalBet = currentBet.value // Store the original bet amount

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

  // Increment handNumber and create hand log for this hand
  handNumber.value++
  updateHandLog({
    handNumber: handNumber.value,
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
  handLog.value = []
  handNumber.value = 0 // Reset handNumber
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
  width: 98vw;
  height: 98vh;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-color: #2c7a2c;
  color: #ffffff;
  overflow: hidden;
}

.version {
  position: absolute;
  top: 15px;
  right: 45px;
  font-size: 14px;
  color: ##ffffff;
  font-weight: bold;
}

.grid-container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(3, 1fr);
  gap: 10px;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
}

.grid-item {
  padding: 10px;
  border-radius: 10px;
  background-color: rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  overflow: hidden;
}

.grid-item-1,
.grid-item-3,
.grid-item-7,
.grid-item-9 {
  background-color: transparent;
}

.game-info {
  text-align: center;
}

.game-info h2 {
  margin: 0 0 5px 0;
  font-size: 1.2em;
}

.game-info p {
  margin: 0;
  font-size: 0.9em;
}

.hand-log {
  width: 100%;
  height: 100%;
  overflow-y: auto;
  font-size: 0.8em;
}

.message {
  margin-top: 10px;
  font-weight: bold;
  text-align: center;
  font-size: 0.9em;
}

.hand {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
}

.hand :deep(.card) {
  width: 80px; /* Set a fixed width for cards */
  height: 120px; /* Set a fixed height for cards */
  margin: 5px; /* Adjust margin for spacing */
}

@media (max-width: 1024px) {
  .grid-container {
    grid-template-columns: 1fr;
    grid-template-rows: repeat(5, auto);
    height: auto;
    aspect-ratio: auto;
  }

  .grid-item-1,
  .grid-item-3,
  .grid-item-7,
  .grid-item-9 {
    display: none;
  }

  .grid-item-2 {
    order: 1;
  }
  .grid-item-8 {
    order: 2;
  }
  .grid-item-5 {
    order: 3;
  }
  .grid-item-4 {
    order: 4;
  }
  .grid-item-6 {
    order: 5;
  }

  .grid-item {
    padding: 10px;
    margin-bottom: 10px;
  }
}
</style>
