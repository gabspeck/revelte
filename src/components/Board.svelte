<canvas bind:this={canvas} on:mousemove={updateCurrentSquare} on:click={humanMove} width={width}
				height={height}></canvas>
<ul>
	<li>Current player: {currentPlayer}</li>
	<li>Human: {score[Player.Human]}</li>
	<li>Computer: {score[Player.Computer]}</li>
	<li>Current score: {getScore(board, currentPlayer)}</li>
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
</p>

<script lang='ts'>

	import { onMount } from 'svelte';

	enum Player {
		Computer = 'red',
		Human = 'blue'
	}

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

	let maxDepth = 2;

	const victory = Number.MAX_VALUE;

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
	let board: Player[] = boards[0];
	let currentPlayer = Player.Human;

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let score: Record<Player, number> = { [Player.Human]: 0, [Player.Computer]: 0 };

	let currentSquare = 0;

	$: boards[0] = board;

	$: [Player.Human, Player.Computer].forEach(p => {
		score[p] = board.filter(sq => sq === p).length;
	});

	$: board.forEach((player: Player, square: number) => {
		renderPiece(square, player);
	});

	$: if (canvas) {
		canvas.style.cursor = getMoveVectors({
			player: currentPlayer,
			start: currentSquare,
			board: board
		}).length ? 'crosshair' : 'not-allowed';
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
			const enemy = player == Player.Human ? Player.Computer : Player.Human;
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

	function humanMove(ev: MouseEvent) {
		const square = clientCoordsToSquare(ev);
		const vectors = getMoveVectors({ start: square, player: currentPlayer, board });
		vectors.forEach(v => makeMove(v));
		if (vectors.length) {
			board = board;
			currentPlayer = currentPlayer == Player.Human ? Player.Computer : Player.Human;
		}
		const mmax = minimax(board, maxDepth, Player.Computer)
	}

	function makeMove(move: MoveVector) {
		const { direction, end, start } = move;
		for (let square = start; direction > 0 && square <= end || direction < 0 && square >= end; square += direction) {
			move.board[square] = move.player;
		}
	}

	function getScore(board: Player[], player: Player) {

		const coordValueOffset = -11;


		const freeCornersValues = [
			99, -8, 8, 6, 6, 8, -8, 99, 0,
			0, -8, -24, -4, -3, -3, -4, -24, -8, 0,
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

			if (upperLeft){
				board[11 + Direction.E - coordValueOffset] = - 8;
				board[11 + Direction.S - coordValueOffset] = - 8;
				board[11 + Direction.SE - coordValueOffset] = -24;
			}

			if (upperRight){
				board[18 + Direction.W - coordValueOffset] = - 8;
				board[18 + Direction.S - coordValueOffset] = - 8;
				board[18 + Direction.SW - coordValueOffset] = -24;
			}

			if (lowerLeft){
				board[81 + Direction.E - coordValueOffset] = - 8;
				board[81 + Direction.N - coordValueOffset] = - 8;
				board[81 + Direction.NE - coordValueOffset] = -24;
			}

			if (lowerRight){
				board[88 + Direction.W - coordValueOffset] = - 8;
				board[88 + Direction.N - coordValueOffset] = - 8;
				board[88 + Direction.NW - coordValueOffset] = -24;
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
				if (board[square] !== player) {
					enemyCount++;
				}
				boardScore.set(board[square], boardScore.get(board[square]) + value);
			}
		});

		if (enemyCount === 0) {
			return victory;
		}

		const friendlyScore = boardScore.get(player);
		boardScore.delete(player);

		return friendlyScore - boardScore.values().next().value;

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

	function minimax(board: Player[], ply, player) {
		if (ply === 0 || ply === maxDepth) {
			return getScore(board, player);
		}
		const validMoves = getValidMoves(board, player);
		if (player === Player.Computer) {
			let value = -Infinity;
			for (const m of validMoves) {
				board[m] = player;
				value = Math.max(value, minimax(board, ply - 1, Player.Human));
			}
		} else {
			let value = Infinity;
			for (const m of validMoves) {
				board[m] = player;
				value = Math.min(value, minimax(board, ply - 1, Player.Computer));
			}
		}
	}

	function renderGrid() {
		ctx.beginPath();
		ctx.fillStyle = 'white'
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
		ctx.fillStyle = 'white'
		ctx.fillRect(x + ctx.lineWidth - squareSide / 2, y + ctx.lineWidth - squareSide / 2, squareSide - ctx.lineWidth * 2, squareSide - ctx.lineWidth * 3);
		ctx.arc(x, y, radius, 0, 2 * Math.PI);
		ctx.fillStyle = color;
		ctx.fill();

	}

	function reset() {
		board = [];
		board[45] = Player.Computer;
		board[54] = Player.Computer;
		board[44] = Player.Human;
		board[55] = Player.Human;
		renderGrid();
		currentPlayer = Player.Human;
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
