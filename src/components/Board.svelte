<canvas bind:this={canvas} on:mousemove={updateCurrentSquare} on:click={humanMove} width={width}
				height={height}></canvas>
<ul>
	<li>Current player: {currentPlayer}</li>
	<li>Human: {score[Player.Human]}</li>
	<li>Computer: {score[Player.Computer]}</li>
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
		N = -1,
		NE = 9,
		E = 10,
		SE = 11,
		S = 1,
		SW = -9,
		W = -10,
		NW = -11,
	}

	let currentSquare = 0;

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

	let board: Player[] = [];
	let currentPlayer = Player.Human;

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let score: Record<Player, number> = { [Player.Human]: 0, [Player.Computer]: 0 };

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
				board
			}).length ? 'crosshair' : 'not-allowed';
		}

	const pointToSquare = ({ x, y }: Point): number => {
		const col = Math.trunc(x / squareSide + 1);
		const row = Math.trunc(y / squareSide + 1);

		return col * 10 + row;
	};

	const getSquareCenter = (square: number): Point => {
		const row = square % 10;
		const col = (square - row) / 10;
		const x = squareSide / 2 + ((col - 1) * squareSide);
		const y = squareSide / 2 + ((row - 1) * squareSide);

		return { x, y };
	};

	function getMoveVectors ({ start, player, board }: Move): MoveVector[] {
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

	const clientCoordsToSquare = ({ clientX, clientY, target }: Partial<MouseEvent>): number => {
		const rect = (target as Element).getBoundingClientRect();
		const point: Point = { x: clientX - rect.left, y: clientY - rect.top };
		return pointToSquare(point);
	};

	const updateCurrentSquare = (ev: MouseEvent) => {
		const square = clientCoordsToSquare(ev);

		if (currentSquare !== square) {
			currentSquare = square;
		}
	};

	const humanMove = (ev: MouseEvent) => {
		const square = clientCoordsToSquare(ev);
		const vectors = getMoveVectors({ start: square, player: currentPlayer, board });
		vectors.forEach(v => makeMove(v));
		if (vectors.length) {
			board = board;
			currentPlayer = currentPlayer == Player.Human ? Player.Computer : Player.Human;
		}
	};

	const makeMove = (move: MoveVector) => {
		const { direction, end, start } = move;
		for (let square = start; direction > 0 && square <= end || direction < 0 && square >= end; square += direction) {
			move.board[square] = move.player;
		}
	};

	const renderGrid = () => {
		ctx.beginPath();
		ctx.clearRect(0, 0, width, height);
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
	};


	function renderPiece(square: number, color: string) {
		const { x, y } = getSquareCenter(square);
		const margin = 10 * ctx.lineWidth;

		const radius = Math.floor((squareSide - margin) / 2);

		ctx.beginPath();
		ctx.clearRect(x + ctx.lineWidth - squareSide / 2, y + ctx.lineWidth - squareSide / 2, squareSide - ctx.lineWidth * 2, squareSide - ctx.lineWidth * 3);
		ctx.arc(x, y, radius, 0, 2 * Math.PI);
		ctx.fillStyle = color;
		ctx.fill();

	}

	const reset = () => {
		board = [];
		board[44] = Player.Human;
		board[55] = Player.Human;
		board[45] = Player.Computer;
		board[54] = Player.Computer;
		renderGrid();
		currentPlayer = Player.Human;
	};

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
