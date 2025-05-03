# Workshop Session 1: Building Memory Match Game with Base MiniKit

## Introduction

Welcome to our hands-on workshop on building interactive Mini Apps using Base MiniKit! In this session, we'll build a fun Memory Match game that demonstrates the power and flexibility of MiniKit for creating engaging experiences within Farcaster Frames.

### What We'll Build
- A fully functioning Memory Match card game
- Interactive game mechanics
- Score tracking and game progression
- Responsive design that works in Farcaster Frames

### Prerequisites
- Node.js installed (v18+ recommended)
- Basic knowledge of React and TypeScript
- A Farcaster account (for testing)
- Coinbase Developer Platform account (for CDP API key)

## Setup 

Let's start by setting up our development environment and creating a new MiniKit project.

### 1. Create a new MiniKit project using the CLI

```bash
npx create-onchain --mini
```

### 2. When prompted, enter your CDP Client API key.

You can get a CDP API key by going to the [CDP Portal](portal.cdp.coinbase.com) and navigating API Keys -> Client API Key.

```bash
npx @coinbase/create-minikit-app memory-match
```

Follow the prompts to set up your project. When asked about which template to use, select the basic template.

### 3. Skip Frames Account Manifest Setup

You will be asked if you'd like to set up your manifest. You can skip the manifest setup step as we'll handle that separately once we know our project's URL.

### 4. Navigate to your project directory and install dependencies

```bash
cd memory-match
npm install
npm run dev
```

### 5. Testing Your Mini App

To test your Mini App in Warpcast, you'll need a live URL.

We recommend using [Vercel](https://vercel.com/)to deploy your MiniKit app, as it integrates seamlessly with the upstash/redis backend required for stateful frames, webhooks, and notifications.

Alternatively, you can use [ngrok](https://ngrok.com/docs/getting-started/) to tunnel your localhost to a live url.

You can now test your mini app:

- Copy your deployed vercel URL
- Visit Warpcast Frames Developer Tools on web or mobile
- Paste URL into "Preview Frames"
- Tap Launch



## Building the Core Game 

Now, let's build the core components of our Memory Match game.

### 1. Setup the Game State (Game Context)

First, create a context to manage the game state:

```bash
mkdir -p contexts
touch contexts/GameContext.tsx
```

Now, open `contexts/GameContext.tsx` in your editor and add:

```typescript
// contexts/GameContext.tsx
import React, { createContext, useContext, useState, useEffect } from 'react';

// Define game history type
type GameHistory = {
  id: number;
  date: string;
  score: number;
  timeElapsed: number;
  flips: number;
};

// Define types
type Card = {
  id: number;
  value: string;
  flipped: boolean;
  matched: boolean;
};

type GameState = 'idle' | 'playing' | 'paused' | 'completed';

interface GameContextType {
  cards: Card[];
  gameState: GameState;
  score: number;
  timeElapsed: number;
  flips: number;
  gameHistory: GameHistory[];
  startGame: () => void;
  pauseGame: () => void;
  resumeGame: () => void;
  resetGame: () => void;
  flipCard: (id: number) => void;
  clearHistory: () => void;
}

// Create the context
const GameContext = createContext<GameContextType | undefined>(undefined);

// Create provider component
export const GameProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [cards, setCards] = useState<Card[]>([]);
  const [gameState, setGameState] = useState<GameState>('idle');
  const [score, setScore] = useState(0);
  const [timeElapsed, setTimeElapsed] = useState(0);
  const [flips, setFlips] = useState(0);
  const [timer, setTimer] = useState<NodeJS.Timeout | null>(null);
  const [gameHistory, setGameHistory] = useState<GameHistory[]>([]);
  
  // Load game history from localStorage on initial render
  useEffect(() => {
    const savedHistory = localStorage.getItem('memoryGameHistory');
    if (savedHistory) {
      try {
        setGameHistory(JSON.parse(savedHistory));
      } catch (error) {
        console.error('Error parsing saved game history:', error);
      }
    }
  }, []);
  
  // Save game history to localStorage when it changes
  useEffect(() => {
    localStorage.setItem('memoryGameHistory', JSON.stringify(gameHistory));
  }, [gameHistory]);
  
  // Initialize game with shuffled cards
  const initializeCards = () => {
    const emojis = ['🚀', '🌟', '🔥', '💎', '🌈', '🎮', '🎯', '🏆'];
    const cardPairs = [...emojis, ...emojis];
    const shuffled = cardPairs.sort(() => Math.random() - 0.5);
    
    setCards(
      shuffled.map((value, index) => ({
        id: index,
        value,
        flipped: false,
        matched: false,
      }))
    );
  };

  // Start the game
  const startGame = () => {
    initializeCards();
    setScore(0);
    setTimeElapsed(0);
    setFlips(0);
    setGameState('playing');
    
    // Start timer
    const interval = setInterval(() => {
      setTimeElapsed(prev => prev + 1);
    }, 1000);
    
    setTimer(interval);
  };

  // Pause the game
  const pauseGame = () => {
    if (timer) {
      clearInterval(timer);
      setTimer(null);
    }
    setGameState('paused');
  };

  // Resume the game
  const resumeGame = () => {
    const interval = setInterval(() => {
      setTimeElapsed(prev => prev + 1);
    }, 1000);
    
    setTimer(interval);
    setGameState('playing');
  };

  // Reset the game
  const resetGame = () => {
    if (timer) {
      clearInterval(timer);
      setTimer(null);
    }
    setGameState('idle');
    setScore(0);
    setTimeElapsed(0);
    setFlips(0);
    setCards([]);
  };

  // Clear game history
  const clearHistory = () => {
    setGameHistory([]);
    localStorage.removeItem('memoryGameHistory');
  };

  // Flip a card
  const flipCard = (id: number) => {
    // Don't allow flips if game is not in playing state
    if (gameState !== 'playing') return;
    
    // Don't allow flipping more than 2 unmatched cards
    const flippedUnmatched = cards.filter(card => card.flipped && !card.matched);
    if (flippedUnmatched.length >= 2) return;
    
    // Don't flip already flipped or matched cards
    const card = cards.find(card => card.id === id);
    if (!card || card.flipped || card.matched) return;
    
    // Flip the card
    setCards(prevCards => 
      prevCards.map(card => 
        card.id === id ? { ...card, flipped: true } : card
      )
    );
    
    setFlips(prev => prev + 1);
    
    // Check for match when two cards are flipped
    const newFlippedUnmatched = cards
      .filter(card => card.flipped && !card.matched)
      .concat({ ...card, flipped: true });
    
    if (newFlippedUnmatched.length === 2) {
      const [first, second] = newFlippedUnmatched;
      
      if (first.value === second.value) {
        // Match found - mark cards as matched
        setTimeout(() => {
          setCards(prevCards => 
            prevCards.map(card => 
              (card.id === first.id || card.id === second.id) 
                ? { ...card, matched: true } 
                : card
            )
          );
          
          // Add to score (100 points per match)
          setScore(prev => prev + 100);
          
          // Check if game is completed
          const allMatched = cards.every(card => 
            (card.id === first.id || card.id === second.id || card.matched)
          );
          
          if (allMatched) {
            if (timer) {
              clearInterval(timer);
              setTimer(null);
            }
            setGameState('completed');
            
            // Calculate final score based on time and flips
            const timeBonus = Math.max(0, 1000 - timeElapsed * 10);
            const flipsBonus = Math.max(0, 500 - flips * 10);
            const finalScore = score + timeBonus + flipsBonus + 100; // +100 for the last match
            setScore(finalScore);
            
            // Add game to history
            const newGameRecord: GameHistory = {
              id: Date.now(),
              date: new Date().toLocaleString(),
              score: finalScore,
              timeElapsed,
              flips
            };
            
            setGameHistory(prev => [newGameRecord, ...prev].slice(0, 10)); // Keep only last 10 games
          }
        }, 500);
      } else {
        // No match - flip cards back
        setTimeout(() => {
          setCards(prevCards => 
            prevCards.map(card => 
              (card.id === first.id || card.id === second.id) 
                ? { ...card, flipped: false } 
                : card
            )
          );
        }, 1000);
      }
    }
  };

  // Cleanup timer on component unmount
  useEffect(() => {
    return () => {
      if (timer) {
        clearInterval(timer);
      }
    };
  }, [timer]);

  // Provide the game context
  return (
    <GameContext.Provider
      value={{
        cards,
        gameState,
        score,
        timeElapsed,
        flips,
        gameHistory,
        startGame,
        pauseGame,
        resumeGame,
        resetGame,
        flipCard,
        clearHistory,
      }}
    >
      {children}
    </GameContext.Provider>
  );
};

// Create a hook to use the game context
export const useGame = () => {
  const context = useContext(GameContext);
  if (context === undefined) {
    throw new Error('useGame must be used within a GameProvider');
  }
  return context;
};
```

### 2. Create the Card Component

Now, let's create a component for the individual cards:

```bash
mkdir -p components
touch components/Card.tsx
```

Add the following to `components/Card.tsx`:

```typescript
// components/Card.tsx (update)
import React from 'react';
import { useGame } from '../contexts/GameContext';

interface CardProps {
  card: {
    id: number;
    value: string;
    flipped: boolean;
    matched: boolean;
  };
}

const Card: React.FC<CardProps> = ({ card }) => {
  const { flipCard } = useGame();
  
  const handleClick = () => {
    flipCard(card.id);
  };
  
  return (
    <div
      onClick={handleClick}
      className={`
        memory-card
        flex items-center justify-center text-2xl
        ${card.flipped || card.matched ? 'bg-white' : 'bg-blue-600'}
        ${card.matched ? 'bg-green-100 border-2 border-green-500 card-matched' : ''}
        ${card.flipped && !card.matched ? 'card-flip-in' : ''}
        ${!card.flipped && !card.matched ? 'card-flip-out' : ''}
        cursor-pointer
      `}
    >
      {(card.flipped || card.matched) && card.value}
    </div>
  );
};

export default Card;
```

### 3. Create the Game Board Component

Now, let's create the component for the game board:

```bash
touch components/GameBoard.tsx
```

Add the following to `components/GameBoard.tsx`:

```typescript
// components/GameBoard.tsx (update)
import React from 'react';
import { useGame } from '../contexts/GameContext';
import Card from './Card';

const GameBoard: React.FC = () => {
  const { cards, gameState, score, timeElapsed, flips } = useGame();
  
  // Don't render if game is not started
  if (gameState === 'idle') {
    return null;
  }
  
  return (
    <div className="game-content">
      {/* Game stats */}
      <div className="game-status">
        <div className="bg-blue-100 px-2 py-1 rounded-md">
          <span className="text-blue-800 font-medium">{timeElapsed}s</span>
        </div>
        <div className="bg-purple-100 px-2 py-1 rounded-md">
          <span className="text-purple-800 font-medium">{flips} flips</span>
        </div>
        <div className="bg-green-100 px-2 py-1 rounded-md">
          <span className="text-green-800 font-medium">{score} pts</span>
        </div>
      </div>
      
      {/* Card grid */}
      <div className="card-grid">
        {cards.map(card => (
          <Card key={card.id} card={card} />
        ))}
      </div>
    </div>
  );
};

export default GameBoard;
```

### 4. Create the Start Screen Component

```bash
touch components/StartScreen.tsx
```

Add the following to `components/StartScreen.tsx`:

```typescript
// components/StartScreen.tsx
import React from 'react';
import { useGame } from '../contexts/GameContext';

const StartScreen: React.FC = () => {
  const { gameState, startGame } = useGame();
  
  // Only show start screen when game is idle
  if (gameState !== 'idle') {
    return null;
  }
  
  return (
    <div className="flex flex-col items-center justify-center p-4">
      <h1 className="text-2xl font-bold mb-2 text-blue-600">Memory Match</h1>
      <p className="text-gray-600 text-center mb-6">
        Find matching pairs of cards to win!
      </p>
      
      <div className="p-4 bg-blue-50 rounded-lg border border-blue-100 w-full max-w-xs">
        <p className="text-sm text-blue-800 mb-4 text-center">
          Test your memory and earn points!
        </p>
        <button
          onClick={startGame}
          className="w-full bg-blue-600 text-white py-2 rounded-md font-medium"
        >
          Start Game
        </button>
      </div>
    </div>
  );
};

export default StartScreen;
```

### 5. Create the Game Complete Screen Component

```bash
touch components/GameComplete.tsx
```

Add the following to `components/GameComplete.tsx`:

```typescript
// components/GameComplete.tsx
import React from 'react';
import { useGame } from '../contexts/GameContext';

const GameComplete: React.FC = () => {
  const { gameState, score, timeElapsed, flips, resetGame } = useGame();
  
  // Only show when game is completed
  if (gameState !== 'completed') {
    return null;
  }
  
  return (
    <div className="fixed inset-0 flex items-center justify-center z-50 bg-black bg-opacity-50">
      <div className="bg-white p-6 rounded-xl shadow-xl max-w-xs text-center">
        <h2 className="text-2xl font-bold mb-2">Game Complete!</h2>
        <p className="text-4xl font-bold text-blue-600 mb-4">{score} pts</p>
        
        <div className="grid grid-cols-2 gap-2 mb-4">
          <div className="bg-blue-600 p-2 rounded">
            <p className="text-xs text-gray-100">Time</p>
            <p className="font-medium">{timeElapsed}s</p>
          </div>
          <div className="bg-blue-600 p-2 rounded">
            <p className="text-xs text-gray-100">Flips</p>
            <p className="font-medium">{flips}</p>
          </div>
        </div>
        
        <p className="text-sm mb-4">
          {score > 1500 ? 'Amazing! You have an incredible memory!' : 
           score > 1000 ? 'Great job! Your memory is impressive!' :
           'Good effort! Keep practicing to improve your score!'}
        </p>
        
        <button
          onClick={resetGame}
          className="bg-blue-600 text-white px-6 py-2 rounded-md"
        >
          Play Again
        </button>
      </div>
    </div>
  );
};

export default GameComplete;
```

### 6. Create the Controls Component with MiniKit Integration

```bash
touch components/GameControls.tsx
```

Add the following to `components/GameControls.tsx`:

```typescript
// components/GameControls.tsx
import React from 'react';
import { usePrimaryButton } from '@coinbase/onchainkit/minikit';
import { useGame } from '../contexts/GameContext';

const GameControls: React.FC = () => {
  const { gameState, startGame, pauseGame, resumeGame, resetGame } = useGame();
  
  // Configure primary button based on game state
  usePrimaryButton(
    { 
      text: gameState === 'idle' 
        ? 'Start Game' 
        : gameState === 'playing' 
          ? 'Pause' 
          : gameState === 'paused' 
            ? 'Resume'
            : 'Play Again' 
    },
    () => {
      if (gameState === 'idle') {
        startGame();
      } else if (gameState === 'playing') {
        pauseGame();
      } else if (gameState === 'paused') {
        resumeGame();
      } else if (gameState === 'completed') {
        resetGame();
      }
    }
  );
  
  return null; // This component only handles the primary button
};

export default GameControls;
```

### 7. Update the Main Page

Now, let's update the main page to use our components:

Open `app/page.tsx` and replace its contents with:

```typescript
// app/page.tsx (update)
'use client';
import { useEffect } from 'react';
import { useMiniKit, useOpenUrl } from '@coinbase/onchainkit/minikit';
import { GameProvider } from './contexts/GameContext';
import StartScreen from './components/StartScreen';
import GameBoard from './components/GameBoard';
import GameControls from './components/GameControls';
import GameComplete from './components/GameComplete';
import GameHistory from './components/GameHistory';

export default function Home() {
  const { setFrameReady, isFrameReady } = useMiniKit();
  const openUrl = useOpenUrl();
  
  // Initialize the frame
  useEffect(() => {
    if (!isFrameReady) {
      setFrameReady();
    }
  }, [setFrameReady, isFrameReady]);
  
  return (
    <GameProvider>
      <main className="flex min-h-screen min-w-screen flex-col items-center p-4 bg-white memory-game-container ">
        <StartScreen />
        <GameBoard />
        <GameControls />
        <GameComplete />
        <GameHistory />
        
        <footer className="mt-auto ">
          <button 
            onClick={() => openUrl('https://base.org/builders/minikit')}
            className="text-xs opacity-60 px-2 py-1 border border-blue-300 text-blue-600 rounded-full"
          >
            BUILT WITH MINIKIT
          </button>
        </footer>
      </main>
    </GameProvider>
  );
}

```

## Adding Final Touches and Testing (20 minutes)

### 1. Add CSS Animations

Let's add some animations to make our game more engaging. Create a new file:

```bash
touch app/animations.css
```

Add the following CSS:

```css
/* app/animations.css */
@keyframes flipIn {
  from {
    transform: rotateY(0deg);
  }
  to {
    transform: rotateY(180deg);
  }
}

@keyframes flipOut {
  from {
    transform: rotateY(180deg);
  }
  to {
    transform: rotateY(0deg);
  }
}

.card-flip-in {
  animation: flipIn 0.3s forwards;
}

.card-flip-out {
  animation: flipOut 0.3s forwards;
}

.card-matched {
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(52, 211, 153, 0.7);
  }
  70% {
    box-shadow: 0 0 0 10px rgba(52, 211, 153, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(52, 211, 153, 0);
  }
}
```


### 2. Add Responsive layout

Let's add some responsive css, Create a new file:

```bash
touch app/responsive.css
```

Add the following CSS:

```
/* app/responsive.css */
/* Main container responsive rules */
.memory-game-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center; /* Center vertically */
    min-height: 100vh; /* Use full viewport height */
    padding: 1rem;
    width: 100%;
    max-width: 520px;
    margin: 0 auto;
  }
  
  /* Game header styling */
  .game-header {
    width: 100%;
    margin-bottom: 1rem;
    text-align: center;
  }
  
  /* Game content area */
  .game-content {
    width: 100%;
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 1rem;
  }
  
  /* Card grid container */
  .card-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 0.5rem;
    width: 100%;
    max-width: 400px;
    margin: 0 auto;
  }
  
  /* Card styling */
  .memory-card {
    aspect-ratio: 1/1; /* Keep cards square */
    width: 100%;
    border-radius: 0.5rem;
    transition: all 0.3s ease;
  }
  
  /* Game status bar */
  .game-status {
    display: flex;
    justify-content: space-between;
    width: 100%;
    max-width: 400px;
    margin-bottom: 0.5rem;
  }
  
  /* Footer positioning */
  .game-footer {
    margin-top: 1rem;
    padding-top: 1rem;
    width: 100%;
    display: flex;
    justify-content: center;
  }
  
  /* Ensure proper modal centering */
  .modal-container {
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 100%;
  }
  
  /* Media query for very small screens */
  @media (max-height: 600px) {
    .card-grid {
      gap: 0.25rem;
    }
    
    .memory-card {
      border-radius: 0.25rem;
    }
  }
  
  /* Media query for larger screens */
  @media (min-height: 800px) {
    .game-content {
      gap: 2rem;
    }
  }
```

### 3. Import the CSS in Layout.tsx

Open `app/layout.tsx` and add the import:

```typescript
import './animation.css';
import './responsive.css';
```

### 4. Test the Game

Now, let's test our game:

1. Make sure your development server is running (`npm run dev`)
2. Open `http://localhost:3000` in your browser
3. Play the game to check if everything works properly
4. Test the game on different screen sizes to ensure responsive design

## Preparing for Farcaster Integration 

### 1. Configuring the Manifest

To make our game available on Farcaster, we need to set up a frame manifest:

```bash
npx create-onchain --manifest
```

The wallet that you connect must be your Farcaster custody wallet. You can import this wallet to your prefered wallet using the recovery phrase. You can find your recovery phrase in the Warpcast app under Settings -> Advanced -> Farcster recovery phrase.

Once connected, add the vercel url and sign the manifest. This will automatically update your .env variables locally, but we'll need to update Vercel's .env variables.

Create the following new .env variables in your vercel instance and paste the value you see in your local.env file

- FARCASTER_HEADER
- FARCASTER_PAYLOAD
- FARCASTER_SIGNATURE

Now that the manifest is correctly set up, the Save Frame button in the template app should work. We'll explore that below.

You can now test your mini app:

- Copy your deployed vercel URL
- Visit Warpcast Frames Developer Tools on web or mobile
- Paste URL into "Preview Frames"
- Tap Launch

## Conclusion and Next Steps 

Congratulations! You've built a complete Memory Match game using Base MiniKit. Let's recap what we've learned:

1. Setting up a MiniKit project
2. Creating game mechanics with React state management
3. Using MiniKit hooks for Farcaster integration
4. Deploying the application on Vercel

In the next session, we'll enhance our game by adding blockchain integration:
- On-chain leaderboards
- Prize pools
- Smart contract integration

## Q&A (10 minutes)

Any questions about what we've covered so far?
