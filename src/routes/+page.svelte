<script lang="ts">
	import { Canvas, Layer, type Render } from 'svelte-canvas';

	let x = 0;
	let y = 0;

	let mouseDown = false;
	let dragging = false;

	let boxSize = 50;

	let width: number;
	let height: number;

	type Vector = [x: number, y: number];
	type Map = Record<string, boolean>;

	let map: Map = {};
	let history: Map[] = []

	function getNeighbors(x: number, y: number): Vector[] {
		return [
			[x - 1, y - 1],
			[x - 1, y],
			[x - 1, y + 1],
			[x, y - 1],
			[x, y + 1],
			[x + 1, y - 1],
			[x + 1, y],
			[x + 1, y + 1]
		];
	}

	const xOffset = 12;

	function inBounds(x: number, y: number): boolean {
		if (y > 11) return false;

		if (y > 10) {
			// only allow x -1, 0, and 1
			if (x > xOffset + 1 || x < xOffset + -1) {
				return false;
			}
		}

		return true;
	}

	function clickable(x: number, y: number): boolean {
		return y === 11;
	}

	// make 1 iter of the game of life
	function step() {
		// Create a new object to store the updated state
		const updatedMap: Map = {};

		// iter through game of life
		for (const [key, value] of Object.entries(map)) {
			if (!value) continue;
			const [x, y] = JSON.parse(key);
			const neighbors = getNeighbors(x, y);

			// check for any neighbors that aren't alive and decide whether they should be
			for (const [nx, ny] of neighbors) {
				if (!inBounds(nx, ny)) continue;

				if (!map[JSON.stringify([nx, ny])]) {
					const aliveNeighbors = getNeighbors(nx, ny).filter(
						([x, y]) => map[JSON.stringify([x, y])]
					).length;

					if (aliveNeighbors === 3) {
						updatedMap[JSON.stringify([nx, ny])] = true;
					}
				}
			}

			// decide whether this cell lives or dies
			const neighborCount = neighbors.filter(([nx, ny]) => map[JSON.stringify([nx, ny])]).length;

			if (neighborCount < 2 || neighborCount > 3) {
				updatedMap[key] = false;
			} else {
				updatedMap[key] = true;
			}
		}

		// Store the current state in the history
		history.push(structuredClone(map));

		// Apply the changes to the map
		for (const [key, value] of Object.entries(updatedMap)) {
			map[key] = value;
		}
	}

	function screenToMap(xCursor: number, yCursor: number): Vector {
		return [Math.floor((xCursor - x) / boxSize), Math.floor((yCursor - y) / boxSize)];
	}

	let render: Render;
	$: render = ({ context, width, height }) => {
		for (let i = -5; i < width / boxSize + 5; i++) {
			for (let j = -5; j < height / boxSize + 5; j++) {
				let xCoord = i - (x > 0 ? Math.floor(x / boxSize) : Math.ceil(x / boxSize));
				let yCoord = j - (y > 0 ? Math.floor(y / boxSize) : Math.ceil(y / boxSize));
				const xPos = i * boxSize + (x % boxSize);
				const yPos = j * boxSize + (y % boxSize);

				if (!inBounds(xCoord, yCoord)) continue;

				const activeCell = map[JSON.stringify([xCoord, yCoord])];

				if (activeCell) {
					if (clickable(xCoord, yCoord)) {
						context.fillStyle = 'black';
					} else {
						context.fillStyle = '#2A2B2A';
					}
					context.fillRect(xPos, yPos, boxSize, boxSize);
				} else {
					context.fillStyle = 'white';
					if (clickable(xCoord, yCoord)) {
						context.strokeStyle = 'black';
					} else {
						context.strokeStyle = 'gray';
					}
					context.strokeRect(xPos, yPos, boxSize, boxSize);
				}
			}
		}
	};
</script>

<svelte:window
	bind:innerWidth={width}
	bind:innerHeight={height}
	on:keypress={({ key }) => {
		if (key === ' ') {
			step();
		}

		if (key === 'z') {
			map = history.pop() || {};
		}

		if (key.match(/\d/)) {
			const xCoord = parseInt(key) - 2 + xOffset;
			const yCoord = 11;
			if (inBounds(xCoord, yCoord)) {
				map[JSON.stringify([xCoord, yCoord])] = !map[JSON.stringify([xCoord, yCoord])];
			}
		}
	}}
/>

<Canvas
	on:wheel={({ deltaY }) => {
		boxSize += deltaY / 100;
		boxSize = Math.min(Math.max(30, boxSize), 100);
	}}
	on:mousedown={() => {
		mouseDown = true;
	}}
	on:mousemove={({ movementX, movementY }) => {
		if (mouseDown) {
			dragging = true;
			x += movementX;
			y += movementY;
		}
	}}
	on:mouseup={({ offsetX, offsetY }) => {
		mouseDown = false;
		if (!dragging) {
			const [xCoord, yCoord] = screenToMap(offsetX, offsetY);
			if (clickable(xCoord, yCoord)) {
				map[JSON.stringify([xCoord, yCoord])] = !map[JSON.stringify([xCoord, yCoord])];
			}
		}
		dragging = false;
	}}
	{width}
	{height}
>
	<Layer {render} />
</Canvas>

<button on:click={step}>Step</button>

<style>
	:global(canvas) {
		width: 100%;
		height: 100%;
	}

	button {
		position: absolute;
		bottom: 10px;
		right: 10px;
	}
</style>
