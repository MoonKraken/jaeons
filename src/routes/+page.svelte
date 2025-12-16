<script lang="ts">
	import { parse as parseToml, stringify as stringifyToml } from 'smol-toml';

	interface Item {
		id: string;
		name: string;
		tierId: string | null;
		bucketId: string | null;
		bgColor: string;
		textGradientFrom: string;
		textGradientTo: string;
		position: number;
	}

	interface Tier {
		id: string;
		name: string;
	}

	interface UnrankedBucket {
		id: string;
		name: string;
		labelBgColor: string;
	}

	let editMode = $state(true);
	let draggedItem = $state<Item | null>(null);
	let originalPosition = $state<{ tierId: string | null; bucketId: string | null; position: number } | null>(null);
	let editingItemId = $state<string | null>(null);
	let newItemName = $state('');
	let newTierName = $state('');
	let newBucketName = $state('');
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

	let unrankedBuckets = $state<UnrankedBucket[]>([
		{
			id: 'unranked-1',
			name: 'Unranked',
			labelBgColor: '#4a5568'
		}
	]);

	let items = $state<Item[]>([
		{
			id: '1',
			name: 'Item 1',
			tierId: null,
			bucketId: 'unranked-1',
			bgColor: '#2d3748',
			textGradientFrom: '#a0aec0',
			textGradientTo: '#cbd5e0',
			position: 0
		},
		{
			id: '2',
			name: 'Item 2',
			tierId: null,
			bucketId: 'unranked-1',
			bgColor: '#2d3748',
			textGradientFrom: '#a0aec0',
			textGradientTo: '#cbd5e0',
			position: 1
		},
		{
			id: '3',
			name: 'Item 3',
			tierId: null,
			bucketId: 'unranked-1',
			bgColor: '#2d3748',
			textGradientFrom: '#a0aec0',
			textGradientTo: '#cbd5e0',
			position: 2
		},
		{
			id: '4',
			name: 'Item 4',
			tierId: null,
			bucketId: 'unranked-1',
			bgColor: '#2d3748',
			textGradientFrom: '#a0aec0',
			textGradientTo: '#cbd5e0',
			position: 3
		},
		{
			id: '5',
			name: 'Item 5',
			tierId: null,
			bucketId: 'unranked-1',
			bgColor: '#2d3748',
			textGradientFrom: '#a0aec0',
			textGradientTo: '#cbd5e0',
			position: 4
		}
	]);

	function handleDragStart(item: Item) {
		draggedItem = item;
		originalPosition = {
			tierId: item.tierId,
			bucketId: item.bucketId,
			position: item.position
		};
	}

	function handleDragOver(event: DragEvent) {
		event.preventDefault();
	}

	function handleDrop(tierId: string | null, bucketId: string | null = null) {
		if (draggedItem && originalPosition) {
			// Check if dropping back to original location
			const isOriginalLocation =
				originalPosition.tierId === tierId && originalPosition.bucketId === bucketId;

			if (isOriginalLocation) {
				// Restore original position
				draggedItem.tierId = tierId;
				draggedItem.bucketId = bucketId;
				draggedItem.position = originalPosition.position;
			} else {
				// Calculate the max position in the target tier/bucket
				const targetItems = items.filter((item) => {
					if (tierId !== null) {
						return item.tierId === tierId;
					} else if (bucketId !== null) {
						return item.bucketId === bucketId && item.tierId === null;
					}
					return false;
				});

				const maxPosition =
					targetItems.length > 0 ? Math.max(...targetItems.map((i) => i.position)) : -1;

				draggedItem.tierId = tierId;
				draggedItem.bucketId = bucketId;
				draggedItem.position = maxPosition + 1;
			}

			draggedItem = null;
			originalPosition = null;
		}
	}

	function addItem() {
		if (newItemName.trim()) {
			const bucketId = unrankedBuckets.length > 0 ? unrankedBuckets[0].id : null;
			const existingItems = items.filter(
				(item) => item.bucketId === bucketId && item.tierId === null
			);
			const maxPosition = existingItems.length > 0 ? Math.max(...existingItems.map((i) => i.position)) : -1;

			items.push({
				id: Date.now().toString(),
				name: newItemName.trim(),
				tierId: null,
				bucketId: bucketId,
				bgColor: '#2d3748',
				textGradientFrom: '#a0aec0',
				textGradientTo: '#cbd5e0',
				position: maxPosition + 1
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
				item.bucketId = unrankedBuckets.length > 0 ? unrankedBuckets[0].id : null;
			}
		});
		tiers = tiers.filter((tier) => tier.id !== id);
	}

	function addBucket() {
		if (newBucketName.trim()) {
			const newBucket: UnrankedBucket = {
				id: Date.now().toString(),
				name: newBucketName.trim(),
				labelBgColor: '#4a5568'
			};
			unrankedBuckets.push(newBucket);
			newBucketName = '';
		}
	}

	function removeBucket(id: string) {
		items.forEach((item) => {
			if (item.bucketId === id) {
				item.bucketId = unrankedBuckets.length > 1 ? unrankedBuckets[0].id : null;
			}
		});
		unrankedBuckets = unrankedBuckets.filter((bucket) => bucket.id !== id);
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
			title: title,
			tiers_config: tiers.map((tier) => ({
				id: tier.id,
				name: tier.name
			})),
			buckets_config: unrankedBuckets.map((bucket) => ({
				id: bucket.id,
				name: bucket.name,
				labelBgColor: bucket.labelBgColor
			})),
			items_config: items.map((item) => ({
				id: item.id,
				name: item.name,
				bgColor: item.bgColor,
				textGradientFrom: item.textGradientFrom,
				textGradientTo: item.textGradientTo,
				position: item.position
			}))
		};

		tiers.forEach((tier) => {
			const tierItems = getItemsForTier(tier.id);
			if (tierItems.length > 0) {
				data[tier.name] = {
					items: tierItems.map((item) => item.id)
				};
			}
		});

		unrankedBuckets.forEach((bucket) => {
			const bucketItems = getItemsForBucket(bucket.id);
			if (bucketItems.length > 0) {
				data[bucket.name] = {
					items: bucketItems.map((item) => item.id)
				};
			}
		});

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
			const newBuckets: UnrankedBucket[] = [];
			const newItems: Item[] = [];
			const itemsMap = new Map<string, Item>();

			// First, load all items from items_config
			if (Array.isArray(data.items_config)) {
				data.items_config.forEach((itemConfig: any) => {
					const item: Item = {
						id: itemConfig.id || Date.now().toString(),
						name: itemConfig.name,
						tierId: null,
						bucketId: null,
						bgColor: itemConfig.bgColor || '#2d3748',
						textGradientFrom: itemConfig.textGradientFrom || '#a0aec0',
						textGradientTo: itemConfig.textGradientTo || '#cbd5e0',
						position: itemConfig.position || 0
					};
					itemsMap.set(item.id, item);
				});
			}

			// Load tiers config
			if (Array.isArray(data.tiers_config)) {
				data.tiers_config.forEach((tierConfig: any) => {
					newTiers.push({
						id: tierConfig.id || tierConfig.name.toLowerCase().replace(/\s+/g, '-'),
						name: tierConfig.name
					});
				});
			}

			// Load buckets config
			if (Array.isArray(data.buckets_config)) {
				data.buckets_config.forEach((bucketConfig: any) => {
					newBuckets.push({
						id: bucketConfig.id || bucketConfig.name.toLowerCase().replace(/\s+/g, '-'),
						name: bucketConfig.name,
						labelBgColor: bucketConfig.labelBgColor || '#4a5568'
					});
				});
			}

			const tierNames = new Set(newTiers.map((t) => t.name));
			const bucketNames = new Set(newBuckets.map((b) => b.name));

			// Assign items to tiers/buckets
			Object.keys(data).forEach((key) => {
				if (
					key === 'title' ||
					key === 'tiers_config' ||
					key === 'buckets_config' ||
					key === 'items_config'
				)
					return;

				const section = data[key];
				if (!section || typeof section !== 'object') return;

				const sectionItems = section.items;
				if (!Array.isArray(sectionItems)) return;

				if (bucketNames.has(key)) {
					const bucket = newBuckets.find((b) => b.name === key);
					sectionItems.forEach((itemId) => {
						const item = itemsMap.get(itemId);
						if (item) {
							item.bucketId = bucket?.id || null;
							item.tierId = null;
						}
					});
				} else if (tierNames.has(key)) {
					const tier = newTiers.find((t) => t.name === key);
					sectionItems.forEach((itemId) => {
						const item = itemsMap.get(itemId);
						if (item) {
							item.tierId = tier?.id || null;
							item.bucketId = null;
						}
					});
				}
			});

			// Add all items to the array
			newItems.push(...itemsMap.values());

			if (newBuckets.length === 0) {
				newBuckets.push({
					id: 'unranked-1',
					name: 'Unranked',
					labelBgColor: '#4a5568'
				});
			}

			tiers = newTiers;
			unrankedBuckets = newBuckets;
			items = newItems;
			showImportModal = false;
			importText = '';
		} catch (error) {
			alert('Error parsing TOML: ' + (error instanceof Error ? error.message : String(error)));
		}
	}

	function getItemsForTier(tierId: string | null) {
		return items.filter((item) => item.tierId === tierId).sort((a, b) => a.position - b.position);
	}

	function getItemsForBucket(bucketId: string) {
		return items
			.filter((item) => item.bucketId === bucketId && item.tierId === null)
			.sort((a, b) => a.position - b.position);
	}

	const gradientPresets = [
		{ name: 'Ocean', from: '#667eea', to: '#764ba2' },
		{ name: 'Sunset', from: '#ff6b6b', to: '#feca57' },
		{ name: 'Forest', from: '#11998e', to: '#38ef7d' },
		{ name: 'Fire', from: '#ee0979', to: '#ff6a00' },
		{ name: 'Cool', from: '#4facfe', to: '#00f2fe' },
		{ name: 'Warm', from: '#fa709a', to: '#fee140' },
		{ name: 'Purple', from: '#a8edea', to: '#fed6e3' },
		{ name: 'Gold', from: '#f7971e', to: '#ffd200' }
	];

	function applyGradientPreset(item: Item, preset: { from: string; to: string }) {
		item.textGradientFrom = preset.from;
		item.textGradientTo = preset.to;
	}
</script>

<div class="min-h-screen bg-[#0a0a0a] text-white p-8">
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
			{#if editMode}
				<button
					onclick={() => (editMode = !editMode)}
					class="px-6 py-2 bg-blue-600 hover:bg-blue-700 rounded-lg transition-colors"
				>
					Present Mode
				</button>
			{:else}
				<button
					onclick={() => (editMode = !editMode)}
					class="p-2 opacity-20 hover:opacity-60 transition-opacity"
					title="Edit Mode"
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						width="24"
						height="24"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
						stroke-linecap="round"
						stroke-linejoin="round"
					>
						<path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z" />
						<path d="m15 5 4 4" />
					</svg>
				</button>
			{/if}
		</div>

		{#if editMode}
			<div class="mb-8 space-y-4">
				<div class="grid grid-cols-3 gap-4">
					<div>
						<h2 class="text-xl font-semibold mb-2">Add New Item</h2>
						<div class="flex gap-2">
							<input
								type="text"
								bind:value={newItemName}
								onkeydown={(e) => e.key === 'Enter' && addItem()}
								placeholder="Enter item name"
								class="flex-1 px-4 py-2 bg-[#1a1a1a] border border-gray-700 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
							/>
							<button
								onclick={addItem}
								class="px-6 py-2 bg-green-600 hover:bg-green-700 rounded-lg transition-colors"
							>
								Add Item
							</button>
						</div>
					</div>

					<div>
						<h2 class="text-xl font-semibold mb-2">Add New Tier</h2>
						<div class="flex gap-2">
							<input
								type="text"
								bind:value={newTierName}
								onkeydown={(e) => e.key === 'Enter' && addTier()}
								placeholder="Enter tier name"
								class="flex-1 px-4 py-2 bg-[#1a1a1a] border border-gray-700 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
							/>
							<button
								onclick={addTier}
								class="px-6 py-2 bg-green-600 hover:bg-green-700 rounded-lg transition-colors"
							>
								Add Tier
							</button>
						</div>
					</div>

					<div>
						<h2 class="text-xl font-semibold mb-2">Add New Bucket</h2>
						<div class="flex gap-2">
							<input
								type="text"
								bind:value={newBucketName}
								onkeydown={(e) => e.key === 'Enter' && addBucket()}
								placeholder="Enter bucket name"
								class="flex-1 px-4 py-2 bg-[#1a1a1a] border border-gray-700 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
							/>
							<button
								onclick={addBucket}
								class="px-6 py-2 bg-green-600 hover:bg-green-700 rounded-lg transition-colors"
							>
								Add Bucket
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
						class="flex-1 bg-[#1a1a1a] p-4 min-h-[100px] flex items-center"
						ondragover={handleDragOver}
						ondrop={() => handleDrop(tier.id)}
					>
						<div class="flex flex-wrap gap-2 items-center">
							{#each getItemsForTier(tier.id) as item (item.id)}
								<div class="flex flex-col gap-1">
									<div
										draggable="true"
										ondragstart={() => handleDragStart(item)}
										onclick={() => editMode && (editingItemId = editingItemId === item.id ? null : item.id)}
										class="px-4 py-2 text-xl font-bold rounded-lg cursor-move hover:opacity-90 transition-opacity flex items-center gap-2 {draggedItem?.id === item.id ? 'opacity-30' : ''}"
										style="background-color: {item.bgColor}"
									>
										<span
											class="bg-gradient-to-r bg-clip-text text-transparent"
											style="background-image: linear-gradient(to right, {item.textGradientFrom}, {item.textGradientTo})"
										>
											{item.name}
										</span>
										{#if editMode}
											<button
												onclick={(e) => {
													e.stopPropagation();
													removeItem(item.id);
												}}
												class="text-red-400 hover:text-red-300"
											>
												✕
											</button>
										{/if}
									</div>
									{#if editMode && editingItemId === item.id}
										<div
											class="bg-[#0a0a0a] border border-gray-600 rounded p-2 flex flex-col gap-2 text-xs"
										>
											<div class="flex gap-2 items-center">
												<div class="flex items-center gap-1">
													<label class="text-gray-400">BG:</label>
													<input
														type="color"
														bind:value={item.bgColor}
														class="w-8 h-6 rounded cursor-pointer"
														onclick={(e) => e.stopPropagation()}
													/>
												</div>
												<div class="flex items-center gap-1">
													<label class="text-gray-400">Text:</label>
													<input
														type="color"
														bind:value={item.textGradientFrom}
														class="w-8 h-6 rounded cursor-pointer"
														onclick={(e) => e.stopPropagation()}
													/>
													<span class="text-gray-500">→</span>
													<input
														type="color"
														bind:value={item.textGradientTo}
														class="w-8 h-6 rounded cursor-pointer"
														onclick={(e) => e.stopPropagation()}
													/>
												</div>
											</div>
											<div class="flex gap-1 flex-wrap">
												{#each gradientPresets as preset}
													<button
														onclick={(e) => {
															e.stopPropagation();
															applyGradientPreset(item, preset);
														}}
														class="w-12 h-6 rounded cursor-pointer hover:scale-110 transition-transform border border-gray-500"
														style="background: linear-gradient(to right, {preset.from}, {preset.to})"
														title={preset.name}
													></button>
												{/each}
											</div>
										</div>
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

		<div class="mt-8 space-y-2">
			{#each unrankedBuckets as bucket (bucket.id)}
				<div class="border border-gray-700 rounded-lg overflow-hidden">
					<div class="flex">
						<div
							class="w-24 flex items-center justify-center font-bold text-xl"
							style="background-color: {bucket.labelBgColor}"
						>
							{bucket.name}
						</div>
						<div
							class="flex-1 bg-[#1a1a1a] p-4 min-h-[100px] flex items-center"
							ondragover={handleDragOver}
							ondrop={() => handleDrop(null, bucket.id)}
						>
							<div class="flex flex-wrap gap-2 items-center">
								{#each getItemsForBucket(bucket.id) as item (item.id)}
									<div class="flex flex-col gap-1">
										<div
											draggable="true"
											ondragstart={() => handleDragStart(item)}
											onclick={() => editMode && (editingItemId = editingItemId === item.id ? null : item.id)}
											class="px-4 py-2 text-xl font-bold rounded-lg cursor-move hover:opacity-90 transition-opacity flex items-center gap-2 {draggedItem?.id === item.id ? 'opacity-30' : ''}"
											style="background-color: {item.bgColor}"
										>
											<span
												class="bg-gradient-to-r bg-clip-text text-transparent"
												style="background-image: linear-gradient(to right, {item.textGradientFrom}, {item.textGradientTo})"
											>
												{item.name}
											</span>
											{#if editMode}
												<button
													onclick={(e) => {
														e.stopPropagation();
														removeItem(item.id);
													}}
													class="text-red-400 hover:text-red-300"
												>
													✕
												</button>
											{/if}
										</div>
										{#if editMode && editingItemId === item.id}
											<div
												class="bg-[#0a0a0a] border border-gray-600 rounded p-2 flex flex-col gap-2 text-xs"
											>
												<div class="flex gap-2 items-center">
													<div class="flex items-center gap-1">
														<label class="text-gray-400">BG:</label>
														<input
															type="color"
															bind:value={item.bgColor}
															class="w-8 h-6 rounded cursor-pointer"
															onclick={(e) => e.stopPropagation()}
														/>
													</div>
													<div class="flex items-center gap-1">
														<label class="text-gray-400">Text:</label>
														<input
															type="color"
															bind:value={item.textGradientFrom}
															class="w-8 h-6 rounded cursor-pointer"
															onclick={(e) => e.stopPropagation()}
														/>
														<span class="text-gray-500">→</span>
														<input
															type="color"
															bind:value={item.textGradientTo}
															class="w-8 h-6 rounded cursor-pointer"
															onclick={(e) => e.stopPropagation()}
														/>
													</div>
												</div>
												<div class="flex gap-1 flex-wrap">
													{#each gradientPresets as preset}
														<button
															onclick={(e) => {
																e.stopPropagation();
																applyGradientPreset(item, preset);
															}}
															class="w-12 h-6 rounded cursor-pointer hover:scale-110 transition-transform border border-gray-500"
															style="background: linear-gradient(to right, {preset.from}, {preset.to})"
															title={preset.name}
														></button>
													{/each}
												</div>
											</div>
										{/if}
									</div>
								{/each}
							</div>
						</div>
						{#if editMode}
							<button
								onclick={() => removeBucket(bucket.id)}
								class="w-12 bg-red-600 hover:bg-red-700 transition-colors flex items-center justify-center"
							>
								✕
							</button>
						{/if}
					</div>
					{#if editMode}
						<div class="bg-[#1a1a1a] border-t border-gray-700 p-3">
							<div class="flex gap-4 items-center text-sm">
								<div class="flex items-center gap-2">
									<label class="text-gray-400">Label BG:</label>
									<input
										type="color"
										bind:value={bucket.labelBgColor}
										class="w-12 h-8 rounded cursor-pointer"
									/>
								</div>
							</div>
						</div>
					{/if}
				</div>
			{/each}
		</div>

		{#if showImportModal}
			<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
				<div class="bg-[#1a1a1a] rounded-lg p-6 max-w-2xl w-full">
					<h2 class="text-2xl font-bold mb-4">Import from TOML</h2>
					<textarea
						bind:value={importText}
						placeholder="Paste your TOML content here..."
						class="w-full h-64 px-4 py-2 bg-[#0a0a0a] border border-gray-700 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 font-mono text-sm"
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
