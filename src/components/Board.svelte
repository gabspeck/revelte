<canvas bind:this={canvas} on:mousemove={updateCurrentSquare} on:click={humanMove} width={width}
				height={height}></canvas>
<ul>
	<li>Human: {score[Player.Human]}</li>
	<li>Computer: {score[Player.Computer]}</li>
	<li>Difficulty (depth):
		<select bind:value={maxDepth}>
			{#each [1, 2, 3, 4, 5, 6] as i}
				<option value='{i}'>{i}</option>
			{/each}
		</select>
	</li>
</ul>
<p>
	<button on:click={reset}>Reset</button>
	<button on:click={pass} disabled={!canPass}>Pass</button>
</p>

<script lang='ts'>

	import { onMount } from 'svelte';

	enum Player {
		Computer = 'red',
		Human = 'blue'
	}

	const opponent = Object.freeze({
		[Player.Computer]: Player.Human,
		[Player.Human]: Player.Computer
	});

	interface Point {
		x: number,
		y: number
	}

	enum Direction {
		NW = -11,
		N = -10,
		NE = -9,
		E = 1,
		SE = 11,
		S = 10,
		SW = 9,
		W = -1,
	}

	const PASS = 0;

	let maxDepth = 2;

	const victory = Infinity;

	const directions: Direction[] = [Direction.N, Direction.NE, Direction.E, Direction.SE, Direction.S, Direction.SW, Direction.W, Direction.NW];

	interface Move {
		player: Player,
		start: number
		board: Player[]
	}

	interface MoveVector extends Move {
		direction: Direction,
		end: number
	}

	const squareSide = 50;
	const columns = 8;
	const rows = 8;
	const width = columns * squareSide;
	const height = rows * squareSide;
	// offset in pixels to avoid subpixel rendering with uneven line widths (i.e. blurry lines)
	const subpixelOffset = 0.5;

	let boards: Player[][] = [[]];
	let gameBoard: Player[] = boards[0];

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let score: Record<Player, number> = { [Player.Human]: 0, [Player.Computer]: 0 };
	let canPass;
	let firstPlay = true;

	let currentSquare = 0;

	$: boards[0] = gameBoard;

	$: [Player.Human, Player.Computer].forEach(p => {
		score[p] = gameBoard.filter(sq => sq === p).length;
	});

	$: gameBoard.forEach((player: Player, square: number) => {
		renderPiece(square, player);
	});

	$: canPass = firstPlay || !getValidMoves(gameBoard, Player.Human).length;

	$: if (canvas) {
		canvas.style.cursor = getMoveVectors({
			player: Player.Human,
			start: currentSquare,
			board: gameBoard
		}).length ? 'crosshair' : 'not-allowed';
	}

	function pass() {
		computerMove(PASS);
		firstPlay = false;
	}

	function pointToSquare({ x, y }: Point): number {
		const col = Math.trunc(y / squareSide + 1);
		const row = Math.trunc(x / squareSide + 1);

		return col * 10 + row;
	}

	function getSquareCenter(square: number): Point {
		const col = square % 10;
		const row = (square - col) / 10;
		const x = squareSide / 2 + ((col - 1) * squareSide);
		const y = squareSide / 2 + ((row - 1) * squareSide);

		return { x, y };
	}

	function getMoveVectors({ start, player, board }: Move): MoveVector[] {
		const vectors: MoveVector[] = [];
		if (!board[start]) {
			const enemy = opponent[player];
			for (const d of directions) {
				let square = start;
				if (board[square += d] === enemy) {
					while (board[square] === enemy) {
						square += d;
					}
					if (board[square] === player) {
						vectors.push({ direction: d, start: start, end: square - d, board, player });
					}
				}
			}
		}
		return vectors;
	}

	function clientCoordsToSquare({ clientX, clientY, target }: Partial<MouseEvent>): number {
		const rect = (target as Element).getBoundingClientRect();
		const point: Point = { x: clientX - rect.left, y: clientY - rect.top };

		return pointToSquare(point);
	}

	function updateCurrentSquare(ev: MouseEvent) {
		const square = clientCoordsToSquare(ev);

		if (currentSquare !== square) {
			currentSquare = square;
		}
	}

	function computerMove(humanMove) {
		const { bestMove } = minimax(boards, Player.Human, humanMove, 0, -Infinity, Infinity);
		if (bestMove) {
			makeMove({ player: Player.Computer, square: bestMove, board: gameBoard });
			gameBoard = gameBoard;
		}
	}

	function humanMove(ev: MouseEvent) {
		const square = clientCoordsToSquare(ev);
		if (makeMove({ square, player: Player.Human, board: gameBoard })) {
			firstPlay = false;
			computerMove(square);
			gameBoard = gameBoard;
		}
	}

	function logBoard(board: Player[]) {
		console.log(' 12345678');
		const symbol = {
			[Player.Computer]: 'C',
			[Player.Human]: 'H',
			undefined: '.'
		};
		for (let i = 0; i < 8; i++) {
			let row = (i + 1).toString();
			for (let j = 0; j < 8; j++) {
				const square = board[i * 10 + j + 11];
				row += symbol[square];
			}
			console.log(row);
		}
	}

	function makeMove({ player, square, board }: { player: Player, square: number, board: Player[] }): boolean {
		const vectors = getMoveVectors({ start: square, player, board });
		vectors.forEach(v => moveTo(v));
		return !!vectors.length;
	}

	function moveTo(vector: MoveVector) {
		const { direction, end, start } = vector;
		for (let square = start; direction > 0 && square <= end || direction < 0 && square >= end; square += direction) {
			vector.board[square] = vector.player;
		}
	}

	function getScore(board: Player[], player: Player) {

		const coordValueOffset = 11;

		const freeCornersValues = [
			99, -8, 8, 6, 6, 8, -8, 99, 0,
			-8, -24, -4, -3, -3, -4, -24, -8, 0,
			0, 8, -4, 7, 4, 4, 7, -4, 8, 0,
			0, 6, -3, 4, 0, 0, 4, -3, 6, 0,
			0, 6, -3, 4, 0, 0, 4, -3, 6, 0,
			0, 8, -4, 7, 4, 4, 7, -4, 8, 0,
			0, -8, -24, -4, -3, -3, -4, -24, -8, 0,
			0, 99, -8, 8, 6, 6, 8, -8, 99
		];

		const playedCornerValues = [
			99, -8, 8, 6, 6, 8, -8, 99, 0,
			0, -8, -24, 0, 1, 1, 0, -24, -8, 0,
			0, 8, 0, 7, 4, 4, 7, 0, 8, 0,
			0, 6, 1, 4, 1, 1, 4, 1, 6, 0,
			0, 6, 1, 4, 1, 1, 4, 1, 6, 0,
			0, 8, 0, 7, 4, 4, 7, 0, 8, 0,
			0, -8, -24, 0, 1, 1, 1, -24, -8, 0,
			0, 99, -8, 8, 6, 6, 8, -8, 99
		];

		let values = freeCornersValues;

		const upperLeft = board[11];
		const upperRight = board[18];
		const lowerLeft = board[81];
		const lowerRight = board[88];

		if (upperLeft || upperRight || lowerLeft || lowerRight) {
			values = playedCornerValues;

			if (upperLeft) {
				values[11 + Direction.E - coordValueOffset] = 12;
				values[11 + Direction.S - coordValueOffset] = 12;
				values[11 + Direction.SE - coordValueOffset] = 8;
			}

			if (upperRight) {
				values[18 + Direction.W - coordValueOffset] = 12;
				values[18 + Direction.S - coordValueOffset] = 12;
				values[18 + Direction.SW - coordValueOffset] = 8;
			}

			if (lowerLeft) {
				values[81 + Direction.E - coordValueOffset] = 12;
				values[81 + Direction.N - coordValueOffset] = 12;
				values[81 + Direction.NE - coordValueOffset] = 8;
			}

			if (lowerRight) {
				values[88 + Direction.W - coordValueOffset] = 12;
				values[88 + Direction.N - coordValueOffset] = 12;
				values[88 + Direction.NW - coordValueOffset] = 8;
			}

		}

		const boardScore = new Map<Player, number>([
			[Player.Human, 0],
			[Player.Computer, 0]
		]);

		let enemyCount = 0;

		values.forEach((value, index) => {
			const square = index + coordValueOffset;
			if (board[square]) {
				if (board[square] === opponent[player]) {
					enemyCount++;
				}
				boardScore.set(board[square], boardScore.get(board[square]) + value);
			}
		});

		if (enemyCount === 0) {
			return victory;
		}

		return boardScore.get(player) - boardScore.get(opponent[player]);

	}

	function getFinalScore(board: Player[], player: Player) {
		let count = 0;
		const win = 32000;
		const loss = -win;
		for (const sq of board) {
			if (sq === player) {
				count++;
			} else if (sq === opponent[player]) {
				count--;
			}
		}
		if (count > 0) {
			return win + count;
		} else if (count < 0) {
			return loss + count;
		}
		return 0;
	}

	function getValidMoves(board: Player[], player: Player) {
		const moves = [
			11, 18, 81, 88, 13, 31, 16, 61,
			38, 83, 68, 86, 14, 41, 15, 51,
			48, 84, 58, 85, 33, 36, 63, 66,
			34, 35, 43, 46, 53, 56, 64, 65,
			24, 25, 42, 47, 52, 57, 74, 75,
			23, 26, 32, 37, 62, 67, 73, 76,
			12, 17, 21, 28, 71, 78, 82, 87,
			22, 27, 72, 77
		];

		const validMoves = [];

		moves.forEach(m => {
			if (getMoveVectors({ start: m, player, board }).length) {
				validMoves.push(m);
			}
		});

		return validMoves;

	}

	function minimax(boards: Player[][], player: Player, playedMove: number, ply: number, vmin: number, vmax: number) {
		const currentPly = ply + 1;
		if (!boards[currentPly]) {
			boards[currentPly] = [];
		}
		for (let i = 11; i <= 88; i++) {
			boards[currentPly][i] = boards[ply][i];
		}
		if (playedMove === PASS) {
			if (ply === maxDepth) {
				const moves = getValidMoves(boards[currentPly], player);
				if (moves.length) {
					return getScore(boards[currentPly], player);
				}
				return getFinalScore(boards[currentPly], player);
			}
		} else if (ply !== 0) {
			makeMove({ player, square: playedMove, board: boards[currentPly] });
			if (ply === maxDepth) {
				return getScore(boards[currentPly], player);
			}
		}
		const validMoves = getValidMoves(boards[currentPly], opponent[player]);
		let currentMove = PASS;
		let bestMove = PASS;
		for (const validMove of validMoves) {
			currentMove = validMove;
			const { value } = minimax(boards, opponent[player], validMove, currentPly, -vmax, -vmin);
			if (value > vmin) {
				vmin = value;
				if (value >= vmax) {
					return { bestMove: validMove, value: -vmin };
				}
			}
		}
		if (currentMove === PASS) {
			if (playedMove === PASS) {
				return { bestMove: currentMove, value: getFinalScore(boards[currentPly], player) };
			}
			const { value } = minimax(boards, opponent[player], PASS, currentPly, -vmax, -vmin);
			if (value > vmin) {
				vmin = value;
			}
		}
		return { bestMove, value: -vmin };
	}

	function renderGrid() {
		ctx.beginPath();
		ctx.fillStyle = 'white';
		ctx.fillRect(0, 0, width, height);
		let x = 0, y = squareSide;
		for (let i = 0; i < rows - 1; i++) {
			ctx.moveTo(x, y);
			ctx.lineTo(x + squareSide * columns, y);
			y += squareSide;
		}
		x = squareSide;
		y = 0;
		for (let i = 0; i < columns - 1; i++) {
			ctx.moveTo(x, y);
			ctx.lineTo(x, y + squareSide * rows);
			x += squareSide;
		}
		ctx.rect(0, 0, width - subpixelOffset, height - subpixelOffset);
		ctx.stroke();
	}

	function renderPiece(square: number, color: string) {
		const { x, y } = getSquareCenter(square);
		const margin = 10 * ctx.lineWidth;

		const radius = Math.floor((squareSide - margin) / 2);

		ctx.beginPath();
		ctx.fillStyle = 'white';
		ctx.fillRect(x + ctx.lineWidth - squareSide / 2, y + ctx.lineWidth - squareSide / 2, squareSide - ctx.lineWidth * 2, squareSide - ctx.lineWidth * 3);
		ctx.arc(x, y, radius, 0, 2 * Math.PI);
		ctx.fillStyle = color;
		ctx.fill();

	}

	function reset() {
		gameBoard = [];
		gameBoard[45] = Player.Computer;
		gameBoard[54] = Player.Computer;
		gameBoard[44] = Player.Human;
		gameBoard[55] = Player.Human;
		firstPlay = true;
		renderGrid();
	}

	onMount(() => {
		let getContextResult: CanvasRenderingContext2D | null;
		getContextResult = canvas.getContext('2d', { alpha: false });
		if (!getContextResult) {
			throw new Error('Could not get 2D rendering context');
		}
		ctx = getContextResult;
		ctx.translate(subpixelOffset, subpixelOffset);
		reset();
	});

</script>
