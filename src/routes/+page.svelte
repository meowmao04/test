<script>
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	// --- Game Constants ---
	const COLS = 10;
	const ROWS = 20;
	const BLOCK_SIZE = 40; // In pixels

	// --- Tetromino Shapes and Colors ---
	const TETROMINOES = {
		'I': { shape: [[1, 1, 1, 1]], color: 'bg-cyan-500' },
		'J': { shape: [[0, 1, 0], [0, 1, 0], [1, 1, 0]], color: 'bg-blue-600' },
		'L': { shape: [[0, 1, 0], [0, 1, 0], [0, 1, 1]], color: 'bg-orange-500' },
		'O': { shape: [[1, 1], [1, 1]], color: 'bg-yellow-400' },
		'S': { shape: [[0, 1, 1], [1, 1, 0], [0, 0, 0]], color: 'bg-green-500' },
		'T': { shape: [[1, 1, 1], [0, 1, 0], [0, 0, 0]], color: 'bg-purple-600' },
		'Z': { shape: [[1, 1, 0], [0, 1, 1], [0, 0, 0]], color: 'bg-red-600' }
	};
	const TETROMINO_KEYS = Object.keys(TETROMINOES);

	// --- Game State Variables ---
	let board = createEmptyBoard();
	let currentPiece;
	let currentPosition;
	let score = 0;
	let gameOver = false;
	let gameStarted = false;
	let gameInterval;

	// --- Core Game Functions ---

	/**
	 * Creates an empty game board grid.
	 * @returns {Array<Array<{value: number, color: string}>>} The empty board.
	 */
	function createEmptyBoard() {
		return Array.from({ length: ROWS }, () => Array(COLS).fill({ value: 0, color: 'bg-gray-800' }));
	}

	/**
	 * Selects a new random tetromino.
	 */
	function selectNewPiece() {
		const key = TETROMINO_KEYS[Math.floor(Math.random() * TETROMINO_KEYS.length)];
		const pieceData = TETROMINOES[key];
		currentPiece = {
			shape: pieceData.shape.map(row => [...row]), // Deep copy shape
			color: pieceData.color
		};
		currentPosition = {
			x: Math.floor(COLS / 2) - Math.floor(currentPiece.shape[0].length / 2),
			y: 0
		};

		// Game over condition
		if (!isValidMove(currentPiece.shape, currentPosition)) {
			gameOver = true;
			gameStarted = false;
			clearInterval(gameInterval);
		}
	}

	/**
	 * Checks if a move is valid (within bounds and not colliding).
	 * @param {Array<Array<number>>} shape - The piece's shape matrix.
	 * @param {{x: number, y: number}} pos - The piece's top-left position.
	 * @returns {boolean} True if the move is valid, false otherwise.
	 */
	function isValidMove(shape, pos) {
		for (let y = 0; y < shape.length; y++) {
			for (let x = 0; x < shape[y].length; x++) {
				if (shape[y][x] !== 0) {
					const newX = pos.x + x;
					const newY = pos.y + y;

					// Check boundaries
					if (newX < 0 || newX >= COLS || newY >= ROWS) {
						return false;
					}
					// Check collision with existing blocks
					if (newY >= 0 && board[newY][newX].value !== 0) {
						return false;
					}
				}
			}
		}
		return true;
	}

	/**
	 * "Stamps" the current piece onto the board when it lands.
	 */
	function placePiece() {
		const { shape, color } = currentPiece;
		shape.forEach((row, y) => {
			row.forEach((value, x) => {
				if (value !== 0) {
					const boardX = currentPosition.x + x;
					const boardY = currentPosition.y + y;
					if (boardY >= 0) {
						board[boardY][boardX] = { value: 1, color };
					}
				}
			});
		});
	}

	/**
	 * Clears completed lines from the board and updates the score.
	 */
	function clearLines() {
		let linesCleared = 0;
		for (let y = ROWS - 1; y >= 0; y--) {
			if (board[y].every(cell => cell.value !== 0)) {
				linesCleared++;
				// Remove the full row
				board.splice(y, 1);
				// Add a new empty row at the top
				board.unshift(Array(COLS).fill({ value: 0, color: 'bg-gray-800' }));
				// Since we removed a row, we need to check the same y index again
				y++;
			}
		}
		// Update score based on lines cleared
		if (linesCleared > 0) {
			score += linesCleared * 100 * linesCleared; // Bonus for multi-line clears
		}
	}

	/**
	 * Rotates the current piece.
	 */
	function rotatePiece() {
		if (gameOver || !currentPiece) return;
		const shape = currentPiece.shape;
		const n = shape.length;
		const newShape = Array.from({ length: n }, () => Array(n).fill(0));

		for (let y = 0; y < n; y++) {
			for (let x = 0; x < n; x++) {
				newShape[x][n - 1 - y] = shape[y][x];
			}
		}

		if (isValidMove(newShape, currentPosition)) {
			currentPiece.shape = newShape;
		}
	}

	/**
	 * Moves the piece horizontally or vertically.
	 * @param {number} dx - Change in x.
	 * @param {number} dy - Change in y.
	 */
	function move(dx, dy) {
		if (gameOver || !currentPosition) return;
		const newPosition = { x: currentPosition.x + dx, y: currentPosition.y + dy };
		if (isValidMove(currentPiece.shape, newPosition)) {
			currentPosition = newPosition;
			return true;
		}
		return false;
	}

	/**
	 * The main game loop function.
	 */
	function gameLoop() {
		if (!move(0, 1)) {
			// Piece has landed
			placePiece();
			clearLines();
			selectNewPiece();
		}
	}

	/**
	 * Starts or restarts the game.
	 */
	function startGame() {
		board = createEmptyBoard();
		score = 0;
		gameOver = false;
		gameStarted = true;
		selectNewPiece();
		clearInterval(gameInterval);
		gameInterval = setInterval(gameLoop, 800);
	}

	// --- Event Handling ---

	/**
	 * Handles keyboard inputs for game control.
	 * @param {KeyboardEvent} e
	 */
	function handleKeydown(e) {
		if (!gameStarted || gameOver) return;

		switch (e.key) {
			case 'ArrowLeft':
				move(-1, 0);
				break;
			case 'ArrowRight':
				move(1, 0);
				break;
			case 'ArrowDown':
				move(0, 1);
				break;
			case 'ArrowUp':
			case ' ': // Space bar for rotation
				e.preventDefault(); // Prevent page scroll on space
				rotatePiece();
				break;
		}
	}

	// --- Lifecycle Hooks ---
	onMount(() => {
		// This ensures the event listener is only added in the browser
		if (browser) {
			window.addEventListener('keydown', handleKeydown);
		}
	});

	onDestroy(() => {
		// This ensures the event listener is only removed in the browser
		if (browser) {
			window.removeEventListener('keydown', handleKeydown);
		}
		// Clear the interval regardless of environment to prevent memory leaks
		clearInterval(gameInterval);
	});

	// --- Reactive UI Calculation ---
	$: displayedBoard = board.map((row, y) => {
		return row.map((cell, x) => {
			let isPiece = false;
			let pieceColor = '';

			if (currentPiece && currentPosition) {
				const pieceX = x - currentPosition.x;
				const pieceY = y - currentPosition.y;

				if (
					pieceX >= 0 && pieceX < currentPiece.shape[0].length &&
					pieceY >= 0 && pieceY < currentPiece.shape.length &&
					currentPiece.shape[pieceY][pieceX]
				) {
					isPiece = true;
					pieceColor = currentPiece.color;
				}
			}
			return {
				...cell,
				isPiece,
				pieceColor
			};
		});
	});

</script>

<!-- Load Tailwind CSS -->
<svelte:head>
	<script src="https://cdn.tailwindcss.com"></script>
</svelte:head>

<div class="bg-gray-900 text-white min-h-screen flex flex-col items-center justify-center font-sans p-4">
	<h1 class="text-5xl font-bold mb-4 tracking-wider">Svelte Tetris</h1>

	<div class="flex flex-col md:flex-row items-start gap-8">
		<!-- Game Board -->
		<div
			class="relative grid border-2 border-gray-600 rounded-lg shadow-2xl shadow-cyan-500/20"
			style="grid-template-columns: repeat({COLS}, {BLOCK_SIZE}px); background-color: #111827;"
		>
			{#each displayedBoard as row, y}
				{#each row as cell, x}
					<div
						class="w-full h-full border border-gray-700/50"
						style="width: {BLOCK_SIZE}px; height: {BLOCK_SIZE}px;"
						class:rounded-sm={cell.value !== 0 || cell.isPiece}
						class:shadow-inner={cell.value !== 0 || cell.isPiece}
						class:bg-gray-800={!cell.isPiece && cell.value === 0}
						class:animate-pulse={cell.isPiece}
						class:{cell.color}={cell.value !== 0}
						class:{cell.pieceColor}={cell.isPiece}
					></div>
				{/each}
			{/each}

			<!-- Game Over / Start Screen Overlay -->
			{#if !gameStarted || gameOver}
				<div class="absolute inset-0 bg-black bg-opacity-70 flex flex-col items-center justify-center rounded-md">
					{#if gameOver}
						<h2 class="text-4xl font-extrabold text-red-500 mb-2">Game Over</h2>
						<p class="text-lg text-gray-300 mb-4">Final Score: {score}</p>
					{/if}
					<button
						on:click={startGame}
						class="px-8 py-3 bg-cyan-500 text-gray-900 font-bold rounded-lg shadow-lg hover:bg-cyan-400 transform hover:scale-105 transition-all duration-200 focus:outline-none focus:ring-4 focus:ring-cyan-300"
					>
						{gameOver ? 'Play Again' : 'Start Game'}
					</button>
				</div>
			{/if}
		</div>

		<!-- Game Info Panel -->
		<div class="w-full md:w-48 flex flex-col gap-4">
			<div class="bg-gray-800 p-4 rounded-lg shadow-lg">
				<h2 class="text-xl font-bold text-cyan-400 mb-2">Score</h2>
				<p class="text-3xl font-mono">{score}</p>
			</div>
			<div class="bg-gray-800 p-4 rounded-lg shadow-lg">
				<h2 class="text-xl font-bold text-cyan-400 mb-2">Controls</h2>
				<ul class="text-gray-300 space-y-1">
					<li><span class="font-bold w-16 inline-block">Left:</span> ←</li>
					<li><span class="font-bold w-16 inline-block">Right:</span> →</li>
					<li><span class="font-bold w-16 inline-block">Down:</span> ↓</li>
					<li><span class="font-bold w-16 inline-block">Rotate:</span> ↑ or Space</li>
				</ul>
			</div>
		</div>
	</div>
</div>
