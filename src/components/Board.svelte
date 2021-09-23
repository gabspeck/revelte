<canvas bind:this={canvas} width={width} height={height}></canvas>
<ul>
	<li>Human: {score[Player.Human]}</li>
	<li>Computer: {score[Player.Computer]}</li>
</ul>

<script lang='ts'>

	import { onMount } from 'svelte';

	enum Player {
		Computer = 'red',
		Human = 'blue'
	}

	const squareSide = 80;
	const columns = 8;
	const rows = 8;
	const width = columns * squareSide;
	const height = rows * squareSide;
	let board: Player[] = [];

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let score: Record<Player, number> = {[Player.Human]: 0, [Player.Computer]: 0}

	$: board.forEach((player: Player, xy: number) => {
		const y = xy % 10;
		const x = (xy - y) / 10;
		score[player] += 1;
		renderPiece(x, y, player);
	})

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

	function renderPiece (c: number, r: number, color: string) {
		let x = squareSide / 2 + ((c - 1) * squareSide);
		let y = squareSide / 2 + ((r - 1) * squareSide);

		ctx.beginPath();
		ctx.arc(x, y, Math.floor(squareSide / 2), 0, 2 * Math.PI);
		ctx.fillStyle = color;
		ctx.fill();
	}

	const reset = () => {
		board = [];
		board[44] = Player.Human;
		board[55] = Player.Human;
		board[45] = Player.Computer;
		board[54] = Player.Computer;
	}

	onMount(() => {
		ctx = canvas.getContext('2d');
		renderGrid();
		reset();
	});

</script>
