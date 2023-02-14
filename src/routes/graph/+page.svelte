<script lang="ts">
	let sizeX = 3;
	let sizeY = 3;
	let algorithm: 'dijkstra' | 'aStar' = 'dijkstra';
	let diagonals = false;

	let cells: Cell[][] = [];

	type Cell = {
		x: number;
		y: number;
		state: 'unvisited' | 'opened' | 'closed' | 'wall' | 'path';
		gCost: number;
		hCost: number;
		fCost: number;
		neighbors: Cell[];
		cameFrom: Cell | null;
		class: string | null;
	};

	$: sizeX, sizeY, diagonals, updateSize();

	let goal: Cell;

	function updateSize() {
		sizeX = Math.max(sizeX, 1);
		sizeY = Math.max(sizeY, 1);

		// Remove extra cells
		cells = cells.filter((_, y) => y < sizeY);
		for (let y = 0; y < sizeY; y++) {
			cells[y] = !cells[y] ? [] : cells[y].filter((_, x) => x < sizeX);
		}

		// Add new cells
		for (let y = 0; y < sizeY; y++) {
			for (let x = 0; x < sizeX; x++) {
				if (cells[y][x]) continue;

				cells[y][x] = {
					state: 'unvisited',
					gCost: 0,
					hCost: 0,
					fCost: 0,
					x,
					y,
					neighbors: [],
					cameFrom: null,
					class: null
				};
			}
		}

		// Update the position and neighbors of each cell
		for (const row of cells) {
			for (const cell of row) {
				cell.neighbors = [];

				for (let deltaX = -1; deltaX <= 1; deltaX++) {
					for (let deltaY = -1; deltaY <= 1; deltaY++) {
						let x = cell.x + deltaX;
						let y = cell.y + deltaY;
						if (x < 0 || x >= sizeX || y < 0 || y >= sizeY || (x == 0 && y == 0)) continue;

						if (!diagonals && Math.abs(deltaX) === 1 && Math.abs(deltaY) === 1) continue;

						cell.neighbors.push(cells[y][x]);
					}
				}
			}
		}

		goal = cells[sizeY - 1][sizeX - 1];

		reset();
	}

	function getColor(state: string) {
		return {
			unvisited: 'white',
			opened: 'rgb(100, 250, 100)',
			closed: 'rgb(250, 100, 100)',
			wall: 'gray',
			path: 'rgb(100, 250, 250)'
		}[state];
	}

	function toggleWall(x: number, y: number) {
		if (x === 0 && y === 0) return;

		cells[y][x].state = cells[y][x].state === 'wall' ? 'unvisited' : 'wall';

		reset();
	}

	function dist(a: Cell, b: Cell) {
		return Math.floor(Math.sqrt((a.x - b.x) ** 2 + (a.y - b.y) ** 2) * 10);
	}

	function reset() {
		currentCode = algorithm;
		currentCodeLine = -1;
		nextStep = null;
		if (seekInterval) clearInterval(seekInterval);

		for (let y = 0; y < sizeY; y++) {
			for (let x = 0; x < sizeX; x++) {
				if (cells[y][x].state !== 'wall') cells[y][x].state = 'unvisited';

				cells[y][x].gCost = Infinity;
				cells[y][x].hCost = dist(cells[y][x], goal);
				cells[y][x].fCost = Infinity;
				cells[y][x].cameFrom = null;
				cells[y][x].class = null;
			}
		}

		cells[0][0].state = 'unvisited';
		cells[0][0].gCost = 0;
		cells[0][0].fCost = 0;
	}

	function showCost(cost: number) {
		return cost == Infinity ? '∞' : cost;
	}

	let nextStep: (() => void) | null = null;
	function step(line: number) {
		currentCodeLine = line;
		return new Promise<void>((resolve) => {
			nextStep = () => {
				nextStep = null;
				resolve();
			};
		});
	}

	async function runAlgorithm() {
		const openSet = new Set([cells[0][0]]);
		cells[0][0].state = 'opened';
		await step(1);

		while (openSet.size > 0) {
			await step(3);
			let node: Cell | null = null;
			for (const cell of openSet) {
				if (node === null || node.fCost > cell.fCost) {
					node = cell;
				}
			}

			if (node === null) throw 'This should never happen.';

			cells[node.y][node.x].class = 'node';
			await step(4);

			await step(6);
			if (node == goal) {
				await step(7);
				return await tracePath();
			}

			openSet.delete(node);
			cells[node.y][node.x].state = 'closed';
			await step(10);

			for (const neighbor of node.neighbors) {
				if (neighbor.state === 'wall' || neighbor.state === 'closed') continue;

				const { x, y } = neighbor;
				cells[y][x].class = 'neighbor';
				await step(12);

				await step(13);
				const gCost = node.gCost + dist(node, neighbor);

				await step(15);
				if (gCost < neighbor.gCost) {
					cells[y][x].gCost = gCost;
					await step(16);
					cells[y][x].fCost = gCost + (algorithm === 'dijkstra' ? 0 : neighbor.hCost);
					await step(17);
					cells[y][x].cameFrom = node;
					await step(18);

					await step(20);
					if (!openSet.has(neighbor)) {
						openSet.add(neighbor);
						cells[y][x].state = 'opened';
						await step(21);
					}
				}

				cells[y][x].class = null;
			}

			cells[node.y][node.x].class = null;
		}

		await step(27);
		return false;
	}

	async function tracePath() {
		currentCode = 'tracePath';
		await step(1);

		let current: Cell | null = goal;
		cells[current.y][current.x].class = 'node';
		await step(2);

		while (current != null) {
			await step(4);
			cells[current.y][current.x].state = 'path';
			await step(5);
			cells[current.y][current.x].class = null;
			current = current.cameFrom;
			if (current) cells[current.y][current.x].class = 'node';
			await step(6);
		}
		await step(4);

		await step(9);
	}

	async function findPath() {
		reset();
		await runAlgorithm();
		currentCodeLine = -1;
	}

	const psuedoCodes = {
		dijkstra: `function dijkstra(start, goal) {
	openSet = [start];

	while (openSet.length > 0) {
		node = {lowest f-cost node in openSet};

		if (node == goal) {
			return tracePath(goal);
		}

		openSet.remove(node);

		for (neighbor of node.neighbors) {
			gCost = node.gCost + dist(node, neighbor)

			if (gCost < neighbor.gCost) {
				neighbor.gCost = gCost;
				neighbor.fCost = gCost;
				neighbor.cameFrom = node;

				if (neighbor not in openSet) {
					openSet.add(neighbor);
				}
			}
		}
	}

	return false;
}`,
		aStar: `function aStar(start, goal) {
	openSet = [start];

	while (openSet.length > 0) {
		node = {lowest f-cost node in openSet};

		if (node == goal) {
			return tracePath(goal);
		}

		openSet.remove(node);

		for (neighbor of node.neighbors) {
			gCost = node.gCost + dist(node, neighbor)

			if (gCost < neighbor.gCost) {
				neighbor.gCost = gCost;
				neighbor.fCost = gCost + neighbor.hCost;
				neighbor.cameFrom = node;

				if (neighbor not in openSet) {
					openSet.add(neighbor);
				}
			}
		}
	}

	return false;
}`,
		tracePath: `function tracePath(goal) {
	current = goal;
	path = [];
	
	while (current is not null) {
		path.add(current);
		current = current.cameFrom;
	}

	return path;
}`
	};

	let currentCodeLine = -1;
	let currentCode: keyof typeof psuedoCodes = algorithm;
	$: psuedoCode = psuedoCodes[currentCode];
	$: lastLine = psuedoCode.split('\n').length - 1;

	let seekInterval: number | null = null;
	function seekTo(i: number) {
		if (nextStep) nextStep();

		if (seekInterval) clearInterval(seekInterval);
		seekInterval = setInterval(() => {
			if (currentCodeLine === i) {
				if (seekInterval) clearInterval(seekInterval);
				return;
			}

			if (nextStep) nextStep();
		}, 25) as unknown as number;
	}
</script>

<div class="content">
	<div class="grid" style:grid-template-columns={`repeat(${sizeX}, minmax(50px, 100px))`}>
		{#each cells as row, y}
			{#each row as cell, x}
				<!-- svelte-ignore a11y-click-events-have-key-events -->
				<div
					style:background={getColor(cell.state)}
					class={cell.class}
					on:click={() => toggleWall(x, y)}
				>
					{#if cell.state !== 'wall'}
						<div class="cost">{showCost(cell.fCost)}</div>
						<div class="g-cost">{showCost(cell.gCost)}</div>
						{#if algorithm === 'aStar'}
							<div class="h-cost">{cell.hCost}</div>
						{/if}
						{#if cell.cameFrom}
							{@const from = cell.cameFrom}
							{@const radians = Math.atan2(from.y - cell.y, from.x - cell.x)}
							<div class="from" style:transform={`rotateZ(${radians}rad) translateX(18px)`}>ᐳ</div>
						{/if}
						{#if x === 0 && y === 0}
							<div class="bl">🏁</div>
						{/if}
						{#if cell === goal}
							<div class="bl">🎯</div>
						{/if}
					{/if}
				</div>
			{/each}
		{/each}
	</div>

	<div class="sidebar">
		<div>
			<span>Size: </span>
			<input type="number" bind:value={sizeX} /> by
			<input type="number" bind:value={sizeY} />
		</div>
		<div>
			<label for="algorithm">Algorithm: </label>
			<select id="algorithm" bind:value={algorithm} on:change={reset}>
				<option value="dijkstra">Dijkstra</option>
				<option value="aStar">A*</option>
			</select>

			<label for="diagonals">Diagonals: </label>
			<input id="diagonals" type="checkbox" bind:checked={diagonals} />
		</div>
		<div>
			<button on:click={findPath}>Find Path</button>
			<button
				on:click={() => {
					if (nextStep) nextStep();
				}}>Step</button
			>
		</div>
		<pre style="tab-size: 4;">
{#each psuedoCode.split('\n') as line, i}
				<span
					>{(i + 1).toString().padStart(2, '0') +
						'\t' +
						'\t'.repeat(line.length - line.trimStart().length)}</span
				><span
					class="line"
					style:background={currentCodeLine === i ? 'rgba(250, 250, 50, 0.5)' : ''}
					on:click={() => seekTo(i)}>{@html line.trimStart() + (i !== lastLine ? '\n' : '')}</span
				>
			{/each}
</pre>
	</div>
</div>

<style>
	:global(html, body, .content) {
		min-height: 100vh;
		margin: 0;
	}

	.content {
		display: grid;
		grid-template-columns: 1.5fr 1fr;
	}

	pre {
		margin: 0;
	}

	.sidebar {
		display: flex;
		flex-direction: column;
		gap: 5px;
		min-width: 0;
		padding: 10px;

		border-left: black 2px solid;
		box-shadow: 0px 0px 6px rgba(0, 0, 0, 0.5);
	}

	input[type='number'] {
		width: 3em;
	}

	.grid {
		justify-self: center;
		align-self: center;

		display: grid;
		gap: 1px;
		box-shadow: 0px 0px 6px rgba(0, 0, 0, 0.5);
		margin: 25px;
		padding: 2px;

		max-width: calc(60vw - 50px);
		overflow: auto;
	}

	.grid > div {
		background-color: white;
		aspect-ratio: 1 / 1;
		display: flex;
		justify-content: center;
		align-items: center;
		position: relative;
		user-select: none;
		cursor: pointer;
		outline: 2px solid black;
	}

	.grid > div:first-child {
		cursor: initial;
	}

	.g-cost {
		position: absolute;
		top: 2px;
		left: 5px;
	}

	.h-cost {
		position: absolute;
		top: 2px;
		right: 5px;
	}

	.from {
		position: absolute;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.node::before {
		content: '⚡';
		position: absolute;
		bottom: 2px;
		right: 2px;
		font-size: 125%;
		color: transparent;
		text-shadow: 0 0 rgb(250, 250, 50);
		filter: drop-shadow(0px 0px 2px black);
	}

	.neighbor::before {
		content: '🏠';
		position: absolute;
		bottom: 2px;
		right: 2px;
		font-size: 125%;
	}

	.bl {
		position: absolute;
		bottom: 2px;
		left: 2px;
		font-size: 125%;
	}
</style>
