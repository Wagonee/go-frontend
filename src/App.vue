<template>
  <div id="go-game">
    <h1>Игра Го с поддержкой сложных очередей</h1>

    <div class="message-bar">
      <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
      <p v-if="successMessage" class="success-message">{{ successMessage }}</p>
      <p v-if="!errorMessage && !successMessage" class="status-message">{{ statusMessage }}</p>
    </div>

    <div v-if="!hasActiveGame" class="settings">
      <h2>Настройки новой игры</h2>

      <div class="options-grid">
        <label>
          Размер доски:
          <select v-model.number="options.board_size" :disabled="isLoading">
            <option value="5">5x5</option>
            <option value="7">7x7</option>
            <option value="9">9x9</option>
            <option value="13">13x13</option>
            <option value="19">19x19</option>
          </select>
        </label>

        <label>
          Тип очереди:
          <select v-model="options.queue_type" :disabled="isLoading">
            <option value="deterministic">Детерминированная</option>
            <option value="random">Случайная</option>
          </select>
        </label>

        <label v-if="options.queue_type === 'deterministic'">
          Шаблон очереди:
          <input type="text" v-model="options.queue_pattern" :disabled="isLoading" placeholder="BW">
        </label>

        <label v-if="options.queue_type === 'random'">
          Глубина очереди:
          <input type="number" v-model.number="options.queue_depth" :disabled="isLoading" min="1">
        </label>

        <!--        <label>-->
        <!--          Режим захвата:-->
        <!--          <select v-model="options.capture_mode" :disabled="isLoading" title="Камни какого цвета можно захватывать">-->
        <!--            <option value="both">Оба</option>-->
        <!--            <option value="white_only">Только Белые</option>-->
        <!--            <option value="black_only">Только Черные</option>-->
        <!--          </select>-->
        <!--        </label>-->

        <label>
          Одновременный захват:
          <select v-model="options.simultaneous_capture_rule" :disabled="isLoading"
                  title="Что происходит, если ход окружает группы обоих цветов одновременно">
            <option value="opponent">Снять противника (Классика)</option>
            <option value="both">Снять обоих</option>
            <option value="self">Снять себя</option>
          </select>
        </label>

        <label class="custom-checkbox">
          Отложенный захват:
          <input
              type="checkbox"
              v-model="options.delayed_capture"
              :disabled="isLoading"
              title="Захваченные камни снимаются перед следующим ходом"
          >
          <span class="checkmark"></span>
        </label>
      </div>

      <div class="pregame-state-setup">
        <h2>Начальная расстановка</h2>
        <div class="add-stone-form">
          <label>
            Цвет камня:
            <select v-model="setupStoneColor" :disabled="isLoading">
              <option value="black">Черный</option>
              <option value="white">Белый</option>
            </select>
          </label>
          <label>
            Строка:
            <input type="number" v-model.number="setupStoneRow" :disabled="isLoading" min="1" :max="options.board_size"
                   placeholder="R">
          </label>
          <label>
            Колонка:
            <input type="number" v-model.number="setupStoneCol" :disabled="isLoading" min="1" :max="options.board_size"
                   placeholder="C">
          </label>
          <button @click="addSetupStone" :disabled="isLoading" class="add-handicap-button">
            Добавить камень
          </button>
        </div>

        <div v-if="initialStonesSetup.length > 0" class="pregame-stones-list">
          <h3>Текущая фора:</h3>
          <ul>
            <li v-for="(stone, index) in initialStonesSetup" :key="`handicap-${index}`">
              <span>({{ stone.row }}, {{ stone.col }}) - {{ stone.color === 'black' ? 'Черный' : 'Белый' }}</span>
              <button @click="removeSetupStone(index)" :disabled="isLoading" class="remove-handicap-button">
                Удалить
              </button>
            </li>
          </ul>
        </div>
        <p v-else class="no-pregame-stones-msg">Фора не задана.</p>
      </div>
      <button class="start-button" @click="startGame" :disabled="isLoading || hasActiveGame">
        Начать новую игру {{ initialStonesSetup.length > 0 ? `с форой (${initialStonesSetup.length})` : '' }}
      </button>
    </div>

    <div v-if="hasActiveGame" class="game-area">
      <div class="game-info">
        <p>ID Игры: <code class="game-id-display">{{ gameId }}</code></p>
        <p>
          Захват: {{ options.capture_mode }}
          | Одновременно: {{ options.simultaneous_capture_rule }}
          {{ options.delayed_capture ? '(Отложенный)' : '(Немедленный)' }}
        </p>
      </div>

      <div class="controls">
        <button @click="passTurn" :disabled="isLoading || isGameOver">Пас</button>
        <button @click="resignGame" :disabled="isLoading || isGameOver">Сдаться</button>
        <button @click="deleteGame" :disabled="isLoading">Удалить/Сбросить Игру</button>
        <button @click="fetchGameState" :disabled="isLoading" title="Запросить состояние вручную">Обновить</button>
      </div>

      <div class="board-container">
        <div
            v-if="board.length > 0"
            class="go-board"
            :style="{
              gridTemplateColumns: `repeat(${boardSize}, 40px)`,
              gridTemplateRows: `repeat(${boardSize}, 40px)`,
              width: `${boardSize * 40}px`,
              height: `${boardSize * 40}px`
            }"
        >
          <template v-for="(row_cells, row_idx) in board" :key="'row-' + row_idx">
            <div
                v-for="(cell_state, col_idx) in row_cells"
                :key="'cell-' + row_idx + '-' + col_idx"
                class="intersection"
                @click="makeMove(row_idx + 1, col_idx + 1)"
                :class="{
                  'valid-hover': !isLoading && !isGameOver && cell_state === 'empty',
                  'black-stone': cell_state === 'black',
                  'white-stone': cell_state === 'white',
                  'last-move': lastMove && lastMove.row === (row_idx + 1) && lastMove.col === (col_idx + 1)
                }"
                :title="`(${row_idx + 1}, ${col_idx + 1})`"
            >
               <span
                   v-if="boardSize === 19 && [3, 9, 15].includes(row_idx) && [3, 9, 15].includes(col_idx)"
                   class="star-point"
               ></span>
              <span
                  v-else-if="boardSize === 13 && [3, 6, 9].includes(row_idx) && [3, 6, 9].includes(col_idx)"
                  class="star-point"
              ></span>
              <span
                  v-else-if="boardSize === 9 && [2, 6].includes(row_idx) && [2, 6].includes(col_idx)"
                  class="star-point"
              ></span>
              <span
                  v-else-if="boardSize >= 7 && boardSize % 2 !== 0
                   && row_idx === Math.floor(boardSize / 2)
                   && col_idx === Math.floor(boardSize / 2)"
                  class="star-point"
              ></span>
            </div>
          </template>
        </div>
        <p v-else-if="isLoading">Загрузка доски...</p>
        <p v-else-if="hasActiveGame && !isLoading">Не удалось загрузить доску. Попробуйте обновить.</p>
        <p v-else>Доска не загружена.</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import {ref, computed, watch} from 'vue';

const API_BASE_URL = 'http://localhost:8000';

const gameId = ref(null);

const options = ref({
  board_size: 9,
  queue_type: 'deterministic',
  queue_pattern: 'BW',
  queue_depth: 20,
  capture_mode: 'both',
  simultaneous_capture_rule: 'opponent',
  delayed_capture: false
});

const board = ref([]);
const nextPlayer = ref('');
const isGameOver = ref(false);
const winner = ref(null);
const lastMove = ref(null);
const boardSize = ref(options.value.board_size);

const isLoading = ref(false);
const errorMessage = ref('');
const successMessage = ref('');

const initialStonesSetup = ref([]);
const setupStoneColor = ref('black');
const setupStoneRow = ref(null);
const setupStoneCol = ref(null);

const hasActiveGame = computed(() => gameId.value !== null);

const statusMessage = computed(() => {
  if (isLoading.value) return 'Загрузка...';
  if (!hasActiveGame.value) return 'Настройте параметры и начните новую игру.';
  if (isGameOver.value) {
    return winner.value ? `Игра окончена! Победитель: ${winner.value}` : 'Игра окончена!';
  }
  return `Ход игрока: ${nextPlayer.value}`;
});


function clearMessages() {
  errorMessage.value = '';
  successMessage.value = '';
}

function setSuccessMessage(msg) {
  clearMessages();
  successMessage.value = msg;
  setTimeout(clearMessages, 3000);
}

function setErrorMessage(msg) {
  clearMessages();
  errorMessage.value = msg;
}

function addSetupStone() {
  clearMessages();
  const row = setupStoneRow.value;
  const col = setupStoneCol.value;
  const color = setupStoneColor.value;
  const currentBoardSize = options.value.board_size;

  if (!row || !col || row <= 0 || col <= 0 || row > currentBoardSize || col > currentBoardSize) {
    setErrorMessage(`Координаты (${row}, ${col}) вне доски ${currentBoardSize}x${currentBoardSize}.`);
    return;
  }

  const existingStone = initialStonesSetup.value.find(stone => stone.row === row && stone.col === col);
  if (existingStone) {
    setErrorMessage(`Точка (${row}, ${col}) уже занята камнем.`);
    return;
  }

  initialStonesSetup.value.push({row, col, color});
  console.log('Added pregame stone:', {row, col, color});
  setupStoneRow.value = null;
  setupStoneCol.value = null;
}

function removeSetupStone(index) {
  if (index >= 0 && index < initialStonesSetup.value.length) {
    const removed = initialStonesSetup.value.splice(index, 1);
    console.log('Removed handicap stone:', removed[0]);
  }
}

async function fetchGameState() {
  if (!gameId.value) {
    console.warn("fetchGameState called without active gameId");
    return;
  }
  isLoading.value = true;
  clearMessages();
  try {
    const response = await fetch(`${API_BASE_URL}/game/${gameId.value}/state`);
    if (!response.ok) {
      if (response.status === 404) {
        setErrorMessage(`Игра ${gameId.value} не найдена на сервере.`);
        gameId.value = null;
      } else {
        const errorData = await response.json().catch(() => ({detail: `HTTP error! status: ${response.status}`}));
        throw new Error(errorData.detail || `HTTP error! status: ${response.status}`);
      }
      return;
    }
    const data = await response.json();
    board.value = data.board;
    nextPlayer.value = data.next_player;
    isGameOver.value = data.is_over;
    winner.value = data.winner;
    lastMove.value = data.last_move;
    boardSize.value = data.board_size;
    options.value.capture_mode = data.capture_mode;
    options.value.simultaneous_capture_rule = data.simultaneous_capture_rule;
    options.value.delayed_capture = data.delayed_capture;
    options.value.board_size = data.board_size;
    options.value.queue_type = data.queue_type;
    console.log("Game state updated:", data);
  } catch (error) {
    console.error("Ошибка при получении состояния игры:", error);
    setErrorMessage(`Не удалось загрузить состояние игры: ${error.message}`);
  } finally {
    isLoading.value = false;
  }
}

async function startGame() {
  if (isLoading.value || hasActiveGame.value) return;
  isLoading.value = true;
  clearMessages();
  try {
    const currentBoardSize = options.value.board_size;
    const validatedInitialStones = initialStonesSetup.value.filter(stone =>
        stone.row > 0 && stone.row <= currentBoardSize &&
        stone.col > 0 && stone.col <= currentBoardSize
    );

    if (validatedInitialStones.length !== initialStonesSetup.value.length) {
      console.warn("Некоторые камни предигровой расстановки были вне доски и проигнорированы при старте.");
      setErrorMessage("Часть камней предигровой расстановки была вне доски и проигнорирована.");
    }

    const startGameConfig = {
      board_size: options.value.board_size,
      initial_stones: validatedInitialStones,
      queue_type: options.value.queue_type,
      queue_pattern: options.value.queue_type === 'deterministic' ? options.value.queue_pattern : undefined,
      queue_depth: options.value.queue_type === 'random' ? options.value.queue_depth : undefined,
      capture_mode: options.value.capture_mode,
      simultaneous_capture_rule: options.value.simultaneous_capture_rule,
      delayed_capture: options.value.delayed_capture,
    };

    console.log("Starting game with config:", startGameConfig);
    const response = await fetch(`${API_BASE_URL}/start`, {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify(startGameConfig),
    });
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({detail: `HTTP error! status: ${response.status}`}));
      throw new Error(errorData.detail || `HTTP error! status: ${response.status}`);
    }
    const data = await response.json();
    console.log("Игра создана на сервере:", data);
    gameId.value = data.game_id;
    await fetchGameState();
    setSuccessMessage(`Новая игра ${gameId.value} начата!`);
  } catch (error) {
    console.error("Ошибка при старте игры:", error);
    setErrorMessage(`Не удалось начать игру: ${error.message}`);
    gameId.value = null;
  } finally {
    isLoading.value = false;
  }
}

async function makeMove(row, col) {
  const cellState = board.value[row - 1]?.[col - 1];
  const cellIsEmpty = cellState === 'empty';

  if (!gameId.value || isLoading.value || isGameOver.value || !cellIsEmpty) {
    console.warn("Make move blocked:", {
      gameId: gameId.value,
      isLoading: isLoading.value,
      isGameOver: isGameOver.value,
      cellIsEmpty,
      cellState
    });
    if (!hasActiveGame.value) {
      setErrorMessage('Сначала начните игру!');
    } else if (cellState === undefined) {
      setErrorMessage('Неверные координаты или доска не загружена.');
    } else if (cellState !== 'empty') {
      setErrorMessage('Эта клетка уже занята!');
    } else if (isLoading.value) {
      setErrorMessage('Подождите, идет загрузка...');
    } else if (isGameOver.value) {
      setErrorMessage('Игра уже окончена!');
    } else {
      setErrorMessage('Действие недоступно!'); // Общая ошибка
    }
    setTimeout(clearMessages, 2000);
    return;
  }

  isLoading.value = true;
  clearMessages();
  try {
    const response = await fetch(`${API_BASE_URL}/game/${gameId.value}/play`, {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({row: row, col: col}),
    });
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({detail: `HTTP error! status: ${response.status}`}));
      throw new Error(errorData.detail || `HTTP error! status: ${response.status}`);
    }
    const data = await response.json();
    console.log("Ход выполнен:", data.status);
    await fetchGameState();
  } catch (error) {
    console.error("Ошибка при совершении хода:", error);
    setErrorMessage(`Ошибка хода: ${error.message}`);
    await fetchGameState();
  } finally {
    isLoading.value = false;
  }
}

async function passTurn() {
  if (!gameId.value || isLoading.value || isGameOver.value) return;
  isLoading.value = true;
  clearMessages();
  try {
    const response = await fetch(`${API_BASE_URL}/game/${gameId.value}/pass`, {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({}),
    });
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({detail: `HTTP error! status: ${response.status}`}));
      throw new Error(errorData.detail || `HTTP error! status: ${response.status}`);
    }
    const data = await response.json();
    console.log("Пас выполнен:", data.status);
    await fetchGameState();
    setSuccessMessage(data.status);
  } catch (error) {
    console.error("Ошибка при пасе:", error);
    setErrorMessage(`Ошибка паса: ${error.message}`);
    await fetchGameState();
  } finally {
    isLoading.value = false;
  }
}

async function resignGame() {
  if (!gameId.value || isLoading.value || isGameOver.value) return;
  if (!confirm('Вы уверены, что хотите сдаться?')) {
    return;
  }
  isLoading.value = true;
  clearMessages();
  try {
    const response = await fetch(`${API_BASE_URL}/game/${gameId.value}/resign`, {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({}),
    });
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({detail: `HTTP error! status: ${response.status}`}));
      throw new Error(errorData.detail || `HTTP error! status: ${response.status}`);
    }
    const data = await response.json();
    console.log("Игрок сдался:", data.status);
    await fetchGameState();
    setSuccessMessage(data.status);
  } catch (error) {
    console.error("Ошибка при сдаче:", error);
    setErrorMessage(`Ошибка: ${error.message}`);
    await fetchGameState();
  } finally {
    isLoading.value = false;
  }
}

async function deleteGame() {
  if (!gameId.value) return;
  if (!confirm(`Вы уверены, что хотите удалить (сбросить) игру ${gameId.value}? Это действие необратимо.`)) return;

  isLoading.value = true;
  clearMessages();
  const idToDelete = gameId.value;

  try {
    const response = await fetch(`${API_BASE_URL}/game/${idToDelete}`, {method: 'DELETE'});
    if (!response.ok && response.status !== 204) {
      let errorDetail = `HTTP error! status: ${response.status}`;
      try {
        const errorData = await response.json();
        errorDetail = errorData.detail || errorDetail;
      } catch (_) {
      }
      throw new Error(errorDetail);
    }

    console.log("Игра", idToDelete, "удалена на сервере.");
    gameId.value = null;
    options.value = {
      board_size: 9,
      queue_type: 'deterministic',
      queue_pattern: 'BW',
      queue_depth: 20,
      capture_mode: 'both',
      simultaneous_capture_rule: 'opponent',
      delayed_capture: false
    };
    boardSize.value = options.value.board_size;
    initialStonesSetup.value = [];
    setSuccessMessage(`Игра ${idToDelete} успешно удалена/сброшена.`);

  } catch (error) {
    console.error("Ошибка при удалении игры:", error);
    setErrorMessage(`Не удалось удалить игру: ${error.message}`);
  } finally {
    isLoading.value = false;
  }
}

watch(gameId, (newId, oldId) => {
  console.log(`FRONTEND: Game ID changed from ${oldId} to ${newId}`);
  if (!newId) {
    board.value = [];
    nextPlayer.value = '';
    isGameOver.value = false;
    winner.value = null;
    lastMove.value = null;
    boardSize.value = options.value.board_size;
    initialStonesSetup.value = [];
    clearMessages();
    console.log('Frontend game state reset.');
  }
});

watch(() => options.value.board_size, (newSize) => {
  if (!hasActiveGame.value) {
    boardSize.value = newSize;
    if (initialStonesSetup.value.length > 0) {
      console.log("Board size changed in settings, clearing handicap stones.");
      initialStonesSetup.value = [];
      setSuccessMessage("Камни предигровой расстановки очищены из-за смены размера доски.");
    }
  }
});
</script>

<style>
body {
  margin: 0;
  padding: 0;
  background-color: #f4f0e8;
  font-family: sans-serif;
  color: #333;
}

#go-game {
  max-width: 900px;
  margin: 20px auto;
  padding: 20px;
  background-color: #f4f0e8;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 25px;
}

h1 {
  color: #4a3a2a;
  text-align: center;
  margin-bottom: 10px;
}

h2 {
  color: #5a4a3a;
  text-align: center;
  margin-bottom: 15px;
  border-bottom: 1px solid #d1b78b;
  padding-bottom: 5px;
}


.settings {
  width: 100%;
  max-width: 600px;
  padding: 20px;
  border: 1px solid #d1b78b;
  border-radius: 6px;
  background-color: #fffaf0;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
}

.options-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 18px;
  margin-bottom: 25px;
}

.options-grid label {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 0.95em;
  color: #5a4a3a;
}

.options-grid select,
.options-grid input[type="text"],
.options-grid input[type="number"] {
  padding: 8px 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1em;
}

.options-grid label.custom-checkbox {
  flex-direction: row;
  align-items: center;
  gap: 8px;
}

.custom-checkbox {
  font-size: 0.95em;
  color: #5a4a3a;
  cursor: pointer;
  user-select: none;
  position: relative;
}

.custom-checkbox input {
  position: absolute;
  opacity: 0;
  cursor: pointer;
  height: 0;
  width: 0;
}

.custom-checkbox .checkmark {
  height: 20px;
  width: 20px;
  background-color: #eee;
  border: 2px solid #ccc;
  border-radius: 4px;
  transition: background-color 0.3s, border-color 0.3s;
  position: relative;
}

.custom-checkbox input:checked ~ .checkmark {
  background-color: #5cb85c;
  border-color: #5cb85c;
}

.custom-checkbox .checkmark:after {
  content: "";
  position: absolute;
  display: none;
}

.custom-checkbox input:checked ~ .checkmark:after {
  display: block;
}

.custom-checkbox .checkmark:after {
  left: 6px;
  top: 2px;
  width: 5px;
  height: 10px;
  border: solid white;
  border-width: 0 2px 2px 0;
  transform: rotate(45deg);
}

.start-button {
  display: block;
  width: 100%;
  padding: 12px 15px;
  font-size: 1.1em;
  font-weight: bold;
  border: none;
  border-radius: 5px;
  background-color: #5cb85c;
  color: white;
  cursor: pointer;
  transition: background-color 0.2s;
  margin-top: 10px;
  position: relative;
}

.start-button:hover:not(:disabled) {
  background-color: #4cae4c;
}

.start-button:hover:not(:disabled):after {
  content: " ➜";
  animation: rocket 0.5s ease-in-out;
}

@keyframes rocket {
  0% {
    transform: translateX(0);
    opacity: 0;
  }
  50% {
    opacity: 1;
  }
  100% {
    transform: translateX(10px);
    opacity: 0;
  }
}

.start-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.game-area {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.game-info {
  font-size: 0.9em;
  color: #555;
  text-align: center;
  background: #eee;
  padding: 5px 10px;
  border-radius: 4px;
}

.game-info p {
  margin: 3px 0;
}

.game-id-display {
  background: #ddd;
  padding: 2px 4px;
  border-radius: 3px;
  font-family: monospace;
}

.message-bar {
  min-height: 2em;
  width: 100%;
  margin-bottom: 0;
  padding: 0;
  text-align: center;
}

.status-message {
  font-size: 1.1em;
  font-weight: bold;
  color: #4a3a2a;
  padding: 10px 0;
}

.error-message,
.success-message {
  padding: 10px;
  border-radius: 4px;
  font-weight: bold;
  margin: 5px 0;
}

.error-message {
  background-color: #f2dede;
  color: #a94442;
  border: 1px solid #ebccd1;
}

.success-message {
  background-color: #dff0d8;
  color: #3c763d;
  border: 1px solid #d6e9c6;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  justify-content: center;
}

.controls button {
  padding: 9px 16px;
  font-size: 1em;
  border: none;
  border-radius: 5px;
  background-color: #8b6e4b;
  color: white;
  cursor: pointer;
  transition: background-color 0.2s;
}

.controls button:hover:not(:disabled) {
  background-color: #6a4e2b;
}

.controls button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.board-container {
  padding: 20px;
  background-color: #d1b78b;
  border-radius: 4px;
  box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.1);
  display: inline-block;
}

.go-board {
  display: grid;
  border: 2px solid #4a3a2a;
  background-color: #d1b78b;
  position: relative;
  background-image: linear-gradient(to right, #5a4a3a 1px, transparent 1px), linear-gradient(to bottom, #5a4a3a 1px, transparent 1px);
  background-size: 40px 40px;
  background-position: -1px -1px;
}

.intersection {
  width: 40px;
  height: 40px;
  box-sizing: border-box;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  cursor: default;
}

.intersection::before {
  content: '';
  display: block;
  width: 85%;
  height: 85%;
  border-radius: 50%;
  box-shadow: 1px 1px 3px rgba(0, 0, 0, 0.4);
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  opacity: 0;
  transition: opacity 0.2s, background-color 0.2s;
  z-index: 1;
}

.intersection.black-stone::before {
  background-color: #222;
  opacity: 1;
}

.intersection.white-stone::before {
  background-color: #fdfdfd;
  border: 1px solid #bbb;
  box-sizing: border-box;
  opacity: 1;
}

.intersection.last-move::after {
  content: '';
  position: absolute;
  width: 30%;
  height: 30%;
  background-color: rgba(255, 0, 0, 0.6);
  border-radius: 50%;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
  z-index: 2;
}

.star-point {
  position: absolute;
  width: 6px;
  height: 6px;
  background-color: #5a4a3a;
  border-radius: 50%;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
  z-index: 0;
}

.intersection.black-stone .star-point,
.intersection.white-stone .star-point {
  display: none;
}

.pregame-state-setup {
  margin-top: 30px;
  padding-top: 20px;
  border-top: 2px dashed #d1b78b;
}

.pregame-state-setup h2 {
  margin-bottom: 20px;
  color: #6a4e2b;
}

.add-stone-form {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  align-items: flex-end;
  margin-bottom: 20px;
  background-color: #fdf5e6;
  padding: 15px;
  border-radius: 4px;
}

.add-stone-form label {
  display: flex;
  flex-direction: column;
  gap: 5px;
  font-size: 0.9em;
}

.add-stone-form select,
.add-stone-form input[type="number"] {
  padding: 6px 8px;
  border: 1px solid #ccc;
  border-radius: 3px;
  font-size: 0.95em;
  width: 70px;
}

.add-handicap-button {
  padding: 8px 12px;
  font-size: 0.95em;
  border: none;
  border-radius: 4px;
  background-color: #4682B4;
  color: white;
  cursor: pointer;
  transition: background-color 0.2s;
  height: fit-content;
}

.add-handicap-button:hover:not(:disabled) {
  background-color: #4175a1;
}

.add-handicap-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.pregame-stones-list {
  margin-top: 15px;
}

.pregame-stones-list h3 {
  font-size: 1em;
  color: #5a4a3a;
  margin-bottom: 10px;
}

.pregame-stones-list ul {
  list-style: none;
  padding: 0;
  margin: 0;
  max-height: 150px;
  overflow-y: auto;
  border: 1px solid #eee;
  border-radius: 4px;
  padding: 5px;
  background-color: #f8f8f8;
}

.pregame-stones-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 10px;
  background-color: #fff;
  border-bottom: 1px solid #eee;
  font-size: 0.9em;
}

.pregame-stones-list li:last-child {
  border-bottom: none;
}

.remove-handicap-button {
  padding: 3px 8px;
  font-size: 0.85em;
  border: none;
  border-radius: 3px;
  background-color: #d9534f;
  color: white;
  cursor: pointer;
  transition: background-color 0.2s;
}

.remove-handicap-button:hover:not(:disabled) {
  background-color: #c9302c;
}

.remove-handicap-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.no-pregame-stones-msg {
  font-style: italic;
  color: #777;
  margin-top: 10px;
  font-size: 0.9em;
  text-align: center;
}

.intersection.valid-hover:hover::before {
  background-color: rgba(0, 0, 0, 0.1);
  opacity: 1;
  cursor: pointer;
}


.intersection.black-stone,
.intersection.white-stone {
  cursor: not-allowed;
}


.intersection.black-stone:hover::before,
.intersection.white-stone:hover::before {
  cursor: not-allowed;
}

</style>