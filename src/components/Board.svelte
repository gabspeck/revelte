<canvas bind:this={board} width={width} height={height}></canvas>

<script lang='ts'>

	import { onMount } from 'svelte';

	const squareSide = 50;
	const columns = 8;
	const rows = 8;
	const width = columns * squareSide;
	const height = rows * squareSide;

	let board: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;

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

	const renderPiece = (c: number, r: number, color: string) => {
		let x = squareSide / 2 + ((c - 1) * squareSide);
		let y = squareSide / 2 + ((r - 1) * squareSide);

		ctx.beginPath();
		ctx.arc(x, y, Math.floor(squareSide / 2), 0, 2 * Math.PI);
		ctx.fillStyle = color;
		ctx.fill();
	};

	const reset = () => {
		renderGrid();
		renderPiece(4, 4, 'blue');
		renderPiece(5, 4, 'red');
		renderPiece(4, 5, 'red');
		renderPiece(5, 5, 'blue');
	}

	onMount(() => {
		ctx = board.getContext('2d');
		reset();
	});

</script>
