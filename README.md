# Vue 3 Blackjack Component

## Description

This is a Vue 3 component that implements a Blackjack game. It features a responsive design, card counting functionality, and customizable game settings.

## Features

- Play Blackjack against a dealer
- Responsive design for various screen sizes
- Card counting display (Running Count and True Count)
- Customizable number of decks and starting bankroll
- Split, Double Down, Hit, and Stand actions
- Realistic game logic including Blackjack payouts

# Blackjack Project Version History

## Version 1.0 - 1.5

- 1.0: Initial release with basic Blackjack gameplay
- 1.1: Added simple betting system
- 1.2: Implemented dealer AI and basic hand evaluation
- 1.3: Introduced card counting feature (running count)
- 1.4: Added true count calculation
- 1.5: Implemented basic hand log

## Version 2.0 - 2.6

- 2.0: Major refactor for improved code structure
- 2.1: Enhanced betting system with minimum and maximum limits
- 2.2: Added split functionality
- 2.3: Implemented double down option
- 2.4: Improved hand log with more detailed information
- 2.5: Added player statistics tracking
- 2.6: Introduced basic achievement system

## Version 3.0 - 3.6

- 3.0: Refactored to component-based architecture
- 3.1: Improved dealer blackjack detection and logging
- 3.2: Created separate HandLog and CardCounting components
- 3.3: Enhanced error handling for split hands and improved win/loss reporting in hand log
- 3.4: Betting Input set to minimum 0.50 increments
- 3.5: Refactored bankroll management, enhanced hand logging with hand numbers, and improved code maintainability
- 3.6: Implemented HandLog component for managing game logs, refactored MainBlackjack.vue to use it, and improved code structure

## Logic and Function Explanations

### Core Game Functions

1. `winLoss(result, bet)`: Centralizes bankroll adjustments for wins, losses, ties, and blackjacks.
2. `dealInitialHands()`: Handles the initial deal of cards and checks for blackjacks.
3. `hit()`: Allows the player to take another card.
4. `stand()`: Ends the player's turn and initiates the dealer's turn.
5. `double()`: Doubles the player's bet and deals one more card.
6. `split()`: Splits a pair into two separate hands.
7. `checkForBlackjackAfterSplit(hand)`: Checks for blackjack in split hands.
8. `compareHands(playerScore, dealerScore, handIndex)`: Compares player and dealer hands to determine the winner.
9. `nextHand()`: Moves to the next split hand.
10. `endGame()`: Finalizes the game round.
11. `resetGame()`: Resets the game state for a new round.

### Helper Functions

1. `calculateHandValue(hand)`: Calculates the value of a hand, accounting for Aces.
2. `dealCard(hand, isDealerHidden)`: Deals a card from the deck to a hand.
3. `shuffleDeck()`: Shuffles the deck of cards.
4. `updateRunningCount(card)`: Updates the running count based on the dealt card.
5. `revealDealerHiddenCard()`: Reveals the dealer's hidden card.

### Game Flow Functions

1. `startGame()`: Initializes the game with user-defined settings.
2. `placeBet()`: Handles the betting process before each hand.

### Computed Properties

1. `visibleDealerHand`: Returns the visible cards in the dealer's hand.
2. `dealerVisibleScore`: Calculates the visible score of the dealer's hand.
3. `canHit`: Determines if the player can hit.
4. `canDouble`: Determines if the player can double down.
5. `canSplit`: Determines if the player can split their hand.
6. `playerScore`: Calculates the player's hand score.
7. `dealerScore`: Calculates the dealer's hand score.

## How to Use

## Customization

You can customize the game by modifying the following:

- Number of decks used (1-6)
- Starting bankroll ($100-$100,000)
- Betting limits

## Future Improvements

- Add multiplayer functionality
- Implement more advanced betting strategies
- Create a leaderboard system
- Add sound effects and animations

## License

This project is open source and available under the [MIT License](LICENSE).
