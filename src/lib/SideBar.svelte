<script>
	import Icon from './Icon.svelte';
	import InformationTab from './InformationTab.svelte';
	import NetworkingTab from './NetworkingTab.svelte';
	import CpuTab from './CpuTab.svelte';
	import DiskTab from './DiskTab.svelte';
	import AnthropicTab from './AnthropicTab.svelte';
	import PostsTab from './PostsTab.svelte';
	import DiscordTab from './DiscordTab.svelte';
	import GitHubTab from './GitHubTab.svelte';
	import { cpuActivity, diskActivity, aiActivity } from './activities.js';
	const icons = [
		{ icon: 'fas fa-info-circle', info: 'Information', activity: null },
		{ icon: 'fas fa-wifi', info: 'Networking', activity: null },
		{ icon: 'fas fa-microchip', info: 'CPU', activity: cpuActivity },
		{ icon: 'fas fa-compact-disc', info: 'Disk', activity: diskActivity },
		{ icon: 'fas fa-robot', info: 'ClaudeAI', activity: aiActivity },
		null,
		{ icon: 'fas fa-book-open', info: 'Posts', activity: null },
		{ icon: 'fab fa-discord', info: 'Discord', activity: null },
		{ icon: 'fab fa-github', info: 'GitHub', activity: null },
	];
	let activeInfo = null; // Tracks currently visible info
	let lastHoveredInfo = null; // Tracks last hovered info
	let isPanelHovered = false; // Tracks if info panel is hovered
	let hideTimeout = 0; // Timeout for hiding info panel

	function showInfo(info) {
		// Clears and reset the active hideTimeout to stop the panel from being hidden
		if (hideTimeout) {
			clearTimeout(hideTimeout);
			hideTimeout = 0;
		}
		// Updates activeInfo and lastHoveredInfo to track the hovered icon, then shows the panel with the hovered icon's info
		if (info !== lastHoveredInfo) {
			activeInfo = info;
			lastHoveredInfo = info;
		}
	}
	function hideInfo() {
		// Set a timeout to hide the panel after 400ms, but only if the panel isn't hovered
		if (!hideTimeout) {
			if (!isPanelHovered) {
				hideTimeout = setTimeout(() => {
					activeInfo = null;
					lastHoveredInfo = null;
					hideTimeout = 0;
				}, 400);
			}
		}
	}
	function handleMouseOverPanel() {
		isPanelHovered = true;
		if (hideTimeout) {
			clearTimeout(hideTimeout);
			hideTimeout = 0;
		}
	}
	function handleMouseLeavePanel() {
		isPanelHovered = false;
		hideInfo();
	}
	// Toggles the info panel for the clicked icon
	function handleClick(icon) {
		// Hides the panel if the icon is active. Otherwise, shows the panel with info
		if (activeInfo === icon.info) {
			activeInfo = null;
		} else {
			activeInfo = icon.info;
		}
	}

	export let handleTool;
</script>

<div class="flex flex-row w-14 h-full bg-neutral-700" on:mouseleave={hideInfo}>
	<div class="flex flex-col shrink-0 w-14 text-gray-300">
		{#each icons as i}
			{#if i}
				<Icon
					icon={i.icon}
					info={i.info}
					activity={i.activity}
					on:mouseover={(e) => showInfo(e.detail)}
					on:click={() => handleClick(i)}
				/>
			{:else}
				<div class="grow pointer-events-none"></div>
			{/if}
		{/each}
	</div>
	<div
		class="flex flex-col gap-5 shrink-0 w-80 h-full z-10 p-2 bg-neutral-600 text-gray-100 opacity-95"
		class:hidden={!activeInfo}
		on:mouseover={handleMouseOverPanel}
		on:mouseleave={handleMouseLeavePanel}
	>
		{#if activeInfo === 'Information'}
			<InformationTab>
				<slot></slot>
			</InformationTab>
		{:else if activeInfo === 'Networking'}
			<NetworkingTab on:connect/>
		{:else if activeInfo === 'CPU'}
			<CpuTab/>
		{:else if activeInfo === 'Disk'}
			<DiskTab on:reset/>
		{:else if activeInfo === 'ClaudeAI'}
			<AnthropicTab handleTool={handleTool} />
		{:else if activeInfo === 'Posts'}
			<PostsTab/>
		{:else if activeInfo === 'Discord'}
			<DiscordTab/>
		{:else if activeInfo === 'GitHub'}
			<GitHubTab/>
		{:else}
			<p>TODO: {activeInfo}</p>
		{/if}

		<div class="mt-auto text-sm text-gray-300">
			<div class="pt-1 pb-1">
				<a href="https://cheerpx.io/" target="_blank">
					<span>Powered by CheerpX</span>
					<img src="assets/cheerpx.svg" alt="CheerpX Logo" class="w-6 h-6 inline-block">
				</a>
			</div>
			<hr class="border-t border-solid border-gray-300">
			<div class="pt-1 pb-1">
				<a href="https://leaningtech.com/" target="”_blank”">© 2022-2025 Leaning Technologies</a>
			</div>
		</div>
	</div>
</div>
