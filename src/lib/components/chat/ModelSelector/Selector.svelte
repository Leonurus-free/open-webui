<script lang="ts">
	import { DropdownMenu } from 'bits-ui';
	import { marked } from 'marked';
	import Fuse from 'fuse.js';

	import dayjs from '$lib/dayjs';
	import relativeTime from 'dayjs/plugin/relativeTime';
	dayjs.extend(relativeTime);

	import Spinner from '$lib/components/common/Spinner.svelte';
	import { flyAndScale } from '$lib/utils/transitions';
	import { createEventDispatcher, onMount, getContext, tick } from 'svelte';
	import { goto } from '$app/navigation';

	import { deleteModel, getOllamaVersion, pullModel, unloadModel } from '$lib/apis/ollama';

	import {
		user,
		MODEL_DOWNLOAD_POOL,
		models,
		mobile,
		temporaryChatEnabled,
		settings,
		config
	} from '$lib/stores';
	import { toast } from 'svelte-sonner';
	import { capitalizeFirstLetter, sanitizeResponseContent, splitStream } from '$lib/utils';
	import { getModels } from '$lib/apis';

	import ChevronDown from '$lib/components/icons/ChevronDown.svelte';
	import Check from '$lib/components/icons/Check.svelte';
	import Search from '$lib/components/icons/Search.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Switch from '$lib/components/common/Switch.svelte';
	import ChatBubbleOval from '$lib/components/icons/ChatBubbleOval.svelte';

	import ModelItem from './ModelItem.svelte';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	export let id = '';
	export let value = '';
	export let placeholder = $i18n.t('Select a model');
	export let searchEnabled = true;
	export let searchPlaceholder = $i18n.t('Search a model');

	export let items: {
		label: string;
		value: string;
		model: Model;
		// eslint-disable-next-line @typescript-eslint/no-explicit-any
		[key: string]: any;
	}[] = [];

	export let className = 'w-[32rem]';
	export let triggerClassName = 'text-lg';

	export let pinModelHandler: (modelId: string) => void = () => {};

	let tagsContainerElement;

	let show = false;
	let tags = [];

	let selectedModel = '';
	$: selectedModel = items.find((item) => item.value === value) ?? '';

	// ========== 自定义功能：模型使用量状态显示 START ==========
	// 添加日期: 2026-01-15
	// 功能说明: 根据模型的 token_usage 字段显示模型状态（空闲/繁忙）
	//
	// 使用说明:
	// 1. 后端 API (/api/models) 返回的模型数据中可能包含 token_usage 字段
	// 2. token_usage 是一个 0-1 之间的小数，表示模型的使用量百分比
	// 3. 根据使用量显示状态：
	//    - token_usage < 0.6 (60%)：显示"空闲"标签（绿色）
	//    - token_usage >= 0.6 (60%)：显示"繁忙"标签（红色）
	// 4. 如果模型没有 token_usage 字段，则不显示状态标签
	//
	// 测试模式说明:
	// - ENABLE_TEST_MODE = true：为没有 token_usage 字段的模型添加测试值 0.45
	// - ENABLE_TEST_MODE = false：只显示真实的 token_usage 数据
	// - 当后端真正返回 token_usage 字段后，将 ENABLE_TEST_MODE 改为 false
	//
	const ENABLE_TEST_MODE = true; // 设置为 false 可关闭测试模式
	$: selectedModelTokenUsage = (() => {
		if (!selectedModel) return undefined;
		const realValue = selectedModel.model?.token_usage;
		// 如果有真实值，直接使用
		if (realValue !== undefined && realValue !== null) {
			return realValue;
		}
		// 测试模式：只为测试添加假数据，生产环境下如果没有字段则返回 undefined
		return ENABLE_TEST_MODE ? 0.45 : undefined;
	})();
	// ========== 自定义功能：模型使用量状态显示 END ==========

	let searchValue = '';

	let selectedTag = '';
	let selectedConnectionType = '';

	let ollamaVersion = null;
	let selectedModelIdx = 0;

	const fuse = new Fuse(
		items.map((item) => {
			const _item = {
				...item,
				modelName: item.model?.name,
				tags: (item.model?.tags ?? []).map((tag) => tag.name).join(' '),
				desc: item.model?.info?.meta?.description
			};
			return _item;
		}),
		{
			keys: ['value', 'tags', 'modelName'],
			threshold: 0.4
		}
	);

	const updateFuse = () => {
		if (fuse) {
			fuse.setCollection(
				items.map((item) => {
					const _item = {
						...item,
						modelName: item.model?.name,
						tags: (item.model?.tags ?? []).map((tag) => tag.name).join(' '),
						desc: item.model?.info?.meta?.description
					};
					return _item;
				})
			);
		}
	};

	$: if (items) {
		updateFuse();
	}

	$: filteredItems = (
		searchValue
			? fuse
					.search(searchValue)
					.map((e) => {
						return e.item;
					})
					.filter((item) => {
						if (selectedTag === '') {
							return true;
						}

						return (item.model?.tags ?? [])
							.map((tag) => tag.name.toLowerCase())
							.includes(selectedTag.toLowerCase());
					})
					.filter((item) => {
						if (selectedConnectionType === '') {
							return true;
						} else if (selectedConnectionType === 'local') {
							return item.model?.connection_type === 'local';
						} else if (selectedConnectionType === 'external') {
							return item.model?.connection_type === 'external';
						} else if (selectedConnectionType === 'direct') {
							return item.model?.direct;
						}
					})
			: items
					.filter((item) => {
						if (selectedTag === '') {
							return true;
						}
						return (item.model?.tags ?? [])
							.map((tag) => tag.name.toLowerCase())
							.includes(selectedTag.toLowerCase());
					})
					.filter((item) => {
						if (selectedConnectionType === '') {
							return true;
						} else if (selectedConnectionType === 'local') {
							return item.model?.connection_type === 'local';
						} else if (selectedConnectionType === 'external') {
							return item.model?.connection_type === 'external';
						} else if (selectedConnectionType === 'direct') {
							return item.model?.direct;
						}
					})
	).filter((item) => !(item.model?.info?.meta?.hidden ?? false));

	$: if (selectedTag || selectedConnectionType) {
		resetView();
	} else {
		resetView();
	}

	const resetView = async () => {
		await tick();

		const selectedInFiltered = filteredItems.findIndex((item) => item.value === value);

		if (selectedInFiltered >= 0) {
			// The selected model is visible in the current filter
			selectedModelIdx = selectedInFiltered;
		} else {
			// The selected model is not visible, default to first item in filtered list
			selectedModelIdx = 0;
		}

		await tick();
		const item = document.querySelector(`[data-arrow-selected="true"]`);
		item?.scrollIntoView({ block: 'center', inline: 'nearest', behavior: 'instant' });
	};

	const pullModelHandler = async () => {
		const sanitizedModelTag = searchValue.trim().replace(/^ollama\s+(run|pull)\s+/, '');

		console.log($MODEL_DOWNLOAD_POOL);
		if ($MODEL_DOWNLOAD_POOL[sanitizedModelTag]) {
			toast.error(
				$i18n.t(`Model '{{modelTag}}' is already in queue for downloading.`, {
					modelTag: sanitizedModelTag
				})
			);
			return;
		}
		if (Object.keys($MODEL_DOWNLOAD_POOL).length === 3) {
			toast.error(
				$i18n.t('Maximum of 3 models can be downloaded simultaneously. Please try again later.')
			);
			return;
		}

		const [res, controller] = await pullModel(localStorage.token, sanitizedModelTag, '0').catch(
			(error) => {
				toast.error(`${error}`);
				return null;
			}
		);

		if (res) {
			const reader = res.body
				.pipeThrough(new TextDecoderStream())
				.pipeThrough(splitStream('\n'))
				.getReader();

			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL,
				[sanitizedModelTag]: {
					...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
					abortController: controller,
					reader,
					done: false
				}
			});

			while (true) {
				try {
					const { value, done } = await reader.read();
					if (done) break;

					let lines = value.split('\n');

					for (const line of lines) {
						if (line !== '') {
							let data = JSON.parse(line);
							console.log(data);
							if (data.error) {
								throw data.error;
							}
							if (data.detail) {
								throw data.detail;
							}

							if (data.status) {
								if (data.digest) {
									let downloadProgress = 0;
									if (data.completed) {
										downloadProgress = Math.round((data.completed / data.total) * 1000) / 10;
									} else {
										downloadProgress = 100;
									}

									MODEL_DOWNLOAD_POOL.set({
										...$MODEL_DOWNLOAD_POOL,
										[sanitizedModelTag]: {
											...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
											pullProgress: downloadProgress,
											digest: data.digest
										}
									});
								} else {
									toast.success(data.status);

									MODEL_DOWNLOAD_POOL.set({
										...$MODEL_DOWNLOAD_POOL,
										[sanitizedModelTag]: {
											...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
											done: data.status === 'success'
										}
									});
								}
							}
						}
					}
				} catch (error) {
					console.log(error);
					if (typeof error !== 'string') {
						error = error.message;
					}

					toast.error(`${error}`);
					// opts.callback({ success: false, error, modelName: opts.modelName });
					break;
				}
			}

			if ($MODEL_DOWNLOAD_POOL[sanitizedModelTag].done) {
				toast.success(
					$i18n.t(`Model '{{modelName}}' has been successfully downloaded.`, {
						modelName: sanitizedModelTag
					})
				);

				models.set(
					await getModels(
						localStorage.token,
						$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
					)
				);
			} else {
				toast.error($i18n.t('Download canceled'));
			}

			delete $MODEL_DOWNLOAD_POOL[sanitizedModelTag];

			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL
			});
		}
	};

	const setOllamaVersion = async () => {
		ollamaVersion = await getOllamaVersion(localStorage.token).catch((error) => false);
	};

	onMount(async () => {
		if (items) {
			tags = items
				.filter((item) => !(item.model?.info?.meta?.hidden ?? false))
				.flatMap((item) => item.model?.tags ?? [])
				.map((tag) => tag.name.toLowerCase());
			// Remove duplicates and sort
			tags = Array.from(new Set(tags)).sort((a, b) => a.localeCompare(b));
		}
	});

	$: if (show) {
		setOllamaVersion();
	}

	const cancelModelPullHandler = async (model: string) => {
		const { reader, abortController } = $MODEL_DOWNLOAD_POOL[model];
		if (abortController) {
			abortController.abort();
		}
		if (reader) {
			await reader.cancel();
			delete $MODEL_DOWNLOAD_POOL[model];
			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL
			});
			await deleteModel(localStorage.token, model);
			toast.success($i18n.t('{{model}} download has been canceled', { model: model }));
		}
	};

	const unloadModelHandler = async (model: string) => {
		const res = await unloadModel(localStorage.token, model).catch((error) => {
			toast.error($i18n.t('Error unloading model: {{error}}', { error }));
		});

		if (res) {
			toast.success($i18n.t('Model unloaded successfully'));
			models.set(
				await getModels(
					localStorage.token,
					$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
				)
			);
		}
	};
</script>

<DropdownMenu.Root
	bind:open={show}
	onOpenChange={async () => {
		searchValue = '';
		window.setTimeout(() => document.getElementById('model-search-input')?.focus(), 0);

		resetView();
	}}
	closeFocus={false}
>
	<DropdownMenu.Trigger
		class="relative w-full {($settings?.highContrastMode ?? false)
			? ''
			: 'outline-hidden focus:outline-hidden'}"
		aria-label={placeholder}
		id="model-selector-{id}-button"
	>
		<div
			class="flex items-center w-full text-left px-0.5 bg-transparent truncate {triggerClassName} justify-between {($settings?.highContrastMode ??
			false)
				? 'dark:placeholder-gray-100 placeholder-gray-800'
				: 'placeholder-gray-400'}"
			on:mouseenter={async () => {
				models.set(
					await getModels(
						localStorage.token,
						$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
					)
				);
			}}
		>
			{#if selectedModel}
				<span class="truncate">{selectedModel.label}</span>
				<!-- ========== 自定义功能：显示模型状态标签 START ========== -->
				<!-- 功能说明: 在选中的模型名称后面显示状态标签（空闲/繁忙） -->
				<!-- 显示条件: 只有当 selectedModelTokenUsage 有值时才显示 -->
				<!-- 状态判断:
					- token_usage < 0.6 (60%)：显示"空闲"，绿色背景和文字
					- token_usage >= 0.6 (60%)：显示"繁忙"，红色背景和文字
				-->
				<!-- 兼容性说明:
					- 使用内联样式 + rgb() 颜色值以支持老版本浏览器（如 Chrome 106）
					- 避免使用 Tailwind 的透明度语法（/30）以提高兼容性
				-->
				{#if selectedModelTokenUsage !== undefined && selectedModelTokenUsage !== null}
					{#if selectedModelTokenUsage < 0.6}
						<!-- 空闲状态：绿色标签 -->
						<span
							class="ml-2 px-1.5 py-0.5 text-xs font-medium rounded"
							style="background-color: rgb(220, 252, 231); color: rgb(21, 128, 61);"
						>
							空闲
						</span>
					{:else}
						<!-- 繁忙状态：红色标签 -->
						<span
							class="ml-2 px-1.5 py-0.5 text-xs font-medium rounded"
							style="background-color: rgb(254, 226, 226); color: rgb(185, 28, 28);"
						>
							繁忙
						</span>
					{/if}
				{/if}
				<!-- ========== 自定义功能：显示模型状态标签 END ========== -->
			{:else}
				{placeholder}
			{/if}
			<ChevronDown className=" self-center ml-2 size-3" strokeWidth="2.5" />
		</div>
	</DropdownMenu.Trigger>

	<DropdownMenu.Content
		class=" z-40 {$mobile
			? `w-full`
			: `${className}`} max-w-[calc(100vw-1rem)] justify-start rounded-2xl  bg-white dark:bg-gray-850 dark:text-white shadow-lg  outline-hidden"
		transition={flyAndScale}
		side={$mobile ? 'bottom' : 'bottom-start'}
		sideOffset={2}
		alignOffset={-1}
	>
		<slot>
			{#if searchEnabled}
				<div class="flex items-center gap-2.5 px-4.5 mt-3.5 mb-1.5">
					<Search className="size-4" strokeWidth="2.5" />

					<input
						id="model-search-input"
						bind:value={searchValue}
						class="w-full text-sm bg-transparent outline-hidden"
						placeholder={searchPlaceholder}
						autocomplete="off"
						aria-label={$i18n.t('Search In Models')}
						on:keydown={(e) => {
							if (e.code === 'Enter' && filteredItems.length > 0) {
								value = filteredItems[selectedModelIdx].value;
								show = false;
								return; // dont need to scroll on selection
							} else if (e.code === 'ArrowDown') {
								e.stopPropagation();
								selectedModelIdx = Math.min(selectedModelIdx + 1, filteredItems.length - 1);
							} else if (e.code === 'ArrowUp') {
								e.stopPropagation();
								selectedModelIdx = Math.max(selectedModelIdx - 1, 0);
							} else {
								// if the user types something, reset to the top selection.
								selectedModelIdx = 0;
							}

							const item = document.querySelector(`[data-arrow-selected="true"]`);
							item?.scrollIntoView({ block: 'center', inline: 'nearest', behavior: 'instant' });
						}}
					/>
				</div>
			{/if}

			<div class="px-2">
				{#if tags && items.filter((item) => !(item.model?.info?.meta?.hidden ?? false)).length > 0}
					<div
						class=" flex w-full bg-white dark:bg-gray-850 overflow-x-auto scrollbar-none font-[450] mb-0.5"
						on:wheel={(e) => {
							if (e.deltaY !== 0) {
								e.preventDefault();
								e.currentTarget.scrollLeft += e.deltaY;
							}
						}}
					>
						<div
							class="flex gap-1 w-fit text-center text-sm rounded-full bg-transparent px-1.5 whitespace-nowrap"
							bind:this={tagsContainerElement}
						>
							{#if items.find((item) => item.model?.connection_type === 'local') || items.find((item) => item.model?.connection_type === 'external') || items.find((item) => item.model?.direct) || tags.length > 0}
								<button
									class="min-w-fit outline-none px-1.5 py-0.5 {selectedTag === '' &&
									selectedConnectionType === ''
										? ''
										: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition capitalize"
									aria-pressed={selectedTag === '' && selectedConnectionType === ''}
									on:click={() => {
										selectedConnectionType = '';
										selectedTag = '';
									}}
								>
									{$i18n.t('All')}
								</button>
							{/if}

							{#if items.find((item) => item.model?.connection_type === 'local')}
								<button
									class="min-w-fit outline-none px-1.5 py-0.5 {selectedConnectionType === 'local'
										? ''
										: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition capitalize"
									aria-pressed={selectedConnectionType === 'local'}
									on:click={() => {
										selectedTag = '';
										selectedConnectionType = 'local';
									}}
								>
									{$i18n.t('Local')}
								</button>
							{/if}

							{#if items.find((item) => item.model?.connection_type === 'external')}
								<button
									class="min-w-fit outline-none px-1.5 py-0.5 {selectedConnectionType === 'external'
										? ''
										: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition capitalize"
									aria-pressed={selectedConnectionType === 'external'}
									on:click={() => {
										selectedTag = '';
										selectedConnectionType = 'external';
									}}
								>
									{$i18n.t('External')}
								</button>
							{/if}

							{#if items.find((item) => item.model?.direct)}
								<button
									class="min-w-fit outline-none px-1.5 py-0.5 {selectedConnectionType === 'direct'
										? ''
										: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition capitalize"
									aria-pressed={selectedConnectionType === 'direct'}
									on:click={() => {
										selectedTag = '';
										selectedConnectionType = 'direct';
									}}
								>
									{$i18n.t('Direct')}
								</button>
							{/if}

							{#each tags as tag}
								<Tooltip content={tag}>
									<button
										class="min-w-fit outline-none px-1.5 py-0.5 {selectedTag === tag
											? ''
											: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition capitalize"
										aria-pressed={selectedTag === tag}
										on:click={() => {
											selectedConnectionType = '';
											selectedTag = tag;
										}}
									>
										{tag.length > 16 ? `${tag.slice(0, 16)}...` : tag}
									</button>
								</Tooltip>
							{/each}
						</div>
					</div>
				{/if}
			</div>

			<div class="px-2.5 max-h-64 overflow-y-auto group relative">
				{#each filteredItems as item, index}
					<ModelItem
						{selectedModelIdx}
						{item}
						{index}
						{value}
						{pinModelHandler}
						{unloadModelHandler}
						onClick={() => {
							value = item.value;
							selectedModelIdx = index;

							show = false;
						}}
					/>
				{:else}
					<div class="">
						<div class="block px-3 py-2 text-sm text-gray-700 dark:text-gray-100">
							{$i18n.t('No results found')}
						</div>
					</div>
				{/each}

				{#if !(searchValue.trim() in $MODEL_DOWNLOAD_POOL) && searchValue && ollamaVersion && $user?.role === 'admin'}
					<Tooltip
						content={$i18n.t(`Pull "{{searchValue}}" from Ollama.com`, {
							searchValue: searchValue
						})}
						placement="top-start"
					>
						<button
							class="flex w-full font-medium line-clamp-1 select-none items-center rounded-button py-2 pl-3 pr-1.5 text-sm text-gray-700 dark:text-gray-100 outline-hidden transition-all duration-75 hover:bg-gray-100 dark:hover:bg-gray-800 rounded-xl cursor-pointer data-highlighted:bg-muted"
							on:click={() => {
								pullModelHandler();
							}}
						>
							<div class=" truncate">
								{$i18n.t(`Pull "{{searchValue}}" from Ollama.com`, { searchValue: searchValue })}
							</div>
						</button>
					</Tooltip>
				{/if}

				{#each Object.keys($MODEL_DOWNLOAD_POOL) as model}
					<div
						class="flex w-full justify-between font-medium select-none rounded-button py-2 pl-3 pr-1.5 text-sm text-gray-700 dark:text-gray-100 outline-hidden transition-all duration-75 rounded-xl cursor-pointer data-highlighted:bg-muted"
					>
						<div class="flex">
							<div class="mr-2.5 translate-y-0.5">
								<Spinner />
							</div>

							<div class="flex flex-col self-start">
								<div class="flex gap-1">
									<div class="line-clamp-1">
										Downloading "{model}"
									</div>

									<div class="shrink-0">
										{'pullProgress' in $MODEL_DOWNLOAD_POOL[model]
											? `(${$MODEL_DOWNLOAD_POOL[model].pullProgress}%)`
											: ''}
									</div>
								</div>

								{#if 'digest' in $MODEL_DOWNLOAD_POOL[model] && $MODEL_DOWNLOAD_POOL[model].digest}
									<div class="-mt-1 h-fit text-[0.7rem] dark:text-gray-500 line-clamp-1">
										{$MODEL_DOWNLOAD_POOL[model].digest}
									</div>
								{/if}
							</div>
						</div>

						<div class="mr-2 ml-1 translate-y-0.5">
							<Tooltip content={$i18n.t('Cancel')}>
								<button
									class="text-gray-800 dark:text-gray-100"
									on:click={() => {
										cancelModelPullHandler(model);
									}}
								>
									<svg
										class="w-4 h-4 text-gray-800 dark:text-white"
										aria-hidden="true"
										xmlns="http://www.w3.org/2000/svg"
										width="24"
										height="24"
										fill="currentColor"
										viewBox="0 0 24 24"
									>
										<path
											stroke="currentColor"
											stroke-linecap="round"
											stroke-linejoin="round"
											stroke-width="2"
											d="M6 18 17.94 6M18 18 6.06 6"
										/>
									</svg>
								</button>
							</Tooltip>
						</div>
					</div>
				{/each}
			</div>

			<div class="mb-2.5"></div>

			<div class="hidden w-[42rem]" />
			<div class="hidden w-[32rem]" />
		</slot>
	</DropdownMenu.Content>
</DropdownMenu.Root>
