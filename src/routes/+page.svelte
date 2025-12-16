<script lang="ts">
	import { parse as parseToml, stringify as stringifyToml } from 'smol-toml';

	interface Item {
		id: string;
		name: string;
		tierId: string | null;
	}

	interface Tier {
		id: string;
		name: string;
	}

	let editMode = $state(true);
	let draggedItem = $state<Item | null>(null);
	let newItemName = $state('');
	let newTierName = $state('');
	let title = $state('Tier List');
	let importText = $state('');
	let showImportModal = $state(false);

	let tiers = $state<Tier[]>([
		{ id: 's', name: 'S' },
		{ id: 'a', name: 'A' },
		{ id: 'b', name: 'B' },
		{ id: 'c', name: 'C' },
		{ id: 'd', name: 'D' },
		{ id: 'f', name: 'F' }
	]);

	let items = $state<Item[]>([
		{ id: '1', name: 'Item 1', tierId: null },
		{ id: '2', name: 'Item 2', tierId: null },
		{ id: '3', name: 'Item 3', tierId: null },
		{ id: '4', name: 'Item 4', tierId: null },
		{ id: '5', name: 'Item 5', tierId: null }
	]);

	function handleDragStart(item: Item) {
		draggedItem = item;
	}

	function handleDragOver(event: DragEvent) {
		event.preventDefault();
	}

	function handleDrop(tierId: string | null) {
		if (draggedItem) {
			draggedItem.tierId = tierId;
			draggedItem = null;
		}
	}

	function addItem() {
		if (newItemName.trim()) {
			items.push({
				id: Date.now().toString(),
				name: newItemName.trim(),
				tierId: null
			});
			newItemName = '';
		}
	}

	function removeItem(id: string) {
		items = items.filter((item) => item.id !== id);
	}

	function addTier() {
		if (newTierName.trim()) {
			const newTier: Tier = {
				id: Date.now().toString(),
				name: newTierName.trim()
			};
			tiers.push(newTier);
			newTierName = '';
		}
	}

	function removeTier(id: string) {
		items.forEach((item) => {
			if (item.tierId === id) {
				item.tierId = null;
			}
		});
		tiers = tiers.filter((tier) => tier.id !== id);
	}

	function getTierColorClass(index: number, total: number): string {
		const colors = [
			'bg-red-500',
			'bg-orange-500',
			'bg-yellow-500',
			'bg-lime-500',
			'bg-green-500',
			'bg-cyan-500'
		];

		if (total === 1) return colors[0];

		const position = index / (total - 1);
		const colorIndex = Math.round(position * (colors.length - 1));

		return colors[Math.min(colorIndex, colors.length - 1)];
	}

	function moveTierUp(index: number) {
		if (index > 0) {
			[tiers[index - 1], tiers[index]] = [tiers[index], tiers[index - 1]];
		}
	}

	function moveTierDown(index: number) {
		if (index < tiers.length - 1) {
			[tiers[index], tiers[index + 1]] = [tiers[index + 1], tiers[index]];
		}
	}

	function exportToToml() {
		const data: Record<string, any> = {
			title: title
		};

		tiers.forEach((tier) => {
			const tierItems = getItemsForTier(tier.id);
			data[tier.name] = {
				items: tierItems.map((item) => item.name)
			};
		});

		const unrankedItems = getItemsForTier(null);
		if (unrankedItems.length > 0) {
			data['Unranked'] = {
				items: unrankedItems.map((item) => item.name)
			};
		}

		const tomlString = stringifyToml(data);
		const blob = new Blob([tomlString], { type: 'text/plain' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = 'tierlist.toml';
		a.click();
		URL.revokeObjectURL(url);
	}

	function importFromToml() {
		try {
			const data = parseToml(importText);

			if (data.title && typeof data.title === 'string') {
				title = data.title;
			}

			const newTiers: Tier[] = [];
			const newItems: Item[] = [];
			let itemIdCounter = 1;

			Object.keys(data).forEach((key) => {
				if (key === 'title') return;

				const section = data[key];
				if (!section || typeof section !== 'object') return;

				const sectionItems = section.items;
				if (!Array.isArray(sectionItems)) return;

				if (key === 'Unranked') {
					sectionItems.forEach((itemName) => {
						if (typeof itemName === 'string') {
							newItems.push({
								id: (itemIdCounter++).toString(),
								name: itemName,
								tierId: null
							});
						}
					});
				} else {
					const tierId = key.toLowerCase().replace(/\s+/g, '-');
					newTiers.push({
						id: tierId,
						name: key
					});

					sectionItems.forEach((itemName) => {
						if (typeof itemName === 'string') {
							newItems.push({
								id: (itemIdCounter++).toString(),
								name: itemName,
								tierId: tierId
							});
						}
					});
				}
			});

			tiers = newTiers;
			items = newItems;
			showImportModal = false;
			importText = '';
		} catch (error) {
			alert('Error parsing TOML: ' + (error instanceof Error ? error.message : String(error)));
		}
	}

	function getItemsForTier(tierId: string | null) {
		return items.filter((item) => item.tierId === tierId);
	}
</script>

<div class="min-h-screen bg-gray-900 text-white p-8">
	<div class="max-w-7xl mx-auto">
		<div class="flex justify-between items-center mb-8">
			{#if editMode}
				<input
					type="text"
					bind:value={title}
					class="text-4xl font-bold bg-transparent border-b-2 border-gray-600 focus:border-blue-500 focus:outline-none px-2"
				/>
			{:else}
				<h1 class="text-4xl font-bold">{title}</h1>
			{/if}
			<button
				onclick={() => (editMode = !editMode)}
				class="px-6 py-2 bg-blue-600 hover:bg-blue-700 rounded-lg transition-colors"
			>
				{editMode ? 'Present Mode' : 'Edit Mode'}
			</button>
		</div>

		{#if editMode}
			<div class="mb-8 space-y-4">
				<div class="flex gap-4">
					<div class="flex-1">
						<h2 class="text-xl font-semibold mb-2">Add New Item</h2>
						<div class="flex gap-2">
							<input
								type="text"
								bind:value={newItemName}
								onkeydown={(e) => e.key === 'Enter' && addItem()}
								placeholder="Enter item name"
								class="flex-1 px-4 py-2 bg-gray-800 border border-gray-700 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
							/>
							<button
								onclick={addItem}
								class="px-6 py-2 bg-green-600 hover:bg-green-700 rounded-lg transition-colors"
							>
								Add Item
							</button>
						</div>
					</div>

					<div class="flex-1">
						<h2 class="text-xl font-semibold mb-2">Add New Tier</h2>
						<div class="flex gap-2">
							<input
								type="text"
								bind:value={newTierName}
								onkeydown={(e) => e.key === 'Enter' && addTier()}
								placeholder="Enter tier name"
								class="flex-1 px-4 py-2 bg-gray-800 border border-gray-700 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
							/>
							<button
								onclick={addTier}
								class="px-6 py-2 bg-green-600 hover:bg-green-700 rounded-lg transition-colors"
							>
								Add Tier
							</button>
						</div>
					</div>
				</div>

				<div class="flex gap-4 justify-center">
					<button
						onclick={exportToToml}
						class="px-6 py-2 bg-purple-600 hover:bg-purple-700 rounded-lg transition-colors"
					>
						Export to TOML
					</button>
					<button
						onclick={() => (showImportModal = true)}
						class="px-6 py-2 bg-purple-600 hover:bg-purple-700 rounded-lg transition-colors"
					>
						Import from TOML
					</button>
				</div>
			</div>
		{/if}

		<div class="space-y-2">
			{#each tiers as tier, index (tier.id)}
				<div class="flex border border-gray-700 rounded-lg overflow-hidden">
					{#if editMode}
						<div class="flex flex-col w-12 bg-gray-700">
							<button
								onclick={() => moveTierUp(index)}
								disabled={index === 0}
								class="flex-1 hover:bg-gray-600 transition-colors disabled:opacity-30 disabled:cursor-not-allowed flex items-center justify-center"
								title="Move up"
							>
								▲
							</button>
							<button
								onclick={() => moveTierDown(index)}
								disabled={index === tiers.length - 1}
								class="flex-1 hover:bg-gray-600 transition-colors disabled:opacity-30 disabled:cursor-not-allowed flex items-center justify-center"
								title="Move down"
							>
								▼
							</button>
						</div>
					{/if}
					<div
						class="w-24 flex items-center justify-center font-bold text-2xl {getTierColorClass(index, tiers.length)}"
					>
						{tier.name}
					</div>
					<div
						class="flex-1 bg-gray-800 p-4 min-h-[100px]"
						ondragover={handleDragOver}
						ondrop={() => handleDrop(tier.id)}
					>
						<div class="flex flex-wrap gap-2">
							{#each getItemsForTier(tier.id) as item (item.id)}
								<div
									draggable="true"
									ondragstart={() => handleDragStart(item)}
									class="px-4 py-2 text-lg bg-gray-700 rounded-lg cursor-move hover:bg-gray-600 transition-colors flex items-center gap-2"
								>
									<span>{item.name}</span>
									{#if editMode}
										<button
											onclick={() => removeItem(item.id)}
											class="text-red-400 hover:text-red-300"
										>
											✕
										</button>
									{/if}
								</div>
							{/each}
						</div>
					</div>
					{#if editMode}
						<button
							onclick={() => removeTier(tier.id)}
							class="w-12 bg-red-600 hover:bg-red-700 transition-colors flex items-center justify-center"
						>
							✕
						</button>
					{/if}
				</div>
			{/each}
		</div>

		<div class="mt-8 border border-gray-700 rounded-lg overflow-hidden">
			<div class="flex">
				<div class="w-24 bg-gray-700 flex items-center justify-center font-bold text-xl">
					Unranked
				</div>
				<div
					class="flex-1 bg-gray-800 p-4 min-h-[100px]"
					ondragover={handleDragOver}
					ondrop={() => handleDrop(null)}
				>
					<div class="flex flex-wrap gap-2">
						{#each getItemsForTier(null) as item (item.id)}
							<div
								draggable="true"
								ondragstart={() => handleDragStart(item)}
								class="px-4 py-2 text-lg bg-gray-700 rounded-lg cursor-move hover:bg-gray-600 transition-colors flex items-center gap-2"
							>
								<span>{item.name}</span>
								{#if editMode}
									<button
										onclick={() => removeItem(item.id)}
										class="text-red-400 hover:text-red-300"
									>
										✕
									</button>
								{/if}
							</div>
						{/each}
					</div>
				</div>
			</div>
		</div>

		{#if showImportModal}
			<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
				<div class="bg-gray-800 rounded-lg p-6 max-w-2xl w-full">
					<h2 class="text-2xl font-bold mb-4">Import from TOML</h2>
					<textarea
						bind:value={importText}
						placeholder="Paste your TOML content here..."
						class="w-full h-64 px-4 py-2 bg-gray-900 border border-gray-700 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 font-mono text-sm"
					></textarea>
					<div class="flex gap-4 mt-4 justify-end">
						<button
							onclick={() => {
								showImportModal = false;
								importText = '';
							}}
							class="px-6 py-2 bg-gray-600 hover:bg-gray-700 rounded-lg transition-colors"
						>
							Cancel
						</button>
						<button
							onclick={importFromToml}
							class="px-6 py-2 bg-purple-600 hover:bg-purple-700 rounded-lg transition-colors"
						>
							Import
						</button>
					</div>
				</div>
			</div>
		{/if}
	</div>
</div>
