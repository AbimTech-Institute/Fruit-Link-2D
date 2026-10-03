const FRUITS = [
  { key: 'apple', icon: '🍎' },
  { key: 'banana', icon: '🍌' },
  { key: 'grape', icon: '🍇' },
  { key: 'orange', icon: '🍊' },
  { key: 'strawberry', icon: '🍓' },
  { key: 'watermelon', icon: '🍉' },
  { key: 'pear', icon: '🍐' },
  { key: 'peach', icon: '🍑' },
  { key: 'kiwi', icon: '🥝' },
  { key: 'pineapple', icon: '🍍' }
];

const state = {
  level: 1,
  score: 0,
  timeLeft: 120,
  board: [],
  selected: null,
  pairsLeft: 0,
  timerId: null,
  gameStarted: false,
  boardSize: { rows: 5, cols: 6 }
};

const boardEl = document.getElementById('board');
const scoreLabel = document.getElementById('score-label');
const levelLabel = document.getElementById('level-label');
const timeLabel = document.getElementById('time-label');
const pairsLabel = document.getElementById('pairs-label');
const messageBox = document.getElementById('message-box');
const restartBtn = document.getElementById('restart-btn');
const startBtn = document.getElementById('start-btn');
const adBox = document.getElementById('ad-box');
const adText = document.getElementById('ad-text');
const connectionLine = document.getElementById('connection-line');

function startGame() {
  clearInterval(state.timerId);
  state.level = 1;
  state.score = 0;
  state.timeLeft = 120;
  state.gameStarted = true;
  renderHUD();
  buildLevel();
  startTimer();
  setMessage('Level 1 is live. Link matching fruits before time runs out!');
  showCrazyGamesAd('Challenge loaded. New fruit wave arriving...');
}

function buildLevel() {
  const rows = 5 + Math.min(state.level, 3);
  const cols = 6 + Math.min(state.level, 3);
  const fruitTypes = FRUITS.slice(0, Math.min(4 + state.level, FRUITS.length));
  const boardArray = [];

  fruitTypes.forEach((fruit) => {
    boardArray.push({ key: fruit.key, icon: fruit.icon });
    boardArray.push({ key: fruit.key, icon: fruit.icon });
  });

  const neededPairs = Math.min(Math.max(8, 10 + state.level * 2), Math.floor((rows * cols) / 2));
  const finalBoard = [];

  while (finalBoard.length < rows * cols) {
    if (boardArray.length === 0) {
      break;
    }
    const pickIndex = Math.floor(Math.random() * boardArray.length);
    finalBoard.push(boardArray.splice(pickIndex, 1)[0]);
  }

  while (finalBoard.length < rows * cols) {
    finalBoard.push(null);
  }

  const shuffled = shuffle(finalBoard).slice(0, rows * cols);
  state.board = shuffled;
  state.boardSize = { rows, cols };
  state.selected = null;
  state.pairsLeft = state.board.filter(Boolean).length / 2;

  renderBoard();
  renderHUD();
}

function renderBoard() {
  boardEl.style.gridTemplateColumns = `repeat(${state.boardSize.cols}, minmax(0, 1fr))`;
  boardEl.style.gridTemplateRows = `repeat(${state.boardSize.rows}, minmax(0, 1fr))`;
  boardEl.innerHTML = '';

  state.board.forEach((cell, index) => {
    const tile = document.createElement('button');
    tile.type = 'button';
    tile.className = 'cell';
    tile.dataset.index = String(index);
    tile.setAttribute('aria-label', cell ? `${cell.key} fruit tile` : 'empty tile');

    if (cell) {
      tile.textContent = cell.icon;
      tile.addEventListener('click', () => handleTileClick(index));
    } else {
      tile.disabled = true;
      tile.style.visibility = 'hidden';
    }

    if (state.selected === index) {
      tile.classList.add('selected');
    }

    boardEl.appendChild(tile);
  });
}

function handleTileClick(index) {
  if (!state.gameStarted || state.board[index] === null) {
    return;
  }

  const tile = boardEl.querySelector(`[data-index="${index}"]`);

  if (!tile) {
    return;
  }

  if (state.selected === null) {
    state.selected = index;
    renderBoard();
    setMessage('Fruit selected. Choose a matching fruit to link.');
    return;
  }

  if (state.selected === index) {
    state.selected = null;
    renderBoard();
    return;
  }

  const first = state.board[state.selected];
  const second = state.board[index];

  if (!first || !second) {
    state.selected = null;
    renderBoard();
    return;
  }

  if (first.key === second.key) {
    connectFruitPair(state.selected, index);
    state.selected = null;
    renderBoard();
    return;
  }

  state.selected = index;
  renderBoard();
  setMessage('Not a match. Try linking two identical fruits.');
}

function connectFruitPair(a, b) {
  const firstCell = boardEl.querySelector(`[data-index="${a}"]`);
  const secondCell = boardEl.querySelector(`[data-index="${b}"]`);

  if (!firstCell || !secondCell) {
    return;
  }

  const boardRect = boardEl.getBoundingClientRect();
  const firstRect = firstCell.getBoundingClientRect();
  const secondRect = secondCell.getBoundingClientRect();

  const startX = firstRect.left - boardRect.left + firstRect.width / 2;
  const startY = firstRect.top - boardRect.top + firstRect.height / 2;
  const endX = secondRect.left - boardRect.left + secondRect.width / 2;
  const endY = secondRect.top - boardRect.top + secondRect.height / 2;

  const length = Math.hypot(endX - startX, endY - startY);
  const angle = Math.atan2(endY - startY, endX - startX) * (180 / Math.PI);

  connectionLine.style.opacity = '1';
  connectionLine.style.width = `${length}px`;
  connectionLine.style.left = `${startX}px`;
  connectionLine.style.top = `${startY}px`;
  connectionLine.style.transform = `rotate(${angle}deg)`;

  setTimeout(() => {
    connectionLine.style.opacity = '0';
  }, 200);

  state.board[a] = null;
  state.board[b] = null;

  const pairBonus = 80 + state.level * 35;
  state.score += pairBonus;
  state.pairsLeft = state.board.filter(Boolean).length / 2;
  renderHUD();

  const remaining = state.board.filter(Boolean).length;
  if (remaining === 0) {
    clearInterval(state.timerId);
    const levelBonus = 250 + state.timeLeft * 3;
    state.score += levelBonus;
    renderHUD();
    setMessage(`Level ${state.level} cleared! Great run. Loading next wave...`);
    showCrazyGamesAd('Next fruit sequence unlocked. Watch this sponsored break!');

    setTimeout(() => {
      state.level += 1;
      state.timeLeft = Math.max(60, 120 - (state.level - 1) * 6);
      buildLevel();
      startTimer();
      setMessage(`Level ${state.level} started. Keep the combo alive!`);
    }, 1500);
  } else {
    setMessage(`Nice match! ${remaining / 2} fruit pairs left.`);
  }

  renderBoard();
}

function renderHUD() {
  levelLabel.textContent = String(state.level);
  scoreLabel.textContent = String(state.score);
  timeLabel.textContent = String(Math.max(0, state.timeLeft));
  pairsLabel.textContent = String(Math.max(0, state.pairsLeft));
}

function startTimer() {
  clearInterval(state.timerId);
  state.timerId = setInterval(() => {
    if (!state.gameStarted) {
      return;
    }

    state.timeLeft -= 1;
    renderHUD();

    if (state.timeLeft <= 0) {
      clearInterval(state.timerId);
      state.gameStarted = false;
      setMessage('Time is up! Press Restart to try again.');
      showCrazyGamesAd('Game over. A new challenge will open soon.');
    }
  }, 1000);
}

function setMessage(message) {
  messageBox.textContent = message;
}

function shuffle(array) {
  const clone = [...array];
  for (let i = clone.length - 1; i > 0; i -= 1) {
    const j = Math.floor(Math.random() * (i + 1));
    [clone[i], clone[j]] = [clone[j], clone[i]];
  }
  return clone;
}

function showCrazyGamesAd(message) {
  adText.textContent = message;
  adBox.style.transform = 'scale(1.03)';
  adBox.style.boxShadow = '0 12px 26px rgba(255, 183, 3, 0.2)';

  if (window.CrazyGames && window.CrazyGames.SDK && window.CrazyGames.SDK.ad) {
    try {
      window.CrazyGames.SDK.ad.requestAd();
    } catch (error) {
      console.warn('CrazyGames ad request failed:', error);
    }
  }

  setTimeout(() => {
    adBox.style.transform = 'scale(1)';
    adBox.style.boxShadow = 'none';
  }, 1000);
}

restartBtn.addEventListener('click', startGame);
startBtn.addEventListener('click', startGame);

window.addEventListener('resize', () => {
  if (state.selected !== null) {
    renderBoard();
  }
});

renderHUD();
setMessage('Press Start Game to begin your fruit run.');
showCrazyGamesAd('Welcome to Fruit Link 2D. Sponsored challenge loading...');
