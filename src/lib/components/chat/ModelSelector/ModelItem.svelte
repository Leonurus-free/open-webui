<script lang="ts">
	import { marked } from 'marked';

	import { getContext, tick } from 'svelte';
	import dayjs from '$lib/dayjs';

	import { mobile, settings, user } from '$lib/stores';
	import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';

	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import { copyToClipboard, sanitizeResponseContent } from '$lib/utils';
	import ArrowUpTray from '$lib/components/icons/ArrowUpTray.svelte';
	import Check from '$lib/components/icons/Check.svelte';
	import ModelItemMenu from './ModelItemMenu.svelte';
	import EllipsisHorizontal from '$lib/components/icons/EllipsisHorizontal.svelte';
	import { toast } from 'svelte-sonner';
	import Tag from '$lib/components/icons/Tag.svelte';
	import Label from '$lib/components/icons/Label.svelte';

	const i18n = getContext('i18n');

	export let selectedModelIdx: number = -1;
	export let item: any = {};
	export let index: number = -1;
	export let value: string = '';

	export let unloadModelHandler: (modelValue: string) => void = () => {};
	export let pinModelHandler: (modelId: string) => void = () => {};

	export let onClick: () => void = () => {};

	const copyLinkHandler = async (model) => {
		const baseUrl = window.location.origin;
		const res = await copyToClipboard(`${baseUrl}/?model=${encodeURIComponent(model.id)}`);

		if (res) {
			toast.success($i18n.t('Copied link to clipboard'));
		} else {
			toast.error($i18n.t('Failed to copy link'));
		}
	};

	let showMenu = false;

	// ========== 自定义功能：模型使用量进度条显示 START ==========
	// 添加日期: 2026-01-15
	// 功能说明: 在模型下拉列表中，为每个模型条目显示使用量进度条背景
	//
	// 使用说明:
	// 1. 后端 API (/api/models) 返回的模型数据中可能包含 token_usage 字段
	// 2. token_usage 是一个 0-1 之间的小数，表示模型的使用量百分比
	// 3. 进度条效果：整个模型条目会有一个从左到右的渐变背景填充
	// 4. 填充宽度根据 token_usage 值动态变化（0-100%）
	// 5. 如果模型没有 token_usage 字段，则不显示进度条背景
	//
	// 测试模式说明:
	// - ENABLE_TEST_MODE = true：为所有模型生成测试进度条（根据索引计算）
	// - ENABLE_TEST_MODE = false：只显示真实的 token_usage 数据
	// - 当后端真正返回 token_usage 字段后，将 ENABLE_TEST_MODE 改为 false
	// - 测试值计算公式: (index * 0.13) % 0.9，会生成 0-90% 之间的不同值
	//
	const ENABLE_TEST_MODE = true; // 设置为 false 可关闭测试模式
	$: displayTokenUsage = ENABLE_TEST_MODE
		? (item.model.token_usage ?? (index * 0.13) % 0.9) // 根据索引生成不同的测试值
		: item.model.token_usage;
	// ========== 自定义功能：模型使用量进度条显示 END ==========
</script>

<button
	aria-roledescription="model-item"
	aria-label={item.label}
	class="flex group/item w-full text-left font-medium line-clamp-1 select-none items-center rounded-button py-2 pl-3 pr-1.5 text-sm text-gray-700 dark:text-gray-100 outline-hidden transition-all duration-75 hover:bg-gray-100 dark:hover:bg-gray-800 rounded-xl cursor-pointer data-highlighted:bg-muted mb-1 {index ===
	selectedModelIdx
		? 'bg-gray-100 dark:bg-gray-800 group-hover:bg-transparent'
		: ''} relative overflow-hidden"
	data-arrow-selected={index === selectedModelIdx}
	data-value={item.value}
	on:click={() => {
		onClick();
	}}
>
	<!-- ========== 自定义功能：背景进度条效果 START ========== -->
	<!-- 功能说明: 根据 token_usage 显示整个条目的背景填充效果 -->
	<!-- 实现方式:
		1. 使用绝对定位的 div 作为背景层
		2. 背景层宽度根据 displayTokenUsage 动态设置（0-100%）
		3. 使用半透明的蓝色到青色渐变（亮色模式 10% 透明度，暗色模式 20% 透明度）
		4. 添加过渡动画效果（duration-300）
		5. 只有当 displayTokenUsage 有值时才显示
	-->
	<!-- 样式说明:
		- button 添加了 relative：为绝对定位的背景层提供定位上下文
		- button 添加了 overflow-hidden：确保背景层不会溢出圆角边界
		- button 添加了 mb-1：在条目之间添加间距（0.25rem）
	-->
	<!-- 兼容性说明:
		- 使用内联样式 + rgba() 颜色值以支持老版本浏览器（如 Chrome 106）
		- 避免使用 Tailwind 的透明度语法（/10, /20）以提高兼容性
	-->
	{#if displayTokenUsage !== undefined && displayTokenUsage !== null}
		<div
			class="absolute inset-0 transition-all duration-300"
			style="width: {(displayTokenUsage * 100).toFixed(1)}%; background: linear-gradient(to right, rgba(59, 130, 246, 0.1), rgba(34, 211, 238, 0.1));"
		></div>
	{/if}
	<!-- ========== 自定义功能：背景进度条效果 END ========== -->

	<div class="flex flex-col flex-1 gap-1.5 relative z-10">
		<!-- {#if (item?.model?.tags ?? []).length > 0}
			<div
				class="flex gap-0.5 self-center items-start h-full w-full translate-y-[0.5px] overflow-x-auto scrollbar-none"
			>
				{#each item.model?.tags.sort((a, b) => a.name.localeCompare(b.name)) as tag}
					<Tooltip content={tag.name} className="flex-shrink-0">
						<div
							class=" text-xs font-semibold px-1 rounded-sm uppercase bg-gray-500/20 text-gray-700 dark:text-gray-200"
						>
							{tag.name}
						</div>
					</Tooltip>
				{/each}
			</div>
		{/if} -->

		<div class="flex items-center gap-2">
			<div class="flex items-center min-w-fit">
				<Tooltip content={$user?.role === 'admin' ? (item?.value ?? '') : ''} placement="top-start">
					<img
						src={`${WEBUI_API_BASE_URL}/models/model/profile/image?id=${item.model.id}&lang=${$i18n.language}`}
						alt="Model"
						class="rounded-full size-5 flex items-center"
						loading="lazy"
					/>
				</Tooltip>
			</div>

			<div class="flex items-center">
				<Tooltip content={`${item.label} (${item.value})`} placement="top-start">
					<div class="line-clamp-1">
						{item.label}
					</div>
				</Tooltip>
			</div>

			<div class=" shrink-0 flex items-center gap-2">
				{#if item.model.owned_by === 'ollama'}
					{#if (item.model.ollama?.details?.parameter_size ?? '') !== ''}
						<div class="flex items-center translate-y-[0.5px]">
							<Tooltip
								content={`${
									item.model.ollama?.details?.quantization_level
										? item.model.ollama?.details?.quantization_level + ' '
										: ''
								}${
									item.model.ollama?.size
										? `(${(item.model.ollama?.size / 1024 ** 3).toFixed(1)}GB)`
										: ''
								}`}
								className="self-end"
							>
								<span class=" text-xs font-medium text-gray-600 dark:text-gray-400 line-clamp-1"
									>{item.model.ollama?.details?.parameter_size ?? ''}</span
								>
							</Tooltip>
						</div>
					{/if}
				{/if}

				{#if item.model.owned_by === 'ollama'}
					{#if item.model.ollama?.expires_at && new Date(item.model.ollama?.expires_at * 1000) > new Date()}
						<div class="flex items-center translate-y-[0.5px] px-0.5">
							<Tooltip
								content={`${$i18n.t('Unloads {{FROM_NOW}}', {
									FROM_NOW: dayjs(item.model.ollama?.expires_at * 1000).fromNow()
								})}`}
								className="self-end"
							>
								<div class=" flex items-center">
									<span class="relative flex size-2">
										<span
											class="animate-ping absolute inline-flex h-full w-full rounded-full bg-green-400 opacity-75"
										/>
										<span class="relative inline-flex rounded-full size-2 bg-green-500" />
									</span>
								</div>
							</Tooltip>
						</div>
					{/if}
				{/if}

				<!-- {JSON.stringify(item.info)} -->

				{#if (item?.model?.tags ?? []).length > 0}
					{#key item.model.id}
						<Tooltip elementId="tags-{item.model.id}">
							<div slot="tooltip" id="tags-{item.model.id}">
								{#each item.model?.tags.sort((a, b) => a.name.localeCompare(b.name)) as tag}
									<Tooltip content={tag.name} className="flex-shrink-0">
										<div class=" text-xs font-medium rounded-sm uppercase text-white">
											{tag.name}
										</div>
									</Tooltip>
								{/each}
							</div>

							<div class="translate-y-[1px]">
								<Tag />
							</div>
						</Tooltip>
					{/key}
				{/if}

				{#if item.model?.direct}
					<Tooltip content={`${$i18n.t('Direct')}`}>
						<div class="translate-y-[1px]">
							<svg
								xmlns="http://www.w3.org/2000/svg"
								viewBox="0 0 16 16"
								fill="currentColor"
								class="size-3"
							>
								<path
									fill-rule="evenodd"
									d="M2 2.75A.75.75 0 0 1 2.75 2C8.963 2 14 7.037 14 13.25a.75.75 0 0 1-1.5 0c0-5.385-4.365-9.75-9.75-9.75A.75.75 0 0 1 2 2.75Zm0 4.5a.75.75 0 0 1 .75-.75 6.75 6.75 0 0 1 6.75 6.75.75.75 0 0 1-1.5 0C8 10.35 5.65 8 2.75 8A.75.75 0 0 1 2 7.25ZM3.5 11a1.5 1.5 0 1 0 0 3 1.5 1.5 0 0 0 0-3Z"
									clip-rule="evenodd"
								/>
							</svg>
						</div>
					</Tooltip>
				{:else if item.model.connection_type === 'external'}
					<Tooltip content={`${$i18n.t('External')}`}>
						<div class="translate-y-[1px]">
							<svg
								xmlns="http://www.w3.org/2000/svg"
								viewBox="0 0 16 16"
								fill="currentColor"
								class="size-3"
							>
								<path
									fill-rule="evenodd"
									d="M8.914 6.025a.75.75 0 0 1 1.06 0 3.5 3.5 0 0 1 0 4.95l-2 2a3.5 3.5 0 0 1-5.396-4.402.75.75 0 0 1 1.251.827 2 2 0 0 0 3.085 2.514l2-2a2 2 0 0 0 0-2.828.75.75 0 0 1 0-1.06Z"
									clip-rule="evenodd"
								/>
								<path
									fill-rule="evenodd"
									d="M7.086 9.975a.75.75 0 0 1-1.06 0 3.5 3.5 0 0 1 0-4.95l2-2a3.5 3.5 0 0 1 5.396 4.402.75.75 0 0 1-1.251-.827 2 2 0 0 0-3.085-2.514l-2 2a2 2 0 0 0 0 2.828.75.75 0 0 1 0 1.06Z"
									clip-rule="evenodd"
								/>
							</svg>
						</div>
					</Tooltip>
				{/if}

				{#if item.model?.info?.meta?.description}
					<Tooltip
						content={`${marked.parse(
							sanitizeResponseContent(item.model?.info?.meta?.description).replaceAll('\n', '<br>')
						)}`}
					>
						<div class=" translate-y-[1px]">
							<svg
								xmlns="http://www.w3.org/2000/svg"
								fill="none"
								viewBox="0 0 24 24"
								stroke-width="1.5"
								stroke="currentColor"
								class="w-4 h-4"
							>
								<path
									stroke-linecap="round"
									stroke-linejoin="round"
									d="m11.25 11.25.041-.02a.75.75 0 0 1 1.063.852l-.708 2.836a.75.75 0 0 0 1.063.853l.041-.021M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Zm-9-3.75h.008v.008H12V8.25Z"
								/>
							</svg>
						</div>
					</Tooltip>
				{/if}
			</div>
		</div>
	</div>

	<div class="ml-auto pl-2 pr-1 flex items-center gap-1.5 shrink-0">
		{#if $user?.role === 'admin' && item.model.owned_by === 'ollama' && item.model.ollama?.expires_at && new Date(item.model.ollama?.expires_at * 1000) > new Date()}
			<Tooltip
				content={`${$i18n.t('Eject')}`}
				className="flex-shrink-0 group-hover/item:opacity-100 opacity-0 "
			>
				<button
					class="flex"
					on:click={(e) => {
						e.preventDefault();
						e.stopPropagation();
						unloadModelHandler(item.value);
					}}
				>
					<ArrowUpTray className="size-3" />
				</button>
			</Tooltip>
		{/if}

		<ModelItemMenu
			bind:show={showMenu}
			model={item.model}
			{pinModelHandler}
			copyLinkHandler={() => {
				copyLinkHandler(item.model);
			}}
		>
			<button
				aria-label={`${$i18n.t('More Options')}`}
				class="flex"
				on:click={(e) => {
					e.preventDefault();
					e.stopPropagation();
					showMenu = !showMenu;
				}}
			>
				<EllipsisHorizontal />
			</button>
		</ModelItemMenu>

		{#if value === item.value}
			<div>
				<Check className="size-3" />
			</div>
		{/if}
	</div>
</button>
