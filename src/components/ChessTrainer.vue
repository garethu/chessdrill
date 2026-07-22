<script setup lang="ts">
import { ref, onMounted, nextTick } from 'vue';
import { Chess } from 'chess.js';
import { parse as parsePgn } from '@mliebelt/pgn-parser';
import { Chessground } from 'chessground';
import type { Api } from 'chessground/api';
import type { Key } from 'chessground/types';

interface Variation {
  name: string;
  moves: any[];
}

interface PgnLine {
  name: string;
  variations: Variation[];
}

const boardElement = ref<HTMLElement>();
let boardAPI: Api | null = null;
const chess = ref(new Chess());
const pgnContent = ref('');
const pgnLines = ref<PgnLine[]>([]);
const selectedLine = ref<PgnLine | null>(null);
const variations = ref<Variation[]>([]);
const selectedVariation = ref<Variation | null>(null);
const userColor = ref<'white' | 'black'>('white');
const currentMoveIndex = ref(0);
const trainingActive = ref(false);
const showCompletionDialog = ref(false);

const initBoard = () => {
  if (!boardElement.value) return;
  
  boardAPI = Chessground(boardElement.value, {
    orientation: userColor.value,
    movable: {
      free: false,
      color: 'both',
      showDests: true,
      events: {
        after: onMove
      }
    },
    draggable: {
      enabled: true,
      showGhost: true
    }
  });
  
  updateBoard();
};

const updateBoard = () => {
  if (!boardAPI) return;
  
  const fen = chess.value.fen();
  const currentTurn = chess.value.turn() === 'w' ? 'white' : 'black';
  
  boardAPI.set({
    fen: fen,
    turnColor: currentTurn,
    movable: {
      free: false,
      color: trainingActive.value ? currentTurn : 'both',
      dests: getLegalMoves(),
      showDests: true
    }
  });
};

const getLegalMoves = (): Map<Key, Key[]> => {
  const dests = new Map<Key, Key[]>();
  const moves = chess.value.moves({ verbose: true });
  
  moves.forEach(move => {
    const from = move.from as Key;
    const to = move.to as Key;
    if (!dests.has(from)) {
      dests.set(from, []);
    }
    dests.get(from)!.push(to);
  });
  
  return dests;
};

const onMove = (orig: Key, dest: Key) => {
  console.log('=== Move attempted ===');
  console.log('From:', orig, 'To:', dest);
  console.log('Training active:', trainingActive.value);
  
  if (!trainingActive.value || !selectedVariation.value) {
    console.log('Training not active - allowing free move');
    const result = chess.value.move({ from: orig, to: dest, promotion: 'q' });
    if (result) {
      updateBoard();
    }
    return;
  }
  
  if (currentMoveIndex.value >= selectedVariation.value.moves.length) {
    console.log('No more moves in variation');
    updateBoard();
    return;
  }
  
  const correctMove = selectedVariation.value.moves[currentMoveIndex.value];
  console.log('Expected move:', correctMove.from, '→', correctMove.to);
  console.log('User turn:', chess.value.turn(), 'User color:', userColor.value[0]);
  
  // Check if it's user's turn
  if (chess.value.turn() !== userColor.value[0]) {
    console.log("Not user's turn!");
    updateBoard();
    return;
  }
  
  // Check if move is correct
  const isCorrect = orig === correctMove.from && dest === correctMove.to;
  console.log('Move is correct:', isCorrect);
  
  if (isCorrect) {
    // Correct move
    const result = chess.value.move({ 
      from: orig, 
      to: dest, 
      promotion: correctMove.promotion || 'q' 
    });
    
    if (result) {
      currentMoveIndex.value++;
      updateBoard();
      console.log('✓ Correct move!');
      
      // Make opponent's move after delay
      if (currentMoveIndex.value < selectedVariation.value.moves.length) {
        if (chess.value.turn() !== userColor.value[0]) {
          setTimeout(() => {
            makeOpponentMove();
          }, 500);
        }
      } else {
        console.log('Variation completed!');
        showCompletionDialog.value = true;
        trainingActive.value = false;
      }
    }
  } else {
    // Wrong move - show arrows
    console.log('✗ Wrong move!');
    
    // Update board first to restore position
    if (boardAPI) {
      const fen = chess.value.fen();
      const currentTurn = chess.value.turn() === 'w' ? 'white' : 'black';
      const legalMoves = getLegalMoves();
      
      boardAPI.set({
        fen: fen,
        movable: {
          free: false,
          color: currentTurn,
          dests: legalMoves,
          showDests: true
        },
        turnColor: currentTurn
      });
      
      // Show red arrow for wrong move and green for correct
      boardAPI.setShapes([
        { orig: orig, dest: dest, brush: 'red' },
        { orig: correctMove.from as Key, dest: correctMove.to as Key, brush: 'green' }
      ]);
      
      // Clear arrows after 2 seconds
      setTimeout(() => {
        if (boardAPI) {
          boardAPI.setShapes([]);
        }
      }, 2000);
    }
  }
};

const makeOpponentMove = () => {
  if (!selectedVariation.value || currentMoveIndex.value >= selectedVariation.value.moves.length) {
    return;
  }
  
  const opponentMove = selectedVariation.value.moves[currentMoveIndex.value];
  console.log('Making opponent move:', opponentMove.san);
  
  const result = chess.value.move(opponentMove.san);
  if (result) {
    currentMoveIndex.value++;
    
    if (boardAPI) {
      const fen = chess.value.fen();
      const currentTurn = chess.value.turn() === 'w' ? 'white' : 'black';
      const legalMoves = getLegalMoves();
      
      console.log('After opponent move - FEN:', fen);
      console.log('Current turn:', currentTurn);
      console.log('Legal moves count:', legalMoves.size);
      
      boardAPI.set({
        fen: fen,
        movable: {
          free: false,
          color: currentTurn,
          dests: legalMoves,
          showDests: true
        },
        turnColor: currentTurn
      });
    }
    
    console.log('✓ Opponent moved:', opponentMove.san);
  }
};

const handlePgnUpload = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const file = target.files?.[0];
  if (!file) return;
  
  const reader = new FileReader();
  reader.onload = (e) => {
    pgnContent.value = e.target?.result as string;
    parseAndExtractVariations(pgnContent.value);
  };
  reader.readAsText(file);
};

const parseAndExtractVariations = (pgn: string) => {
  try {
    const parsedPgn = parsePgn(pgn, { startRule: 'games' }) as any[];
    console.log('Parsed PGN successfully:', parsedPgn.length, 'games');
    pgnLines.value = [];
    
    parsedPgn.forEach((game: any, index: number) => {
      try {
        const lineName = game.tags?.Event || `Line ${index + 1}`;
        console.log(`\n=== Processing: ${lineName} ===`);
        
        if (game.moves && game.moves.length > 0) {
          const gameChess = new Chess();
          const allMoves: any[] = [];
          
          // Recursive function to extract all moves in order
          const extractAllMoves = (moves: any[]): string[] => {
            const result: string[] = [];
            
            for (const moveEntry of moves) {
              // Each moveEntry represents a move pair or single move
              // Extract the notation for this move
              if (moveEntry.notation) {
                const notation = typeof moveEntry.notation === 'object' 
                  ? moveEntry.notation.notation 
                  : moveEntry.notation;
                if (notation) {
                  result.push(notation);
                }
              }
              
              // If there are nested moves (continuations), extract them too
              if (moveEntry.moves && moveEntry.moves.length > 0) {
                result.push(...extractAllMoves(moveEntry.moves));
              }
            }
            
            return result;
          };
          
          const moveNotations = extractAllMoves(game.moves);
          console.log('Extracted moves:', moveNotations);
          
          for (const sanMove of moveNotations) {
            try {
              const moveResult = gameChess.move(sanMove);
              if (moveResult) {
                allMoves.push(moveResult);
              } else {
                console.warn(`Failed to parse move "${sanMove}" at position: ${gameChess.fen()}`);
              }
            } catch (moveError) {
              console.warn(`Error parsing move "${sanMove}" at position: ${gameChess.fen()}`, moveError);
              // Continue with remaining moves
            }
          }
          
          if (allMoves.length > 0) {
            pgnLines.value.push({
              name: lineName,
              variations: [{
                name: allMoves.map((m: any) => m.san).join(' '),
                moves: allMoves
              }]
            });
            console.log(`✓ Loaded line: ${lineName} with ${allMoves.length} moves`);
          } else {
            console.warn(`✗ No valid moves found in line "${lineName}"`);
          }
        }
      } catch (gameError) {
        console.error(`Error processing game ${index}:`, gameError);
        // Continue with next game
      }
    });
    
    console.log('\n=== Summary ===');
    console.log('Successfully loaded', pgnLines.value.length, 'lines');
    
    if (pgnLines.value.length === 0) {
      alert('No valid games found in PGN file');
      return;
    }
    
    if (pgnLines.value.length === 1) {
      const firstLine = pgnLines.value[0];
      if (firstLine) {
        selectedLine.value = firstLine;
        variations.value = firstLine.variations;
        const firstVariation = variations.value[0];
        if (firstVariation) {
          selectedVariation.value = firstVariation;
        }
        console.log('Auto-selected single line:', firstLine.name);
      }
    }
  } catch (error) {
    console.error('Error parsing PGN:', error);
    console.error('PGN content:', pgn.substring(0, 200));
    alert('Error parsing PGN file. Check console for details.');
  }
};

const selectLine = (event: Event) => {
  const index = parseInt((event.target as HTMLSelectElement).value);
  const line = pgnLines.value[index];
  if (line) {
    selectedLine.value = line;
    variations.value = line.variations;
    const firstVariation = variations.value[0];
    if (firstVariation) {
      selectedVariation.value = firstVariation;
    }
  }
};

const startTraining = () => {
  if (!selectedVariation.value) {
    alert('Please load a PGN file first');
    return;
  }
  
  console.log('=== Starting Training ===');
  console.log('User color:', userColor.value);
  
  trainingActive.value = true;
  chess.value = new Chess();
  currentMoveIndex.value = 0;
  
  if (boardAPI) {
    boardAPI.setShapes([]);
  }
  
  updateBoard();
  
  // If user is black, make white's first move
  if (userColor.value === 'black' && selectedVariation.value.moves.length > 0) {
    console.log('User is Black - making White\'s first move');
    setTimeout(() => {
      makeOpponentMove();
    }, 500);
  }
};

const handleSideChange = (event: Event) => {
  userColor.value = (event.target as HTMLSelectElement).value as 'white' | 'black';
  if (boardAPI) {
    boardAPI.set({ orientation: userColor.value });
  }
  trainingActive.value = false;
  chess.value = new Chess();
  currentMoveIndex.value = 0;
  updateBoard();
};

const closeCompletionDialog = () => {
  showCompletionDialog.value = false;
  chess.value = new Chess();
  currentMoveIndex.value = 0;
  updateBoard();
};

const showHint = () => {
  if (!trainingActive.value || !selectedVariation.value) {
    alert('Start training first!');
    return;
  }
  
  if (currentMoveIndex.value >= selectedVariation.value.moves.length) {
    alert('No more moves');
    return;
  }
  
  const hintMove = selectedVariation.value.moves[currentMoveIndex.value];
  if (boardAPI) {
    boardAPI.setShapes([
      { orig: hintMove.from as Key, dest: hintMove.to as Key, brush: 'blue' }
    ]);
  }
};

onMounted(() => {
  nextTick(() => {
    initBoard();
  });
});
</script>

<template>
  <div class="chess-trainer-container">
    <h1>Chess Drill Trainer</h1>
    
    <div class="controls">
      <div class="pgn-loader">
        <label for="pgn-file">Load PGN:</label>
        <input type="file" id="pgn-file" accept=".pgn" @change="handlePgnUpload" />
      </div>
      
      <div class="line-selector" v-if="pgnLines.length > 1">
        <label for="line-select">Select Line:</label>
        <select id="line-select" @change="selectLine">
          <option value="" disabled selected>Select a line</option>
          <option v-for="(line, index) in pgnLines" :key="index" :value="index">
            {{ line.name }}
          </option>
        </select>
      </div>
      
      <div class="side-selector">
        <label for="side-select">Play as:</label>
        <select id="side-select" @change="handleSideChange" v-model="userColor">
          <option value="white">White</option>
          <option value="black">Black</option>
        </select>
      </div>
      
      <div class="action-buttons">
        <button @click="startTraining" :disabled="!selectedVariation">Train</button>
        <button @click="showHint" :disabled="!trainingActive">Hint</button>
      </div>
    </div>
    
    <div ref="boardElement" class="chessboard"></div>
    
    <!-- Completion Dialog -->
    <div v-if="showCompletionDialog" class="dialog-overlay" @click="closeCompletionDialog">
      <div class="dialog" @click.stop>
        <div class="dialog-icon">🎉</div>
        <h2>Congratulations!</h2>
        <p>You've completed the variation successfully!</p>
        <p class="dialog-line-name">{{ selectedLine?.name }}</p>
        <button @click="closeCompletionDialog" class="dialog-button">Continue Training</button>
      </div>
    </div>
  </div>
</template>

<style>
@import 'chessground/assets/chessground.base.css';
@import 'chessground/assets/chessground.brown.css';
@import 'chessground/assets/chessground.cburnett.css';
</style>

<style scoped>
.chess-trainer-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  color: #333;
  margin-bottom: 20px;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  margin-bottom: 20px;
  justify-content: center;
  width: 100%;
  max-width: 800px;
  background-color: #f0f0f0;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.pgn-loader,
.line-selector,
.side-selector {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

label {
  font-weight: bold;
  color: #555;
}

input[type="file"],
select,
button {
  padding: 8px 12px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  background-color: white;
}

button {
  background-color: #007bff;
  color: white;
  border: none;
}

button:hover {
  background-color: #0056b3;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.action-buttons {
  display: flex;
  gap: 10px;
  align-items: flex-end;
}

.chessboard {
  width: 512px;
  height: 512px;
}

.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.dialog {
  background: white;
  padding: 40px;
  border-radius: 16px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.3);
  text-align: center;
  max-width: 400px;
  animation: slideUp 0.3s ease;
}

@keyframes slideUp {
  from {
    transform: translateY(50px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.dialog-icon {
  font-size: 64px;
  margin-bottom: 20px;
  animation: bounce 0.6s ease;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-20px); }
}

.dialog h2 {
  color: #28a745;
  margin: 0 0 15px 0;
  font-size: 28px;
}

.dialog p {
  color: #555;
  margin: 10px 0;
  font-size: 16px;
}

.dialog-line-name {
  font-weight: bold;
  color: #333;
  font-size: 18px;
  margin: 20px 0;
}

.dialog-button {
  margin-top: 25px;
  padding: 12px 30px;
  font-size: 16px;
  background: linear-gradient(135deg, #28a745 0%, #20c997 100%);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  font-weight: bold;
}

.dialog-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(40, 167, 69, 0.4);
}

.dialog-button:active {
  transform: translateY(0);
}
</style>

