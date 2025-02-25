<script lang="ts">
	import { base } from '$app/paths';
	import { type TreeNode, loadData } from '$lib';
	import { onMount, tick } from 'svelte';
	import { browser } from '$app/environment';
	import { Header } from '$lib/components/ui/header';
	import { TreeNodeTooltip } from '$lib/components/ui/tree-node-tooltip/index.js';

	let { nodes } = loadData();

	let containerEl: HTMLDivElement | null = null;
	let imageEl: HTMLImageElement | null = null;
	let imageWrapperEl: HTMLDivElement | null = null; // Reference to the image wrapper
	let tooltipEl: HTMLDivElement | null = null; // Reference to the tooltip element
	let hasLoaded = false;

	let tooltipNode: TreeNode | null = null;
	let tooltipX = 0;
	let tooltipY = 0;

	// State for panning
	let isPanning = false;
	let panStartX = 0;
	let panStartY = 0;
	let panOffsetX = 0;
	let panOffsetY = 0;

	// State for zoom
	let scale = 0.75; // Initial zoom level (almost fully zoomed out)
	const minScale = 0.5; // Minimum zoom out level
	const maxScale = 3; // Maximum zoom in level

	// Base size for nodes
	const baseNodeSize = 20; // Adjust as needed

	// State for search
	let searchTerm = '';
	let searchResults: string[] = [];

	// State for selected nodes
	let selectedNodes: string[] = [];

	// Load saved selected nodes from localStorage on component initialization
	if (browser) {
		const savedSelectedNodes = localStorage.getItem('selectedSkillNodes');

		if (savedSelectedNodes) {
			try {
				selectedNodes = JSON.parse(savedSelectedNodes);
			} catch (error) {
				console.error('Error parsing saved selected nodes:', error);
			}
		}
	}

	// Reactive statement to save selected nodes to localStorage whenever they change
	$: if (browser) {
		localStorage.setItem('selectedSkillNodes', JSON.stringify(selectedNodes));
	}

	// State for filters
	let highlightKeystones = false;
	let highlightNotables = false;
	let highlightSmalls = false;
	let hideUnidentified = false;
	let hideUnselected = false;
	let hideSmall = false;

	// Reactive statement for search
	$: handleSearch(searchTerm);

	// composable filter functions
	function filterSmallNodes(node: TreeNode) {
		return !hideSmall || node.type !== 'small';
	}

	function filterUnselectedNodes(node: TreeNode) {
		return !hideUnselected || !selectedNodes.includes(node.id);
	}

	function filterUnidentifiedNodes(node: TreeNode) {
		return !hideUnidentified || node.description.length > 0;
	}

	const filterFns = [filterSmallNodes, filterUnselectedNodes, filterUnidentifiedNodes];

	// filter nodes using active filters
	function filterNodes(node: TreeNode) {
		return filterFns.every((filterFn) => filterFn(node));
	}

	const NODE_SIZE = {
		notable: 20,
		small: 10,
		keystone: 24
	};

	// calculate node size in pixels based on type
	function getNodeSize(node: TreeNode) {
		return NODE_SIZE[node.type];
	}

	async function activateTooltip(node: TreeNode) {
		tooltipNode = node;

		if (!imageEl || !containerEl) return;

		// Calculate node position relative to the container, accounting for pan offsets
		const nodeX = node.position.x * imageEl.naturalWidth * scale + panOffsetX;
		const nodeY = node.position.y * imageEl.naturalHeight * scale + panOffsetY;

		// Initial tooltip position
		tooltipX = nodeX + 20; // Adjust as needed
		tooltipY = nodeY - 20;

		await tick(); // Wait for the tooltip to render

		if (tooltipEl && containerEl) {
			const tooltipRect = tooltipEl.getBoundingClientRect();
			const containerRect = containerEl.getBoundingClientRect();

			// Compute tooltip position relative to container
			const tooltipLeftInContainer = tooltipRect.left - containerRect.left;
			const tooltipRightInContainer = tooltipLeftInContainer + tooltipRect.width;
			const tooltipTopInContainer = tooltipRect.top - containerRect.top;
			const tooltipBottomInContainer = tooltipTopInContainer + tooltipRect.height;

			// Adjust positions to keep tooltip within container
			if (tooltipRightInContainer > containerRect.width) {
				tooltipX -= tooltipRightInContainer - containerRect.width + 20;
			}

			if (tooltipLeftInContainer < 0) {
				tooltipX += -tooltipLeftInContainer + 20;
			}

			if (tooltipBottomInContainer > containerRect.height) {
				tooltipY -= tooltipBottomInContainer - containerRect.height + 20;
			}

			if (tooltipTopInContainer < 0) {
				tooltipY += -tooltipTopInContainer + 20;
			}
		}
	}

	function handleContainerMousedown(event: MouseEvent) {
		event.preventDefault();

		if (event.button === 0) {
			if (containerEl) {
				containerEl.focus();
			}

			isPanning = true;
			panStartX = event.clientX - panOffsetX;
			panStartY = event.clientY - panOffsetY;
		}
	}

	function toggleNodeSelection(node: TreeNode) {
		if (selectedNodes.includes(node.id)) {
			// Deselect node
			selectedNodes = selectedNodes.filter((id) => id !== node.id);
		} else {
			// Select node
			selectedNodes = [...selectedNodes, node.id];
		}
	}

	function handleMouseMove(event: MouseEvent) {
		if (isPanning && containerEl) {
			panOffsetX = event.clientX - panStartX;
			panOffsetY = event.clientY - panStartY;
			clampPanOffsets();
		}
	}

	function handleMouseUp(event: MouseEvent) {
		if (event.button === 0) {
			isPanning = false;
		}
	}

	function handleMouseEnter(node: TreeNode) {
		if (!isPanning) {
			activateTooltip(node);
		}
	}

	function handleMouseLeave() {
		if (!isPanning) {
			tooltipNode = null;
		}
	}

	function handleWheel(event: WheelEvent) {
		event.preventDefault();

		const zoomIntensity = 0.1;
		const wheel = event.deltaY < 0 ? 1 : -1;
		const oldScale = scale;

		// Calculate new scale
		scale += wheel * zoomIntensity * scale;
		scale = Math.max(minScale, Math.min(maxScale, scale));

		// Adjust pan offsets to zoom relative to the mouse position
		if (containerEl && imageEl) {
			const rect = containerEl.getBoundingClientRect();
			const mouseX = event.clientX - rect.left;
			const mouseY = event.clientY - rect.top;

			const nodeX = (mouseX - panOffsetX) / oldScale;
			const nodeY = (mouseY - panOffsetY) / oldScale;

			panOffsetX = mouseX - nodeX * scale;
			panOffsetY = mouseY - nodeY * scale;

			clampPanOffsets();
		}
	}

	function handleImageLoad() {
		hasLoaded = true;

		if (imageEl && containerEl) {
			const imageWidth = imageEl.naturalWidth * scale;
			const imageHeight = imageEl.naturalHeight * scale;
			const containerWidth = containerEl.clientWidth;
			const containerHeight = containerEl.clientHeight;

			panOffsetX = (containerWidth - imageWidth) / 2;
			panOffsetY = (containerHeight - imageHeight) / 2;
		}
	}

	function handleSearch(text: string) {
		if (!text) {
			searchResults = [];
			return;
		}

		const search = text.toLowerCase();

		searchResults = Object.entries(nodes)
			.filter(
				([_, values]) =>
					values.name.toLowerCase().includes(search) ||
					values.description.some((value) => value.toLowerCase().includes(search))
			)
			.map(([key, _]) => key);
	}

	function clampPanOffsets() {
		if (imageEl && containerEl) {
			const scaledWidth = imageEl.naturalWidth * scale;
			const scaledHeight = imageEl.naturalHeight * scale;
			const containerWidth = containerEl.clientWidth;
			const containerHeight = containerEl.clientHeight;

			const minPanX = containerWidth - scaledWidth;
			const minPanY = containerHeight - scaledHeight;

			if (containerWidth < scaledWidth) {
				panOffsetX = Math.max(minPanX, Math.min(0, panOffsetX));
			} else {
				panOffsetX = (containerWidth - scaledWidth) / 2;
			}

			if (containerHeight < scaledHeight) {
				panOffsetY = Math.max(minPanY, Math.min(0, panOffsetY));
			} else {
				panOffsetY = (containerHeight - scaledHeight) / 2;
			}
		}
	}

	function clearSelectedNodes() {
		selectedNodes = [];

		// Clear localStorage when all nodes are cleared
		if (browser) {
			localStorage.removeItem('selectedSkillNodes');
		}
	}

	// Add event listeners for global mouse events to handle panning
	onMount(() => {
		const handleMove = (event: MouseEvent) => {
			if (isPanning) {
				handleMouseMove(event);
			}
		};

		const handleUp = (event: MouseEvent) => {
			if (isPanning) {
				handleMouseUp(event);
			}
		};

		window.addEventListener('mousemove', handleMove);
		window.addEventListener('mouseup', handleUp);

		return () => {
			window.removeEventListener('mousemove', handleMove);
			window.removeEventListener('mouseup', handleUp);
		};
	});
</script>

<!-- page layout -->
<div class="grid grid-cols-1 grid-rows-[auto_1fr] h-dvh">
	<Header />
	<!-- Tree -->
	<div class="grid grid-rows-1 grid-cols-[20rem_1fr] min-h-0">
		<!-- Left Sidebar -->
		<aside class="h-full grid grid-cols-1 grid-rows-[auto_1fr_1fr] gap-2 p-2 bg-[#111] min-h-0">
			<!-- Toggleable -->
			<div class="space-y-4">
				<div>
					<b class="block underline underline-offset-2">Highlight:</b>
					<div class="flex flex-row gap-2 flex-wrap">
						<label class="whitespace-nowrap">
							<input type="checkbox" bind:checked={highlightKeystones} />
							<span>Keystones</span>
						</label>
						<label class="whitespace-nowrap">
							<input type="checkbox" bind:checked={highlightNotables} />
							<span>Notables</span>
						</label>
						<label class="whitespace-nowrap">
							<input type="checkbox" bind:checked={highlightSmalls} />
							<span>Smalls</span>
						</label>
					</div>
				</div>
				<div>
					<b class="block underline underline-offset-2">Hide:</b>
					<div class="flex flex-row gap-2 flex-wrap">
						<label class="whitespace-nowrap">
							<input type="checkbox" bind:checked={hideUnidentified} />
							<span>Unidentified</span>
						</label>
						<label class="whitespace-nowrap">
							<input type="checkbox" bind:checked={hideUnselected} />
							<span>Unselected</span>
						</label>
						<label class="whitespace-nowrap">
							<input type="checkbox" bind:checked={hideSmall} />
							<span>Smalls</span>
						</label>
					</div>
				</div>
			</div>
			<!-- Search -->
			<div class="min-h-0 grid grid-cols-1 grid-rows-[auto_auto_auto_1fr]">
				<b class="block underline underline-offset-2">Search:</b>
				<input
					class="block rounded px-2 text-black"
					type="text"
					placeholder="Search..."
					bind:value={searchTerm}
				/>
				<span>Found: {searchResults.length}</span>
				<ul class="block min-h-0 overflow-y-auto">
					{#each searchResults as nodeId}
						<li>
							<strong>{nodes[nodeId].name}</strong>
							<ul>
								{#each nodes[nodeId].description as description}
									<li class="text-sm text-[#7d7aad]">{description}</li>
								{/each}
							</ul>
						</li>
					{/each}
				</ul>
			</div>
			<!-- Selected -->
			<div class="min-h-0 grid grid-cols-1 grid-rows-[auto_auto_auto_1fr]">
				<b class="underline underline-offset-2">Selected:</b>
				<div class="flex flex-row justify-between">
					<button class="px-4 border rounded border-white border-solid" onclick={clearSelectedNodes}
						>Clear
					</button>
					<span
						>Selected:
						{selectedNodes.length} / {Object.entries(nodes).filter(
							([_, n]) => n.description.length > 0
						).length}
					</span>
				</div>
				<ul class="block min-h-0 overflow-y-auto">
					{#each selectedNodes as nodeId}
						{#if !nodeId.startsWith('S')}
							<li>
								<strong>{nodes[nodeId].name}</strong>
								<ul>
									{#each nodes[nodeId].description as description}
										<li class="text-sm text-[#7d7aad]">{description}</li>
									{/each}
								</ul>
							</li>
						{/if}
					{/each}
				</ul>
			</div>
		</aside>
		<!-- Tree View -->
		<div class="bg-black">
			<!-- Skill Tree Container -->
			<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
			<div
				bind:this={containerEl}
				class="overflow-hidden relative block w-full h-full outline-none"
				role="application"
				tabindex="-1"
				onmousedown={handleContainerMousedown}
				onwheel={handleWheel}
			>
				<div
					bind:this={imageWrapperEl}
					class="absolute top-0 left-0"
					style="
			  width: {imageEl ? imageEl.naturalWidth * scale + 'px' : 'auto'};
			  height: {imageEl ? imageEl.naturalHeight * scale + 'px' : 'auto'};
			  transform: translate({panOffsetX}px, {panOffsetY}px);
			  user-select: none;
			  cursor: {isPanning ? 'grabbing' : 'grab'};
		  "
				>
					<img
						class="pointer-events-none max-w-none"
						bind:this={imageEl}
						onload={handleImageLoad}
						src="{base}/skill-tree.png"
						alt="Interactive"
						draggable="false"
						style="
				  width: {imageEl ? imageEl.naturalWidth * scale + 'px' : 'auto'};
				  height: {imageEl ? imageEl.naturalHeight * scale + 'px' : 'auto'};
			  "
					/>

					<!-- Display hoverable regions with lighter color -->
					{#if hasLoaded}
						{#each Object.values(nodes).filter(filterNodes) as node}
							<!-- svelte-ignore a11y_no_static_element_interactions -->
							<!-- svelte-ignore a11y_click_events_have_key_events -->
							<div
								class:keystone={node.type === 'keystone'}
								class:notable={node.type === 'notable'}
								class:small={node.type === 'small'}
								class:unidentified={node.description.length === 0}
								class:search-result={searchResults.includes(node.id)}
								class:selected={selectedNodes.includes(node.id)}
								class:highlighted-keystone={highlightKeystones && node.type === 'keystone'}
								class:highlighted-notable={highlightNotables && node.type === 'notable'}
								class:highlighted-small={highlightSmalls && node.type === 'small'}
								style="
									height: {getNodeSize(node) * scale}px;
									width: {getNodeSize(node) * scale}px;
									left: {node.position.x * imageEl.naturalWidth * scale - (getNodeSize(node) * scale) / 2}px;
									top: {node.position.y * imageEl.naturalHeight * scale - (getNodeSize(node) * scale) / 2}px;
								"
								onmousedown={(event) => event.stopPropagation()}
								onclick={() => toggleNodeSelection(node)}
								onmouseenter={() => handleMouseEnter(node)}
								onmouseleave={handleMouseLeave}
							></div>
						{/each}
					{/if}
				</div>

				<!-- Tooltip displayed when a region is hovered -->
				{#if tooltipNode != null}
					<!-- Tooltip wrapper for absolute position -->
					<div
						bind:this={tooltipEl}
						class="absolute w-[400px] pointer-events-none"
						style="left: {tooltipX}px; top: {tooltipY}px;"
					>
						<TreeNodeTooltip node={tooltipNode} />
					</div>
				{/if}
			</div>
		</div>
	</div>
</div>

<style lang="postcss">
	.small,
	.notable,
	.keystone {
		position: absolute;
		border-radius: 50%;
		pointer-events: auto;
	}

	.notable {
		background-color: rgba(255, 255, 0, 0.2);
	}

	.notable.unidentified {
		background-color: rgba(255, 100, 100, 0.2);
		border-color: rgba(255, 100, 100, 1);
	}

	.keystone {
		background-color: rgba(100, 255, 100, 0.2);
	}

	.keystone.unidentified {
		background-color: rgba(255, 0, 100, 0.2);
		border-color: rgba(255, 0, 100, 1);
	}

	.small {
		background-color: rgba(255, 255, 255, 0.2);
	}

	.small.unidentified {
		background-color: rgba(255, 255, 255, 0.2);
		border-color: rgba(255, 100, 100, 1);
	}

	.notable.selected {
		background-color: rgba(255, 255, 0, 0.6);
	}

	.keystone.selected {
		background-color: rgba(0, 255, 0, 0.6);
	}

	.small.selected {
		background-color: rgba(255, 255, 255, 0.6);
	}

	.highlighted-keystone {
		border: 2px solid green;
	}

	.highlighted-notable {
		border: 1px solid yellow;
	}

	.highlighted-small {
		border: 1px solid yellow;
	}

	@keyframes glow {
		0% {
			box-shadow: 0 0 5px rgba(255, 0, 0, 0.5);
		}
		50% {
			box-shadow: 0 0 15px rgba(255, 0, 0, 1);
		}
		100% {
			box-shadow: 0 0 5px rgba(255, 0, 0, 0.5);
		}
	}

	.search-result {
		border: 3px solid rgba(255, 0, 0, 0.8);
		animation: glow 2s infinite;
	}
</style>
