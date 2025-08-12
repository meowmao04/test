<script>
	import { onMount, onDestroy, tick } from 'svelte';
	import { writable } from 'svelte/store';
	import { browser } from '$app/environment';
	import { slide } from 'svelte/transition';

	// --- Page State Management ---
	let activeSection = writable('Projects');
	const sections = ['Projects', 'Data Viz', 'Fun', 'Resume'];

	// --- Icon SVGs (as components) ---
	const icons = {
		project: `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-lightbulb"><path d="M15 14c.2-1 .7-1.7 1.5-2.5 1-.9 1.5-2.2 1.5-3.5A6 6 0 0 0 6 8c0 1 .2 2.2 1.5 3.5.7.7 1.3 1.5 1.5 2.5"/><path d="M9 18h6"/><path d="M10 22h4"/></svg>`,
		dataViz: `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-bar-chart-3"><path d="M3 3v18h18"/><path d="M18 17V9"/><path d="M13 17V5"/><path d="M8 17v-3"/></svg>`,
		fun: `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-gamepad-2"><line x1="6" x2="10" y1="12" y2="12"/><line x1="8" x2="8" y1="10" y2="14"/><path d="M13 4v4"/><path d="M15 6h-4"/><path d="M4 15v-3a6 6 0 0 1 6-6h4a6 6 0 0 1 6 6v3a6 6 0 0 1-6 6h-4a6 6 0 0 1-6-6Z"/></svg>`,
		resume: `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-file-text"><path d="M14.5 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7.5L14.5 2z"/><polyline points="14 2 14 8 20 8"/><line x1="16" x2="8" y1="13" y2="13"/><line x1="16" x2="8" y1="17" y2="17"/><line x1="10" x2="8" y1="9" y2="9"/></svg>`,
		download: `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-download"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" x2="12" y1="15" y2="3"/></svg>`
	};

	// --- Tetris Game Logic (for the 'Fun' section) ---
	const COLS = 10;
	const ROWS = 20;
	const BLOCK_SIZE = 30;

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

	let board = createEmptyBoard();
	let currentPiece;
	let currentPosition;
	let score = 0;
	let gameOver = false;
	let gameStarted = false;
	let gameInterval;

	function createEmptyBoard() {
		return Array.from({ length: ROWS }, () => Array(COLS).fill({ value: 0, color: 'bg-gray-800' }));
	}

	function selectNewPiece() {
		const key = TETROMINO_KEYS[Math.floor(Math.random() * TETROMINO_KEYS.length)];
		const pieceData = TETROMINOES[key];
		currentPiece = {
			shape: pieceData.shape.map(row => [...row]),
			color: pieceData.color
		};
		currentPosition = {
			x: Math.floor(COLS / 2) - Math.floor(currentPiece.shape[0].length / 2),
			y: 0
		};
		if (!isValidMove(currentPiece.shape, currentPosition)) {
			gameOver = true;
			gameStarted = false;
			clearInterval(gameInterval);
		}
	}

	function isValidMove(shape, pos) {
		for (let y = 0; y < shape.length; y++) {
			for (let x = 0; x < shape[y].length; x++) {
				if (shape[y][x] !== 0) {
					const newX = pos.x + x;
					const newY = pos.y + y;
					if (newX < 0 || newX >= COLS || newY >= ROWS || (newY >= 0 && board[newY][newX].value !== 0)) {
						return false;
					}
				}
			}
		}
		return true;
	}

	function placePiece() {
		currentPiece.shape.forEach((row, y) => {
			row.forEach((value, x) => {
				if (value !== 0) {
					const boardX = currentPosition.x + x; // FIX: boardX was not defined
					const boardY = currentPosition.y + y;
					if (boardY >= 0) {
						board[boardY][boardX] = { value: 1, color: currentPiece.color };
					}
				}
			});
		});
	}

	function clearLines() {
		let linesCleared = 0;
		for (let y = ROWS - 1; y >= 0; y--) {
			if (board[y].every(cell => cell.value !== 0)) {
				linesCleared++;
				board.splice(y, 1);
				board.unshift(Array(COLS).fill({ value: 0, color: 'bg-gray-800' }));
				y++;
			}
		}
		if (linesCleared > 0) {
			score += linesCleared * 100 * linesCleared;
		}
	}

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

	function move(dx, dy) {
		if (gameOver || !currentPosition) return;
		const newPosition = { x: currentPosition.x + dx, y: currentPosition.y + dy };
		if (isValidMove(currentPiece.shape, newPosition)) {
			currentPosition = newPosition;
			return true;
		}
		return false;
	}

	function gameLoop() {
		if (!move(0, 1)) {
			placePiece();
			clearLines();
			selectNewPiece();
		}
	}

	function startGame() {
		board = createEmptyBoard();
		score = 0;
		gameOver = false;
		gameStarted = true;
		selectNewPiece();
		clearInterval(gameInterval);
		gameInterval = setInterval(gameLoop, 800);
	}

	function handleKeydown(e) {
		if ($activeSection !== 'Fun' || !gameStarted || gameOver) return;
		switch (e.key) {
			case 'ArrowLeft': move(-1, 0); break;
			case 'ArrowRight': move(1, 0); break;
			case 'ArrowDown': move(0, 1); break;
			case 'ArrowUp': case ' ': e.preventDefault(); rotatePiece(); break;
		}
	}

	onMount(() => {
		if (browser) {
			window.addEventListener('keydown', handleKeydown);
		}
	});

	onDestroy(() => {
		if (browser) {
			window.removeEventListener('keydown', handleKeydown);
		}
		clearInterval(gameInterval);
	});

	$: displayedBoard = board.map((row, y) => row.map((cell, x) => {
		let isPiece = false;
		let pieceColor = '';
		if (currentPiece && currentPosition) {
			const pieceX = x - currentPosition.x;
			const pieceY = y - currentPosition.y;
			if (pieceX >= 0 && pieceX < currentPiece.shape[0].length && pieceY >= 0 && pieceY < currentPiece.shape.length && currentPiece.shape[pieceY][pieceX]) {
				isPiece = true;
				pieceColor = currentPiece.color;
			}
		}
		return { ...cell, isPiece, pieceColor };
	}));

</script>

<svelte:head>
	<title>Alex Doe - Portfolio</title>
	<script src="https://cdn.tailwindcss.com"></script>
	<link rel="preconnect" href="https://fonts.googleapis.com">
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
	<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700;900&display=swap" rel="stylesheet">
</svelte:head>

<body class="bg-slate-900 text-slate-300 font-['Inter'] antialiased">
	<div class="min-h-screen flex flex-col">
		<!-- Header & Navigation -->
		<header class="sticky top-0 z-50 bg-slate-900/70 backdrop-blur-lg border-b border-slate-700">
			<nav class="container mx-auto px-6 py-4 flex justify-between items-center">
				<a href="/" class="text-2xl font-bold text-white">
					Alex <span class="text-cyan-400">Doe</span>
				</a>
				<ul class="flex items-center space-x-2 md:space-x-4">
					{#each sections as section}
						<li>
							<button
								on:click={() => activeSection.set(section)}
								class="px-3 py-2 rounded-md text-sm font-medium transition-colors duration-200"
								class:text-white={$activeSection === section}
								class:bg-slate-800={$activeSection === section}
								class:text-slate-400={$activeSection !== section}
								class:hover:text-white={$activeSection !== section}
								class:hover:bg-slate-800/50={$activeSection !== section}
							>
								{section}
							</button>
						</li>
					{/each}
				</ul>
			</nav>
		</header>

		<!-- Main Content Area -->
		<main class="container mx-auto px-6 py-12 flex-grow">
			{#if $activeSection === 'Projects'}
				<div transition:slide={{ duration: 300 }}>
					<h1 class="text-4xl font-extrabold text-white mb-2">My Work & <span class="text-cyan-400">Projects</span></h1>
					<p class="text-slate-400 mb-8 max-w-2xl">A selection of projects that showcase my passion for building useful and beautiful things.</p>
					<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8">
						<!-- Project Card Example -->
						{#each Array(3) as _, i}
						<div class="bg-slate-800 rounded-lg overflow-hidden shadow-lg hover:shadow-cyan-500/20 transition-shadow duration-300 transform hover:-translate-y-1">
							<img src="https://placehold.co/600x400/1e293b/94a3b8?text=Project+Screenshot" alt="Project {i+1}" class="w-full h-48 object-cover">
							<div class="p-6">
								<h3 class="text-xl font-bold text-white mb-2">Project Title {i+1}</h3>
								<p class="text-slate-400 mb-4">A brief description of the project, its purpose, and the technologies used.</p>
								<div class="flex space-x-4">
									<a href="#" class="text-cyan-400 hover:text-cyan-300 font-medium">Live Demo</a>
									<a href="#" class="text-cyan-400 hover:text-cyan-300 font-medium">GitHub</a>
								</div>
							</div>
						</div>
						{/each}
					</div>
				</div>
			{/if}

			{#if $activeSection === 'Data Viz'}
				<div transition:slide={{ duration: 300 }}>
					<h1 class="text-4xl font-extrabold text-white mb-2">Data <span class="text-cyan-400">Visualizations</span></h1>
					<p class="text-slate-400 mb-8 max-w-2xl">Exploring complex datasets and turning them into compelling visual stories.</p>
					<div class="bg-slate-800 p-8 rounded-lg shadow-lg">
						<h3 class="text-2xl font-bold text-white mb-4">Interactive Chart Placeholder</h3>
						<p class="text-slate-400 mb-6">This is where an interactive D3.js, Chart.js, or other data visualization would be rendered.</p>
						<div class="h-80 bg-slate-700 rounded-md flex items-center justify-center">
							<span class="text-slate-500">Chart Area</span>
						</div>
					</div>
				</div>
			{/if}

			{#if $activeSection === 'Fun'}
				<div transition:slide={{ duration: 300 }}>
					<h1 class="text-4xl font-extrabold text-white mb-2">Just for <span class="text-cyan-400">Fun</span></h1>
					<p class="text-slate-400 mb-8 max-w-2xl">Sometimes you just have to build a game. Use arrow keys to move and rotate.</p>
					<!-- Tetris Game Component -->
					<div class="flex flex-col md:flex-row items-start justify-center gap-8">
						<div
							class="relative grid border-2 border-slate-700 rounded-lg shadow-2xl shadow-cyan-500/20"
							style="grid-template-columns: repeat({COLS}, {BLOCK_SIZE}px); background-color: #0f172a;"
						>
							{#each displayedBoard as row}
								{#each row as cell}
									<div
										class="w-full h-full border border-slate-700/50"
										style="width: {BLOCK_SIZE}px; height: {BLOCK_SIZE}px;"
										class:rounded-sm={cell.value !== 0 || cell.isPiece}
										class:shadow-inner={cell.value !== 0 || cell.isPiece}
										class:bg-slate-800={!cell.isPiece && cell.value === 0}
										class:animate-pulse={cell.isPiece}
										class:{cell.color}={cell.value !== 0}
										class:{cell.pieceColor}={cell.isPiece}
									></div>
								{/each}
							{/each}

							{#if !gameStarted || gameOver}
								<div class="absolute inset-0 bg-black bg-opacity-75 flex flex-col items-center justify-center rounded-md">
									{#if gameOver}
										<h2 class="text-4xl font-extrabold text-red-500 mb-2">Game Over</h2>
										<p class="text-lg text-slate-300 mb-4">Final Score: {score}</p>
									{/if}
									<button on:click={startGame} class="px-8 py-3 bg-cyan-500 text-slate-900 font-bold rounded-lg shadow-lg hover:bg-cyan-400 transform hover:scale-105 transition-all duration-200 focus:outline-none focus:ring-4 focus:ring-cyan-300">
										{gameOver ? 'Play Again' : 'Start Game'}
									</button>
								</div>
							{/if}
						</div>
						<div class="w-full md:w-48 flex flex-col gap-4">
							<div class="bg-slate-800 p-4 rounded-lg shadow-lg">
								<h2 class="text-xl font-bold text-cyan-400 mb-2">Score</h2>
								<p class="text-3xl font-mono">{score}</p>
							</div>
						</div>
					</div>
				</div>
			{/if}

			{#if $activeSection === 'Resume'}
				<div transition:slide={{ duration: 300 }}>
					<h1 class="text-4xl font-extrabold text-white mb-8">My <span class="text-cyan-400">Resume</span></h1>
					<div class="bg-slate-800 p-8 rounded-lg shadow-lg max-w-4xl mx-auto">
						<div class="flex justify-between items-start mb-8">
							<div>
								<h2 class="text-3xl font-bold text-white">Alex Doe</h2>
								<p class="text-cyan-400">Senior Frontend Developer</p>
							</div>
							<a href="#" class="inline-flex items-center gap-2 px-4 py-2 bg-cyan-500 text-slate-900 font-bold rounded-lg shadow-lg hover:bg-cyan-400 transform hover:scale-105 transition-all duration-200">
								{@html icons.download}
								Download CV
							</a>
						</div>

						<div class="space-y-8">
							<!-- Experience Section -->
							<div>
								<h3 class="text-2xl font-bold text-white border-b-2 border-slate-700 pb-2 mb-4">Experience</h3>
								<div class="space-y-4">
									<div>
										<h4 class="text-xl font-semibold text-cyan-400">Lead Developer at TechCorp</h4>
										<p class="text-slate-500 text-sm mb-1">2020 - Present</p>
										<p class="text-slate-400">Leading a team to build next-generation web applications using Svelte, React, and Node.js.</p>
									</div>
									<div>
										<h4 class="text-xl font-semibold text-cyan-400">Frontend Engineer at WebSolutions</h4>
										<p class="text-slate-500 text-sm mb-1">2017 - 2020</p>
										<p class="text-slate-400">Developed and maintained client websites, focusing on performance and user experience.</p>
									</div>
								</div>
							</div>
							<!-- Skills Section -->
							<div>
								<h3 class="text-2xl font-bold text-white border-b-2 border-slate-700 pb-2 mb-4">Skills</h3>
								<div class="flex flex-wrap gap-2">
									{#each ['SvelteKit', 'React', 'JavaScript (ES6+)', 'TypeScript', 'Node.js', 'Tailwind CSS', 'Figma', 'Git'] as skill}
										<span class="bg-slate-700 text-slate-300 px-3 py-1 rounded-full text-sm font-medium">{skill}</span>
									{/each}
								</div>
							</div>
						</div>
					</div>
				</div>
			{/if}
		</main>
	</div>
</body>
