<canvas bind:this={canvas} on:mousemove={highlightValidMove} width={width} height={height}></canvas>
<ul>
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

	interface Move {
		player: Player,
		square: number
		board: Player[]
	}

	const squareSide = 80;
	const columns = 8;
	const rows = 8;
	const width = columns * squareSide;
	const height = rows * squareSide;
	let board: Player[] = [];

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let score: Record<Player, number> = { [Player.Human]: 0, [Player.Computer]: 0 };

	$: board.forEach((player: Player, square: number) => {
		score[player] += 1;
		renderPiece(square, player);
	});

	const pointToSquare = ({ x, y }: Point): number => {
		const col = Math.trunc(x / squareSide + 1);
		const row = Math.trunc(y / squareSide + 1);

		return col * 10 + row;
	};

	const squareToPoint = (square: number): Point => {
		const row = square % 10;
		const col = (square - row) / 10;
		const x = squareSide / 2 + ((col - 1) * squareSide);
		const y = squareSide / 2 + ((row - 1) * squareSide);

		return { x, y };
	};

	const isMoveValid = (move: Move): boolean => {
		// stub
		return move.square % 2 == 0;
	};

	const highlightValidMove = (ev: MouseEvent) => {
		const { clientX, clientY } = ev;
		const rect = (ev.target as Element).getBoundingClientRect();

		const target = ev.target as HTMLElement;
		const point: Point = { x: clientX - rect.left, y: clientY - rect.top };
		target.style.cursor = isMoveValid({
			player: Player.Human,
			square: pointToSquare(point),
			board
		}) ? 'crosshair' : 'not-allowed';
	};

	const renderGrid = () => {
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
		ctx.rect(0, 0, width, height);
		ctx.stroke();
	};

	function renderPiece(square: number, color: string) {
		const { x, y } = squareToPoint(square);

		ctx.beginPath();
		ctx.arc(x, y, Math.floor(squareSide / 2), 0, 2 * Math.PI);
		ctx.fillStyle = color;
		ctx.fill();
	}

	const reset = () => {
		board = [];
		for (let scoreKey in score) {
			score[scoreKey] = 0;
		}
		board[44] = Player.Human;
		board[55] = Player.Human;
		board[45] = Player.Computer;
		board[54] = Player.Computer;
	};

	onMount(() => {
		let getContextResult: CanvasRenderingContext2D | null;
		getContextResult = canvas.getContext('2d');
		if (!getContextResult) {
			throw new Error('Could not get 2D rendering context');
		}
		ctx = getContextResult;
		renderGrid();
		reset();
	});

</script>
