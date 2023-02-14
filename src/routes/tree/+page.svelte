<script lang="ts">
	import { onMount } from 'svelte';

	let algorithm: 'minimax' | 'alphabeta' = 'minimax';
	let maxDepth = 3;

	let exampleDialog: HTMLDialogElement;

	let nodes: Node[] = [];

	type Node = {
		value: number | null;
		children: Node[];
		x: number;
		y: number;
		depth: number;
		color: string;
		i: number;
	};

	$: maxDepth, updateDepth();

	function updateDepth() {
		maxDepth = Math.max(0, Math.min(maxDepth, 5));

		nodes = [];

		for (let depth = 0; depth < maxDepth; depth++) {
			const count = 2 ** depth;
			for (let i = 0; i < count; i++) {
				let node = {
					value: Math.floor(Math.random() * 11) - 5,
					children: [],
					x: ((i + 0.5) / count) * 2,
					y: depth / (maxDepth - 1),
					depth,
					color: 'black',
					i: 0
				};

				nodes.push(node);

				if (depth == 0) continue;

				nodes[Math.floor((nodes.length - 2) / 2)].children.push(node);
			}
		}

		nodes = nodes;
		reset();
	}

	function inputNumber(node: Node) {
		const value = parseInt(prompt("Node's new value?") ?? '');
		if (Number.isNaN(value)) return;

		nodes[node.i].value = value;

		reset();
	}

	function reset() {
		currentCode = algorithm;
		currentCodeLine = -1;
		nextStep = null;
		if (seekInterval) clearInterval(seekInterval);

		for (const [i, node] of nodes.entries()) {
			if (node.depth !== maxDepth - 1) {
				node.value = null;
			}

			node.i = i;
			nodes[i].color = 'black';
		}
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

	const variables: Record<string, any> = {};

	async function runAlgorithm() {
		if (algorithm === 'minimax') {
			async function minimax(node: Node, depth: number, maximizing: boolean): Promise<number> {
				variables['maximizing'] = maximizing;
				variables['depth'] = depth;
				delete variables['value'];

				nodes[node.i].color = 'rgb(50, 250, 250)';
				await step(1);
				if (depth === 0 || node.children.length === 0) {
					if (node.value === null) throw 'This should never happen!';
					await step(2);
					nodes[node.i].color = 'black';
					return node.value;
				}

				if (maximizing) {
					await step(5);
					let value = -Infinity;
					variables['value'] = value;
					await step(6);

					for (const child of node.children) {
						nodes[node.i].color = 'rgb(50, 50, 250)';
						nodes[child.i].color = 'rgb(50, 250, 250)';
						await step(8);
						await step(9);
						const newValue = await minimax(child, depth - 1, false);
						variables['value'] = value;
						nodes[node.i].color = 'rgb(50, 250, 250)';
						await step(9);
						value = Math.max(value, newValue);

						variables['maximizing'] = maximizing;
						variables['depth'] = depth;

						variables['value'] = value;
						nodes[node.i].value = value;
					}

					await step(12);
					nodes[node.i].color = 'black';
					return value;
				} else {
					await step(13);
					let value = Infinity;
					variables['value'] = value;
					await step(14);

					for (const child of node.children) {
						nodes[node.i].color = 'rgb(50, 50, 250)';
						nodes[child.i].color = 'rgb(50, 250, 250)';
						await step(16);
						await step(17);
						const newValue = await minimax(child, depth - 1, true);
						variables['value'] = value;
						nodes[node.i].color = 'rgb(50, 250, 250)';
						await step(17);
						value = Math.min(value, newValue);

						variables['maximizing'] = maximizing;
						variables['depth'] = depth;

						variables['value'] = value;
						nodes[node.i].value = value;
					}

					await step(20);
					nodes[node.i].color = 'black';
					return value;
				}
			}

			await step(24);
			await minimax(nodes[0], maxDepth - 1, true);
		} else {
			function prune(node: Node) {
				nodes[node.i].color = 'rgb(200, 200, 200)';

				for (const child of node.children) {
					prune(child);
				}
			}

			async function alphaBeta(
				node: Node,
				depth: number,
				a: number,
				b: number,
				maximizing: boolean
			): Promise<number> {
				variables['maximizing'] = maximizing;
				variables['depth'] = depth;
				variables['a'] = a;
				variables['b'] = b;
				delete variables['value'];

				nodes[node.i].color = 'rgb(50, 250, 250)';
				await step(1);
				if (depth === 0 || node.children.length === 0) {
					if (node.value === null) throw 'This should never happen!';
					await step(2);
					nodes[node.i].color = 'black';
					return node.value;
				}

				if (maximizing) {
					await step(5);
					let value = -Infinity;
					variables['value'] = value;
					await step(6);

					let lastIndex = Infinity;
					for (const [i, child] of node.children.entries()) {
						nodes[node.i].color = 'rgb(50, 50, 250)';
						nodes[child.i].color = 'rgb(50, 250, 250)';
						await step(8);
						await step(9);
						const newValue = await alphaBeta(child, depth - 1, a, b, false);
						variables['value'] = value;
						nodes[node.i].color = 'rgb(50, 250, 250)';
						await step(9);
						value = Math.max(value, newValue);

						variables['maximizing'] = maximizing;
						variables['depth'] = depth;
						variables['a'] = a;
						variables['b'] = b;

						variables['value'] = value;
						nodes[node.i].value = value;

						await step(11);
						if (value > b) {
							lastIndex = i;
							await step(12);
							break;
						}
						a = Math.max(a, value);
						variables['a'] = a;
						await step(14);
					}

					for (let i = lastIndex + 1; i < node.children.length; i++) {
						prune(node.children[i]);
					}

					await step(17);
					nodes[node.i].color = 'black';
					return value;
				} else {
					await step(18);
					let value = Infinity;
					variables['value'] = value;
					await step(19);

					let lastIndex = Infinity;
					for (const [i, child] of node.children.entries()) {
						nodes[node.i].color = 'rgb(50, 50, 250)';
						nodes[child.i].color = 'rgb(50, 250, 250)';
						await step(21);
						await step(22);
						const newValue = await alphaBeta(child, depth - 1, a, b, true);
						variables['value'] = value;
						nodes[node.i].color = 'rgb(50, 250, 250)';
						await step(22);
						value = Math.min(value, newValue);

						variables['maximizing'] = maximizing;
						variables['depth'] = depth;
						variables['a'] = a;
						variables['b'] = b;

						variables['value'] = value;
						nodes[node.i].value = value;

						await step(24);
						if (value < a) {
							lastIndex = i;
							await step(25);
							break;
						}
						b = Math.min(b, value);
						variables['b'] = b;
						await step(27);
					}

					for (let i = lastIndex + 1; i < node.children.length; i++) {
						prune(node.children[i]);
					}

					await step(30);
					nodes[node.i].color = 'black';
					return value;
				}
			}

			await step(34);
			await alphaBeta(nodes[0], maxDepth - 1, -Infinity, Infinity, true);
		}
	}

	async function searchTree() {
		reset();
		await runAlgorithm();
		currentCodeLine = -1;
	}

	const psuedoCodes = {
		minimax: `function minimax(node, depth, maximizing) {
	if (depth == 0 or node is a terminal node) {
		return the heuristic value of node
	}

	if (maximizing) {
		value = −∞

		for (child of node) {
			value = max(value, minimax(child, depth - 1, false))
		}

		return value
	} else { /* minimizing player */
		value = +∞

		for (child of node) {
			value = min(value, minimax(child, depth - 1, true))
		}

		return value
	}
}

minimax(root, depth, true)`,
		alphabeta: `function alphaBeta(node, depth, a, b, maximizing) {
	if (depth == 0 or node is a terminal node) {
		return the heuristic value of node
	}

	if (maximizing) {
		value = −∞

		for (child of node) {
			value = max(value, alphaBeta(child, depth - 1, a, b, false))

			if (value > b) {
				break
			}
			a = max(a, value)
		}

		return value
	} else { /* minimizing player */
		value = +∞

		for (child of node) {
			value = min(value, alphaBeta(child, depth - 1, a, b, true))

			if (value < a) {
				break
			}
			b = min(b, value)
		}

		return value
	}
}

alphaBeta(root, depth, −∞, +∞, true)`
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

	function displayVar(value: any) {
		if (value === Infinity) {
			return '+∞';
		} else if (value === -Infinity) {
			return '−∞';
		} else {
			return value.toString();
		}
	}

	function loadExample(value: string) {
		const numbers = value.split(',').map((a) => parseInt(a));
		maxDepth = Math.log2(numbers.length) + 1;

		setTimeout(() => {
			const last = nodes.filter((node) => node.depth === maxDepth - 1);
			for (let i = 0; i < numbers.length; i++) {
				nodes[last[i].i].value = numbers[i];
			}

			reset();
		}, 0);
	}

	onMount(() => {
		const listener = (event: KeyboardEvent) => {
			if (event.ctrlKey && event.key === ']') {
				const response = prompt('Load');
				if (!response) return;

				loadExample(response);
			}
		};

		document.addEventListener('keypress', listener);

		return () => document.removeEventListener('keypress', listener);
	});
</script>

<div class="content">
	<svg class="grid" viewBox="-0.065 -0.065 2.13 1.13">
		{#each Object.entries(variables) as entry, i}
			<text
				font-size="0.075"
				text-anchor="end"
				stroke="white"
				stroke-width="0.01"
				x="2.05"
				y={i * 0.075}>{entry[0]} = {displayVar(entry[1])}</text
			>
		{/each}
		{#each nodes as node}
			{#each node.children as child}
				<path
					d={`M${node.x},${node.y} L${child.x},${child.y}`}
					stroke={child.color}
					stroke-width="0.01"
					opacity="0.5"
				/>
			{/each}
		{/each}
		{#each nodes as node}
			{@const { x, y } = node}
			<!-- svelte-ignore a11y-click-events-have-key-events -->
			<circle
				cx={x}
				cy={y}
				r="0.06"
				fill="white"
				stroke={node.color}
				stroke-width="0.01"
				style:cursor={node.depth === maxDepth - 1 ? 'text' : ''}
				on:click={() => {
					if (node.depth === maxDepth - 1) inputNumber(node);
				}}
			/>
			<text
				class="value"
				{x}
				{y}
				font-size="0.075"
				dominant-baseline="central"
				text-anchor="middle"
				fill={node.color}
			>
				{node.value ?? ''}
			</text>
			{#if node.depth < maxDepth - 1}
				<text
					{x}
					y={y + 0.12}
					font-size="0.075"
					dominant-baseline="central"
					text-anchor="middle"
					fill={node.color}
					stroke="white"
					stroke-width="0.01"
				>
					{node.depth % 2 === 0 ? 'MAX' : 'MIN'}
				</text>
			{/if}
		{/each}
	</svg>
	<div class="sidebar">
		<div>
			<label for="depth">Depth: </label>
			<input id="depth" type="number" bind:value={maxDepth} />
		</div>
		<div>
			<label for="algorithm">Algorithm: </label>
			<select id="algorithm" bind:value={algorithm} on:change={reset}>
				<option value="minimax">Minimax</option>
				<option value="alphabeta">Alpha-beta Pruning</option>
			</select>
		</div>
		<div>
			<button on:click={searchTree}>Search Tree</button>
			<button
				on:click={() => {
					if (nextStep) nextStep();
				}}>Step</button
			>
			<button on:click={() => exampleDialog.showModal()}>Examples</button>
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
	<dialog bind:this={exampleDialog} on:click={() => exampleDialog.close()}>
		<button on:click={() => loadExample('1,3,-6,0')}>Example 1</button>
		<button on:click={() => loadExample('2,-5,0,-3')}>Example 2</button>
		<button on:click={() => loadExample('4,8,9,3,2,-2,9,-1')}>Example 3</button>
	</dialog>
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
		/* box-shadow: 0px 0px 6px rgba(0, 0, 0, 0.5); */
	}

	input[type='number'] {
		width: 3em;
	}

	.grid {
		justify-self: center;
		align-self: center;

		/* box-shadow: 0px 0px 6px rgba(0, 0, 0, 0.5); */
		margin: 25px;
		padding: 2px;

		max-width: calc(60vw - 50px);
		overflow: auto;
	}

	text {
		paint-order: stroke;
	}

	text.value {
		pointer-events: none;
		filter: brightness(0.5);
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
