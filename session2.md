# Workshop Session 2: Integrating On-Chain Features with MiniKit

## Introduction

Welcome to the second session of our MiniKit workshop! Today, we'll build upon our Memory Match game by adding blockchain integration using OnChain Kit. We'll implement on-chain leaderboards, a prize pool, and wallet connectivity to create a fully interactive web3 gaming experience.

### What We'll Build
- Wallet connectivity using OnChain Kit's modern wallet components
- Smart contract for leaderboard and prize pool
- On-chain score submission system using Transaction component
- Admin functions to manage prizes
- Enhanced user interface with blockchain data

### Prerequisites
- Completed Memory Match game from Session 1
- Node.js installed (v18+ recommended)
- Knowledge of Solidity basics (helpful but not required)
- A Base Sepolia testnet wallet with some testnet ETH
- Coinbase Wallet extension or mobile app for testing

## Setup

Let's start by setting up our development environment for blockchain integration.

### 1. Install OnChain Kit Dependencies and set up config

First, let's add the necessary dependencies for blockchain integration:

```bash
cd memory-match
npm install @coinbase/onchainkit @tanstack/react-query viem wagmi react-hot-toast
```


config/index.ts

```typescript
import { baseSepolia } from "viem/chains";
import { createConfig, http } from "wagmi";

export const config = createConfig({
    chains: [baseSepolia],
    transports: {
      [baseSepolia.id]: http(),
    },
})
```

### 2. Create a Smart Contract for Leaderboard and Prize Pool

Create a new directory for our contracts and a Solidity file:

```bash
mkdir -p contracts
touch contracts/MemoryMatchLeaderboard.sol
```

Add the following Solidity code to `contracts/MemoryMatchLeaderboard.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MemoryMatchLeaderboard {
    struct ScoreEntry {
        address player;
        uint256 score;
        uint256 timestamp;
    }
    
    // Owner of the contract
    address public owner;
    
    // Top 10 scores
    ScoreEntry[] public topScores;
    
    // Prize pool amount
    uint256 public prizePool;
    
    // Events
    event ScoreSubmitted(address indexed player, uint256 score, uint256 timestamp);
    event PrizePoolIncreased(address indexed sender, uint256 amount, uint256 newTotal);
    event PrizeAwarded(address indexed winner, uint256 amount);
    
    // Constructor
    constructor() {
        owner = msg.sender;
    }
    
    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }
    
    // Submit a score
    function submitScore(uint256 _score) external {
        uint256 timestamp = block.timestamp;
        
        // Initialize topScores with empty entries if needed
        if (topScores.length < 10) {
            topScores.push(ScoreEntry({
                player: msg.sender,
                score: _score,
                timestamp: timestamp
            }));
            
            // Sort the array
            _sortTopScores();
        } else {
            // Check if new score is higher than the lowest score
            if (_score > topScores[9].score) {
                topScores[9] = ScoreEntry({
                    player: msg.sender,
                    score: _score,
                    timestamp: timestamp
                });
                
                // Sort the array
                _sortTopScores();
            }
        }
        
        emit ScoreSubmitted(msg.sender, _score, timestamp);
    }
    
    // Internal function to sort the topScores array
    function _sortTopScores() internal {
        uint256 n = topScores.length;
        
        for (uint256 i = 0; i < n; i++) {
            for (uint256 j = i + 1; j < n; j++) {
                if (topScores[i].score < topScores[j].score) {
                    ScoreEntry memory temp = topScores[i];
                    topScores[i] = topScores[j];
                    topScores[j] = temp;
                }
            }
        }
    }
    
    // Add funds to prize pool
    function addToPrizePool() external payable {
        require(msg.value > 0, "Must send some ETH");
        prizePool += msg.value;
        emit PrizePoolIncreased(msg.sender, msg.value, prizePool);
    }
    
    // Award prize to top player (owner only)
    function awardPrize() external onlyOwner {
        require(prizePool > 0, "Prize pool is empty");
        require(topScores.length > 0, "No scores submitted yet");
        
        address winner = topScores[0].player;
        uint256 amount = prizePool;
        prizePool = 0;
        
        (bool success, ) = winner.call{value: amount}("");
        require(success, "Transfer failed");
        
        emit PrizeAwarded(winner, amount);
    }
    
    // Get all top scores
    function getTopScores() external view returns (ScoreEntry[] memory) {
        return topScores;
    }
    
    // Get a player's best score and rank
    function getPlayerBestScore(address _player) external view returns (uint256 score, uint256 rank) {
        score = 0;
        rank = 0;
        
        for (uint256 i = 0; i < topScores.length; i++) {
            if (topScores[i].player == _player && topScores[i].score > score) {
                score = topScores[i].score;
                rank = i + 1;
            }
        }
        
        return (score, rank);
    }
}
```

### 3. Create Contract ABIs

After deploying the contract to Base Sepolia, we'll need its ABI. Create a directory for contract ABIs:

create /app/memoryMatchContract.ts

```bash
import { type Abi } from 'viem';

// Cast the ABI to the proper Abi type
export const MEMORY_MATCH_CONTRACT: Abi = [
  {
    "inputs": [],
    "stateMutability": "nonpayable",
    "type": "constructor"
  },
  {
    "anonymous": false,
    "inputs": [
      {
        "indexed": true,
        "internalType": "address",
        "name": "winner",
        "type": "address"
      },
      {
        "indexed": false,
        "internalType": "uint256",
        "name": "amount",
        "type": "uint256"
      }
    ],
    "name": "PrizeAwarded",
    "type": "event"
  },
  {
    "anonymous": false,
    "inputs": [
      {
        "indexed": true,
        "internalType": "address",
        "name": "sender",
        "type": "address"
      },
      {
        "indexed": false,
        "internalType": "uint256",
        "name": "amount",
        "type": "uint256"
      },
      {
        "indexed": false,
        "internalType": "uint256",
        "name": "newTotal",
        "type": "uint256"
      }
    ],
    "name": "PrizePoolIncreased",
    "type": "event"
  },
  {
    "anonymous": false,
    "inputs": [
      {
        "indexed": true,
        "internalType": "address",
        "name": "player",
        "type": "address"
      },
      {
        "indexed": false,
        "internalType": "uint256",
        "name": "score",
        "type": "uint256"
      },
      {
        "indexed": false,
        "internalType": "uint256",
        "name": "timestamp",
        "type": "uint256"
      }
    ],
    "name": "ScoreSubmitted",
    "type": "event"
  },
  {
    "inputs": [],
    "name": "addToPrizePool",
    "outputs": [],
    "stateMutability": "payable",
    "type": "function"
  },
  {
    "inputs": [],
    "name": "awardPrize",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "inputs": [
      {
        "internalType": "address",
        "name": "_player",
        "type": "address"
      }
    ],
    "name": "getPlayerBestScore",
    "outputs": [
      {
        "internalType": "uint256",
        "name": "score",
        "type": "uint256"
      },
      {
        "internalType": "uint256",
        "name": "rank",
        "type": "uint256"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [],
    "name": "getTopScores",
    "outputs": [
      {
        "components": [
          {
            "internalType": "address",
            "name": "player",
            "type": "address"
          },
          {
            "internalType": "uint256",
            "name": "score",
            "type": "uint256"
          },
          {
            "internalType": "uint256",
            "name": "timestamp",
            "type": "uint256"
          }
        ],
        "internalType": "struct MemoryMatchLeaderboard.ScoreEntry[]",
        "name": "",
        "type": "tuple[]"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [],
    "name": "owner",
    "outputs": [
      {
        "internalType": "address",
        "name": "",
        "type": "address"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [],
    "name": "prizePool",
    "outputs": [
      {
        "internalType": "uint256",
        "name": "",
        "type": "uint256"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [
      {
        "internalType": "uint256",
        "name": "_score",
        "type": "uint256"
      }
    ],
    "name": "submitScore",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "inputs": [
      {
        "internalType": "uint256",
        "name": "",
        "type": "uint256"
      }
    ],
    "name": "topScores",
    "outputs": [
      {
        "internalType": "address",
        "name": "player",
        "type": "address"
      },
      {
        "internalType": "uint256",
        "name": "score",
        "type": "uint256"
      },
      {
        "internalType": "uint256",
        "name": "timestamp",
        "type": "uint256"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  }
] as const;

export const MEMORY_MATCH_CONTRACT_ADDRESS = "0x6292801F2598D7a24FA99265685bcCD4DcFB0Fc2";
```

### 4. Set Up OnChain Kit Provider

First, let's set up the OnChain Kit provider in our application layout:

```bash
touch app/providers.tsx
```

Add the following to `app/providers.tsx`:

```typescript
"use client";

import { type ReactNode } from "react";
import { base, baseSepolia } from "wagmi/chains";
import { MiniKitProvider } from "@coinbase/onchainkit/minikit";
import { OnchainKitProvider } from '@coinbase/onchainkit';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { createConfig, http, WagmiProvider } from "wagmi";
import { coinbaseWallet, injected,  } from "wagmi/connectors";


const wagmiConfig = createConfig({
  chains: [baseSepolia],
  connectors: [
    coinbaseWallet({
      appName: 'onchainkit',
    }),

  ],
  ssr: true,
  transports: {
    [baseSepolia.id]: http(),
  },
});

export function Providers(props: { children: ReactNode }) {
  return (
    <QueryClientProvider client={new QueryClient()} >
      <WagmiProvider config={wagmiConfig}> 
      <MiniKitProvider
        apiKey={process.env.NEXT_PUBLIC_ONCHAINKIT_API_KEY}
        chain={baseSepolia}
        config={{
          appearance: {
            mode: "auto",
            theme: "mini-app-theme",
            name: process.env.NEXT_PUBLIC_ONCHAINKIT_PROJECT_NAME,
            logo: process.env.NEXT_PUBLIC_ICON_URL,
          },
        }}
        
      >
        {props.children}
      </MiniKitProvider>
      </WagmiProvider>
    </QueryClientProvider>

    
  );
}
```

## Implementing Blockchain Features

Now, let's integrate the blockchain features into our game using OnChain Kit components.

### 1. Create a Blockchain Context

First, let's create a context to handle blockchain interactions:

```bash
touch contexts/BlockchainContext.tsx
```

Add the following to `contexts/BlockchainContext.tsx`:

```typescript
// contexts/BlockchainContext.tsx
import React, { createContext, useContext, useState, useEffect, useCallback } from 'react';
import { useAccount, useReadContract, useReadContracts } from 'wagmi';
import { formatEther } from 'viem';
import { baseSepolia } from 'viem/chains';
import { MEMORY_MATCH_CONTRACT, MEMORY_MATCH_CONTRACT_ADDRESS } from '../memoryMatchContract';

// Contract address from environment variables
const contractABI = MEMORY_MATCH_CONTRACT
const contractAddress = MEMORY_MATCH_CONTRACT_ADDRESS

// Types for leaderboard scores
export type ScoreEntry = {
  player: `0x${string}`;
  score: number;
  timestamp: number;
};

interface BlockchainContextType {
  address: `0x${string}` | undefined;
  isOwner: boolean;
  topScores: ScoreEntry[];
  prizePool: string;
  playerBestScore: number;
  playerRank: number;
  refreshData: () => void;
  isLoading: boolean;
}

// Create the context
const BlockchainContext = createContext<BlockchainContextType | undefined>(undefined);

// Provider component
export const BlockchainProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const { address, isConnected } = useAccount();
  
  const [isOwner, setIsOwner] = useState(false);
  const [playerBestScore, setPlayerBestScore] = useState(0);
  const [playerRank, setPlayerRank] = useState(0);
  const [topScores, setTopScores] = useState<ScoreEntry[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  
  // Read contract owner
  const { data: ownerData, refetch: refetchOwner } = useReadContract({
    address: contractAddress,
    abi: contractABI,
    functionName: 'owner',
    chainId: baseSepolia.id,
  });
  
  // Read prize pool
  const { data: prizePoolData, refetch: refetchPrizePool } = useReadContract({
    address: contractAddress,
    abi: contractABI,
    functionName: 'prizePool',
    chainId: baseSepolia.id,
  });
  
  // Read top scores
  const { data: topScoresData, refetch: refetchTopScores } = useReadContract({
    address: contractAddress,
    abi: contractABI,
    functionName: 'getTopScores',
    chainId: baseSepolia.id,
  });
  
  // Read player best score
  const { data: playerScoreData, refetch: refetchPlayerScore } = useReadContract({
    address: contractAddress,
    abi: contractABI,
    functionName: 'getPlayerBestScore',
    args: [address ?? '0x0000000000000000000000000000000000000000'],
    chainId: baseSepolia.id,
  });
  
  // Check if connected account is the contract owner
  useEffect(() => {
    if (address && ownerData) {
      setIsOwner(address.toLowerCase() === (ownerData as string).toLowerCase());
    } else {
      setIsOwner(false);
    }
  }, [address, ownerData]);
  
  // Update player score and rank
  useEffect(() => {
    if (playerScoreData) {
      const [score, rank] = playerScoreData as [bigint, bigint];
      setPlayerBestScore(Number(score));
      setPlayerRank(Number(rank));
    }
  }, [playerScoreData]);
  
  // Update top scores
  useEffect(() => {
    if (topScoresData) {
      const formattedScores = (topScoresData as any[]).map(entry => ({
        player: entry.player,
        score: Number(entry.score),
        timestamp: Number(entry.timestamp),
      }));
      setTopScores(formattedScores);
    }
  }, [topScoresData]);
  
  // Function to refresh all data
  const refreshData = useCallback(async () => {
    setIsLoading(true);
    try {
      await Promise.all([
        refetchOwner(),
        refetchPrizePool(),
        refetchTopScores(),
        address ? refetchPlayerScore() : Promise.resolve(),
      ]);
    } catch (error) {
      console.error('Error refreshing data:', error);
    } finally {
      setIsLoading(false);
    }
  }, [refetchOwner, refetchPrizePool, refetchTopScores, refetchPlayerScore, address]);
  
  // Format prize pool
  const prizePool = prizePoolData ? formatEther(prizePoolData as bigint) : '0';
  
  return (
    <BlockchainContext.Provider
      value={{
        address,
        isOwner,
        topScores,
        prizePool,
        playerBestScore,
        playerRank,
        refreshData,
        isLoading,
      }}
    >
      {children}
    </BlockchainContext.Provider>
  );
};

// Hook to use the blockchain context
export const useBlockchain = () => {
  const context = useContext(BlockchainContext);
  if (context === undefined) {
    throw new Error('useBlockchain must be used within a BlockchainProvider');
  }
  return context;
};
```

### 2. Create Wallet Button Component

Now, let's create a component for wallet connection using OnChain Kit's wallet components:

```bash
mkdir -p components
touch components/WalletButton.tsx
```

Add the following to `components/WalletButton.tsx`:

```typescript
// components/WalletButton.tsx
import React from 'react';
import {
  Wallet,
  ConnectWallet,
  WalletDropdown,
  WalletDropdownDisconnect,
  
} from '@coinbase/onchainkit/wallet';
import {
  Name,
  Identity,
  Address,
  Avatar,
  EthBalance
} from '@coinbase/onchainkit/identity';
import { useBlockchain } from '../contexts/BlockchainContext';

const WalletButton: React.FC = () => {
  const { playerBestScore, playerRank } = useBlockchain();
  
  return (
    <div className="flex flex-col items-center">

<Wallet className="z-10">
                <ConnectWallet>
                  <Name className="text-inherit" />
                </ConnectWallet>
                <WalletDropdown>
                  <Identity className="px-4 pt-3 pb-2" hasCopyAddressOnClick>
                    <Avatar />
                    <Name />
                    <Address />
                    <EthBalance />
                  </Identity>

                  {playerBestScore > 0 && (
            <div className="px-4 py-2 border-t border-gray-100">
              <div className="text-sm text-gray-600">
                <span className="font-medium">Best Score:</span> {playerBestScore} pts
              </div>
              {playerRank > 0 && (
                <div className="text-sm text-gray-600">
                  <span className="font-medium">Rank:</span> #{playerRank}
                </div>
              )}
            </div>
          )}
                  <WalletDropdownDisconnect />
                </WalletDropdown>
              </Wallet>
    </div>
  );
};

export default WalletButton;
```

### 3. Create Leaderboard Component

Let's create a leaderboard component to display the top scores and include a transaction button for awarding prizes:

```bash
touch components/Leaderboard.tsx
```

Add the following to `components/Leaderboard.tsx`:

```typescript
// components/Leaderboard.tsx
import React, { useEffect } from 'react';
import { useBlockchain, ScoreEntry } from '../contexts/BlockchainContext';
import { Transaction } from '@coinbase/onchainkit/transaction';
import { baseSepolia } from 'viem/chains';
import { MEMORY_MATCH_CONTRACT, MEMORY_MATCH_CONTRACT_ADDRESS } from '../memoryMatchContract';
import { useAccount, useWriteContract } from 'wagmi';
import { waitForTransactionReceipt } from 'wagmi/actions';
import { config } from '../config';

// Function to format timestamp
const formatTimestamp = (timestamp: number) => {
  const date = new Date(timestamp * 1000);
  return date.toLocaleDateString();
};

// Function to format address
const formatAddress = (address: string) => {
  return `${address.substring(0, 6)}...${address.substring(address.length - 4)}`;
};

const Leaderboard: React.FC = () => {
  const { topScores, prizePool, isOwner, refreshData } = useBlockchain();
  const {address} = useAccount();

  const { writeContractAsync } = useWriteContract()


      const handleAwardPrize = async () => {
        const result = await writeContractAsync(
          {
            address: MEMORY_MATCH_CONTRACT_ADDRESS,
                  abi: MEMORY_MATCH_CONTRACT,
                  functionName: 'awardPrize',
                  args: [],
          }
        );
    
        await waitForTransactionReceipt(config, {
          hash: result as `0x${string}`,
        })
          .then(async () => {
            await refreshData()
          })
          .catch(() => {
            console.log("Transaction failed");
          });
      }
    
  
  
  // Contract calls for awarding prize

  return (
    <div className="bg-white p-4 rounded-lg shadow-md w-full max-w-lg">
      <div className="flex justify-between items-center mb-4">
        <h2 className="text-xl font-bold text-blue-600">Top Players</h2>
        <div className="flex items-center gap-2">
          <div className="bg-yellow-100 text-yellow-800 px-3 py-1 rounded-full text-sm font-medium">
            Prize Pool: {parseFloat(prizePool).toFixed(4)} ETH
          </div>
          
          {address && isOwner && parseFloat(prizePool) > 0 && (
            // <Transaction 
            //   calls={[
            //     {
            //         address: MEMORY_MATCH_CONTRACT_ADDRESS,
            //         abi: MEMORY_MATCH_CONTRACT,
            //         functionName: 'awardPrize',
            //         args: [],
            //       }
            //   ]}
            //   chainId={baseSepolia.id}
            //   onStatus={(status) => {
            //     if (status.statusName === 'success') {
            //       refreshData();
            //     }
            //   }}
            // />

            <button
              className="bg-blue-500 hover:bg-blue-600 text-white px-3 py-1 rounded-full text-sm font-medium"
              onClick={() => {
                handleAwardPrize();
              }}
            >
              Award Prize
            </button>
          )}
        </div>
      </div>
      
      {topScores.length > 0 ? (
        <div className="bg-gray-50 rounded-lg overflow-hidden">
          <table className="min-w-full divide-y divide-gray-200">
            <thead className="bg-gray-100">
              <tr>
                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Rank</th>
                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Player</th>
                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Score</th>
                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Date</th>
              </tr>
            </thead>
            <tbody className="bg-white divide-y divide-gray-200">
              {topScores.map((entry: ScoreEntry, index: number) => (
                <tr key={`${entry.player}-${entry.timestamp}`} className={index === 0 ? 'bg-yellow-50' : ''}>
                  <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-900">
                    {index === 0 ? (
                      <span className="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-yellow-100 text-yellow-800">
                        🏆 1
                      </span>
                    ) : (
                      `${index + 1}`
                    )}
                  </td>
                  <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-600">{formatAddress(entry.player)}</td>
                  <td className="px-3 py-2 whitespace-nowrap text-sm font-medium text-blue-600">{entry.score}</td>
                  <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-500">{formatTimestamp(entry.timestamp)}</td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      ) : (
        <div className="text-center py-8 bg-gray-50 rounded-lg">
          <p className="text-gray-500">No scores submitted yet. Be the first!</p>
        </div>
      )}
    </div>
  );
};

export default Leaderboard;
```

### 4. Create Prize Pool Form Component

Let's create a component for contributing to the prize pool:

```bash
touch components/PrizePoolForm.tsx
```

Add the following to `components/PrizePoolForm.tsx`:

```typescript
// components/PrizePoolForm.tsx
import React, { useEffect, useState } from 'react';
import { useBlockchain } from '../contexts/BlockchainContext';
import { Transaction } from '@coinbase/onchainkit/transaction';
import { parseEther } from 'viem';
import { baseSepolia } from 'viem/chains';
import { MEMORY_MATCH_CONTRACT, MEMORY_MATCH_CONTRACT_ADDRESS } from '../memoryMatchContract';
import { useWriteContract } from 'wagmi';
import { waitForTransactionReceipt } from 'wagmi/actions';
import { config } from '../config';

const PrizePoolForm: React.FC = () => {
  const { prizePool, refreshData } = useBlockchain();
  const [amount, setAmount] = useState('0.01');
  const contractAddress = MEMORY_MATCH_CONTRACT_ADDRESS as `0x${string}`;

  const { writeContractAsync } = useWriteContract()

  const handleContribute = async () => {
          const result = await writeContractAsync(
            {
              address: contractAddress,
                abi: MEMORY_MATCH_CONTRACT,
                functionName: 'addToPrizePool',
                args: [],
                value: BigInt(parseEther(amount).toString()),
            }
          );
      
          await waitForTransactionReceipt(config, {
            hash: result as `0x${string}`,
          })
            .then(async () => {
              await refreshData()
            })
            .catch(() => {
              console.log("Transaction failed");
            });
        }
  // Contract calls for contributing to prize pool

  
  return (
    <div className="bg-white p-4 rounded-lg shadow-md w-full max-w-lg">
      <h2 className="text-xl font-bold text-blue-600 mb-4">Contribute to Prize Pool</h2>
      
      <div className="mb-4 p-3 bg-blue-50 rounded-md">
        <div className="flex justify-between items-center">
          <span className="text-blue-700 font-medium">Current Prize Pool:</span>
          <span className="text-blue-800 font-bold">{parseFloat(prizePool).toFixed(4)} ETH</span>
        </div>
      </div>
      
      <div className="mb-4">
        <label htmlFor="amount" className="block text-sm font-medium text-gray-700 mb-1">
          Amount (ETH)
        </label>
        <div className="flex gap-2">
          <input
            type="number"
            id="amount"
            value={amount}
            onChange={(e) => setAmount(e.target.value)}
            step="0.001"
            min="0.001"
            className="block w-full rounded-md text-blue-600 border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 sm:text-sm"
          />
          
          <button
            onClick={() => {
              handleContribute();
            }}
            className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
          >
            Contribute
          </button>
          {/* {hash && <div>Transaction Hash: {hash}</div>} */}

          {/* <Transaction 
            calls={[
                {
                  address: contractAddress,
                  abi: MEMORY_MATCH_CONTRACT,
                functionName: 'addToPrizePool',
                  args: [],
                  value: BigInt(parseEther(amount).toString()),
                }
              ]}
            chainId={baseSepolia.id}
            onStatus={(status) => {
              if (status.statusName === 'success') {
                refreshData();
              }
            }}
          /> */}
        </div>
      </div>
    </div>
  );
};

export default PrizePoolForm;
```

### 5. Update Game Complete Component

Now, let's update the GameComplete component to allow submitting scores to the blockchain:

```bash
touch components/GameComplete.tsx
```

Add the following to `components/GameComplete.tsx`:

```typescript
// components/GameComplete.tsx
import React, { useState } from 'react';
import { useGame } from '../contexts/GameContext';
import { useBlockchain } from '../contexts/BlockchainContext';
import { Transaction } from '@coinbase/onchainkit/transaction';
import { baseSepolia } from 'viem/chains';
import { MEMORY_MATCH_CONTRACT, MEMORY_MATCH_CONTRACT_ADDRESS } from '../memoryMatchContract';
import { useWaitForTransactionReceipt, useWriteContract } from 'wagmi';
import { waitForTransactionReceipt } from "@wagmi/core";
import { config } from '../config';

const GameComplete: React.FC = () => {
  const { 
    gameState, 
    score, 
    timeElapsed, 
    flips, 
    resetGame 
  } = useGame();
  
  const { address, refreshData } = useBlockchain();
  const [scoreSubmitted, setScoreSubmitted] = useState(false);


  const { data: hash, writeContract, isSuccess, writeContractAsync } = useWriteContract()
  const result = useWaitForTransactionReceipt({
    hash: hash,
  })
  
  
  const contractAddress = process.env.NEXT_PUBLIC_CONTRACT_ADDRESS as `0x${string}`;
  
  // Only show when game is completed
  if (gameState !== 'completed') {
    return null;
  }
  
  // Contract calls for submitting score
  const scoreSubmitCalls = [
    {
      address: contractAddress,
      abi: MEMORY_MATCH_CONTRACT,
      functionName: 'submitScore',
      args: [BigInt(score)],
    }
  ];

  const handleSubmitScore = async () => {
    const result = await writeContractAsync(
      {
        address: contractAddress,
        abi: MEMORY_MATCH_CONTRACT,
        functionName: 'submitScore',
        args: [BigInt(score)],
      }
    );

    await waitForTransactionReceipt(config, {
      hash: result as `0x${string}`,
    })
      .then(async () => {
        await refreshData()
      })
      .catch(() => {
        console.log("Transaction failed");
      });

  }


  
  return (
    <div className="fixed inset-0 flex items-center justify-center z-50 bg-black bg-opacity-50">
      <div className="bg-white p-6 rounded-xl shadow-xl max-w-sm text-center">
        <h2 className="text-2xl font-bold mb-2">Game Complete!</h2>
        <p className="text-4xl font-bold text-blue-600 mb-4">{score} pts</p>
        
        <div className="grid grid-cols-2 gap-2 mb-4">
          <div className="bg-blue-600 p-2 rounded">
            <p className="text-xs text-gray-100">Time</p>
            <p className="font-medium text-white">{timeElapsed}s</p>
          </div>
          <div className="bg-blue-600 p-2 rounded">
            <p className="text-xs text-gray-100">Flips</p>
            <p className="font-medium text-white">{flips}</p>
          </div>
        </div>
        
        <p className="text-sm mb-4">
          {score > 1500 ? 'Amazing! You have an incredible memory!' : 
           score > 1000 ? 'Great job! Your memory is impressive!' :
           'Good effort! Keep practicing to improve your score!'}
        </p>
        
        <div className="flex flex-col gap-2">
          {!scoreSubmitted && address && (
            // <Transaction 
            //   calls={scoreSubmitCalls}
            //   chainId={baseSepolia.id}
            
            //   onStatus={(status) => {
            //     if (status.statusName === 'success') {
            //       setScoreSubmitted(true);
            //       refreshData();
            //     }
            //   }}
            // />

            <button
              onClick={handleSubmitScore}
              className="bg-blue-600 text-white px-6 py-2 rounded-md hover:bg-blue-700"
            >
              Submit Score
            </button>
          )}
          
          {scoreSubmitted && (
            <div className="text-green-600 font-medium mb-2">
              Score submitted to leaderboard! 🎉
            </div>
          )}
          
          <button
            onClick={resetGame}
            className="bg-blue-600 text-white px-6 py-2 rounded-md hover:bg-blue-700"
          >
            Play Again
          </button>
        </div>
      </div>
    </div>
  );
};

export default GameComplete;
```

### 6. Update the Game Context

Let's update our GameContext to ensure it works with our blockchain components:

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

### 7. Update the Main App Component

Finally, let's update our main app component to include all the blockchain components:

```typescript
// app/page.tsx (update)
'use client';
import { useCallback, useEffect, useState } from 'react';
import { useAddFrame, useMiniKit, useOpenUrl } from '@coinbase/onchainkit/minikit';
import { GameProvider } from './contexts/GameContext';
import StartScreen from './components/StartScreen';
import GameBoard from './components/GameBoard';
import GameControls from './components/GameControls';
import GameComplete from './components/GameComplete';
import GameHistory from './components/GameHistory';
import WalletButton from './components/WalletButton';
import { BlockchainProvider } from './contexts/BlockchainContext';
import Leaderboard from './components/Leaderboard';
import PrizePoolForm from './components/PrizePoolForm';

export default function Home() {
  const { setFrameReady, isFrameReady, context } = useMiniKit();
  const [frameAdded, setFrameAdded] = useState(false);
  const addFrame = useAddFrame();
  const openUrl = useOpenUrl();
  
    // Initialize the frame
    useEffect(() => {
      if (!isFrameReady) {
        setFrameReady();
      }
    }, [setFrameReady, isFrameReady]);


  
  return (
    <BlockchainProvider>
      <GameProvider>
      <main className="flex min-h-screen min-w-screen flex-col items-center p-4 bg-white  ">
          <div className="w-full flex justify-between items-center mb-4">
            <h1 className="text-xl font-bold text-blue-600">Memory Match</h1>
            <div className="flex items-center gap-2">
              <WalletButton />
            </div>
          </div>
        
        <StartScreen />
        <GameBoard />
        <GameControls />
        <GameComplete />
        <div className="mt-8 w-full space-y-4">
            <Leaderboard />
            <PrizePoolForm />
          </div>
        
        <footer className="flex justify-center mt-6">
          <button 
            onClick={() => openUrl('https://base.org/builders/minikit')}
            className="text-xs opacity-60 px-2 py-1 border border-blue-300 text-blue-600 rounded-full"
          >
            BUILT WITH MINIKIT
          </button>
        </footer>
      </main>
    </GameProvider>
    </BlockchainProvider>
  );
}

```

### 2. Deploy to Vercel

```bash
# Push your changes to GitHub and deploy from Vercel
git add .
git commit -m "Add blockchain integration with OnChain Kit"
git push

# Or deploy directly from the CLI if you have Vercel CLI installed
vercel --prod
```

### 3. Test Your Deployed Application

1. Open your deployed application in Warpcast Frame Developer Tools
2. Connect your wallet using the wallet button
3. Play the game and submit your score to the leaderboard
4. Test adding to the prize pool
5. If you're the contract owner, test the "Award Prize" function

## Deploying the Contract to Base Sepolia

To deploy the contract to Base Sepolia, you can use Hardhat or Foundry. Here's a simple approach using Hardhat:

### 1. Set Up Hardhat Project

```bash
mkdir -p hardhat
cd hardhat
npm init -y
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox dotenv
npx hardhat init
```

Choose "Create a JavaScript project" when prompted.

### 2. Copy Contract to Hardhat Project

Copy the `MemoryMatchLeaderboard.sol` file to the `contracts` directory in your Hardhat project.

### 3. Create Deployment Script

Create a file `scripts/deploy.js` in your Hardhat project:

```javascript
// scripts/deploy.js
const { ethers } = require("hardhat");

async function main() {
  const [deployer] = await ethers.getSigners();
  console.log("Deploying contracts with the account:", deployer.address);

  const MemoryMatchLeaderboard = await ethers.getContractFactory("MemoryMatchLeaderboard");
  const memoryMatch = await MemoryMatchLeaderboard.deploy();

  await memoryMatch.deployed();
  console.log("MemoryMatchLeaderboard deployed to:", memoryMatch.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### 4. Update Hardhat Config for Base Sepolia

Update your `hardhat.config.js`:

```javascript
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

const PRIVATE_KEY = process.env.PRIVATE_KEY || "0x0000000000000000000000000000000000000000000000000000000000000000";

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: "0.8.20",
  networks: {
    hardhat: {},
    baseSepolia: {
      url: "https://sepolia.base.org",
      accounts: [PRIVATE_KEY],
      chainId: 84532,
      gasPrice: 1000000000,
    },
  },
};
```

### 5. Deploy to Base Sepolia

```bash
# Create a .env file with your private key
echo "PRIVATE_KEY=your_private_key_here" > .env

# Deploy to Base Sepolia
npx hardhat run scripts/deploy.js --network baseSepolia
```

## Conclusion and Next Steps

Congratulations! You've successfully integrated blockchain functionality into your Memory Match game using the latest OnChain Kit components. Let's recap what we've built:

1. A smart contract for storing scores and managing a prize pool
2. Modern wallet connectivity using OnChain Kit's wallet components
3. Blockchain transaction handling with the Transaction component
4. On-chain leaderboard with top scores
5. Prize pool contributions and award functionality
6. Seamless integration with the game mechanics

### What You've Learned

- How to connect a MiniKit app to blockchain using OnChain Kit
- Creating and deploying smart contracts on Base Sepolia
- Using modern Transaction components for blockchain interactions
- Working with wallet components for seamless user experience
- Building a full-stack dApp with gaming mechanics

### Next Steps for Your Project

Now that you have a working prototype, here are some ideas to enhance your project:

1. **Enhanced Game Mechanics**
   - Add difficulty levels (more cards, less time)
   - Implement different card themes (NFT collections, crypto logos)
   - Create power-ups that can be purchased with tokens

2. **Expanded Blockchain Features**
   - Implement SBTs (Soulbound Tokens) as achievements
   - Create a token economy with in-game rewards
   - Add NFT rewards for top players
   - Implement daily/weekly tournaments with prize pools

3. **Social Features**
   - Add friend challenges through Farcaster
   - Implement team competitions
   - Create achievement sharing via Frames

## MiniKit Workshop Session 2: Hands-On Exercise

Now, let's put what we've learned into practice with a hands-on exercise.

### Exercise: Implementing Daily Tournaments

In this exercise, you'll implement a daily tournament feature for the Memory Match game:

1. **Modify the Smart Contract**
   - Add a mapping to track daily tournaments
   - Create a function to start and end tournaments
   - Implement a prize distribution mechanism for multiple winners

2. **Update the UI**
   - Add a tournament banner showing current status
   - Create a tournament leaderboard
   - Implement a countdown timer for tournament end

3. **Integrate with MiniKit Features**
   - Use notifications to alert users of tournament start/end
   - Create a shareable frame for tournament results
   - Implement a tournament invitation system

## Troubleshooting Common Issues

Here are solutions to some common issues you might encounter:

1. **Transaction Errors**
   - Ensure you have enough testnet ETH for gas fees (get from Base Sepolia faucet)
   - Check that contract addresses are correct in your environment variables
   - Verify function parameters match the contract requirements

2. **Wallet Connection Issues**
   - Make sure you're using OnChain Kit components properly
   - Test with Coinbase Wallet which is fully compatible with Base
   - Check browser console for connection errors

3. **Contract Interaction Problems**
   - Verify ABI matches deployed contract
   - Ensure you're on the Base Sepolia network
   - Check for any contract function access control issues

## Resources and Documentation

- [Base MiniKit Documentation](https://docs.base.org/docs/building-with-base/mini-apps/mini-kit)
- [OnChain Kit Documentation](https://docs.base.org/docs/building-with-base/wallet/onchain-kit)
- [Coinbase Wallet SDK](https://docs.cloud.coinbase.com/wallet-sdk/docs)
- [Base Sepolia Faucet](https://www.coinbase.com/faucets/base-sepolia-faucet)
- [Warpcast Frames Developer Documentation](https://docs.farcaster.xyz/learn/what-is-farcaster/frames)

## Conclusion

Congratulations on completing Session 2 of our MiniKit Workshop! You've successfully transformed a simple browser game into a full-fledged web3 application with on-chain leaderboards, prize pools, and wallet integration.

By leveraging the power of Base MiniKit and OnChain Kit, you've created an engaging gaming experience that combines the accessibility of web games with the innovation of blockchain technology.

In the next session, we'll explore advanced MiniKit features including:
- Multi-step frames for complex user journeys
- Webhook integration for asynchronous processing
- Cross-app interactions between multiple Mini Apps

Keep building and exploring the possibilities of web3 gaming!
}
