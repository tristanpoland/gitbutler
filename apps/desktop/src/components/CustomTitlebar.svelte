<script lang="ts">
	import { goto } from '$app/navigation';
	import { page } from '$app/state';
	import CreateBranchModal from '$components/CreateBranchModal.svelte';
	import ShareIssueModal from '$components/ShareIssueModal.svelte';
	import { BACKEND } from '$lib/backend';
	import { FILE_SERVICE } from '$lib/files/fileService';
	import { handleAddProjectOutcome, vscodePath } from '$lib/project/project';
	import { PROJECTS_SERVICE } from '$lib/project/projectsService';
	import { clonePath, historyPath, projectPath } from '$lib/routes/routes.svelte';
	import { useSettingsModal } from '$lib/settings/settingsModal.svelte';
	import { SETTINGS } from '$lib/settings/userSettings';
	import { UI_STATE } from '$lib/state/uiState.svelte';
	import { UPDATER_SERVICE } from '$lib/updater/updater';
	import { getEditorUri, URL_SERVICE } from '$lib/utils/url';
	import { inject } from '@gitbutler/core/context';

	const backend = inject(BACKEND);
	const projectsService = inject(PROJECTS_SERVICE);
	const urlService = inject(URL_SERVICE);
	const userSettings = inject(SETTINGS);
	const fileService = inject(FILE_SERVICE);
	const uiState = inject(UI_STATE);
	const updaterService = inject(UPDATER_SERVICE);
	const { openGeneralSettings, openProjectSettings } = useSettingsModal();

	const projectId = $derived(page.params.projectId);

	let openMenu = $state<string | null>(null);
	let createBranchModal = $state<CreateBranchModal>();
	let shareIssueModal = $state<ShareIssueModal>();

	// File menu actions
	async function addLocalRepo() {
		const outcome = await projectsService.addProject();
		if (!outcome) return;
		handleAddProjectOutcome(outcome, (project) => goto(projectPath(project.id)));
		openMenu = null;
	}

	function cloneRepo() {
		goto(clonePath());
		openMenu = null;
	}

	async function quit() {
		await backend.invoke('quit', {});
	}

	function createBranch() {
		createBranchModal?.show();
		openMenu = null;
	}

	function createDependentBranch() {
		createBranchModal?.show('dependent');
		openMenu = null;
	}

	// View menu actions
	function switchTheme() {
		userSettings.update((s) => ({
			...s,
			theme: s.theme === 'light' ? 'dark' : 'light'
		}));
		openMenu = null;
	}

	function toggleSidebar() {
		const unassignedSidebarFolded = uiState.global.unassignedSidebarFolded;
		unassignedSidebarFolded.set(!unassignedSidebarFolded.current);
		openMenu = null;
	}

	function zoomIn() {
		const MIN_ZOOM = 0.375;
		const MAX_ZOOM = 3;
		const ZOOM_STEP = 0.0625;
		const currentZoom = $userSettings.zoom;
		const newZoom = Math.min(Math.max(currentZoom + ZOOM_STEP, MIN_ZOOM), MAX_ZOOM);
		document.documentElement.style.fontSize = newZoom + 'rem';
		userSettings.update((s) => ({ ...s, zoom: newZoom }));
		openMenu = null;
	}

	function zoomOut() {
		const MIN_ZOOM = 0.375;
		const MAX_ZOOM = 3;
		const ZOOM_STEP = 0.0625;
		const currentZoom = $userSettings.zoom;
		const newZoom = Math.min(Math.max(currentZoom - ZOOM_STEP, MIN_ZOOM), MAX_ZOOM);
		document.documentElement.style.fontSize = newZoom + 'rem';
		userSettings.update((s) => ({ ...s, zoom: newZoom }));
		openMenu = null;
	}

	function zoomReset() {
		const DEFAULT_ZOOM = 1;
		document.documentElement.style.fontSize = DEFAULT_ZOOM + 'rem';
		userSettings.update((s) => ({ ...s, zoom: DEFAULT_ZOOM }));
		openMenu = null;
	}

	function reload() {
		location.reload();
		openMenu = null;
	}

	// Project menu actions
	function goToHistory() {
		if (!projectId) return;
		goto(historyPath(projectId));
		openMenu = null;
	}

	async function openInEditor() {
		if (!projectId) return;
		const project = await projectsService.fetchProject(projectId);
		if (!project) return;
		urlService.openExternalUrl(
			getEditorUri({
				schemeId: $userSettings.defaultCodeEditor.schemeIdentifer,
				path: [vscodePath(project.path)]
			})
		);
		openMenu = null;
	}

	async function showInFileManager() {
		if (!projectId) return;
		const project = await projectsService.fetchProject(projectId);
		if (!project) return;
		await fileService.showFileInFolder(project.path);
		openMenu = null;
	}

	function openProjectSettingsMenu() {
		if (!projectId) return;
		openProjectSettings(projectId);
		openMenu = null;
	}

	// Global menu actions
	function openGlobalSettingsMenu() {
		openGeneralSettings();
		openMenu = null;
	}

	function checkForUpdates() {
		updaterService.checkForUpdate(true);
		openMenu = null;
	}

	// Help menu actions
	function openExternal(url: string) {
		urlService.openExternalUrl(url);
		openMenu = null;
	}

	function shareDebugInfo() {
		shareIssueModal?.show();
		openMenu = null;
	}

	function toggleMenu(menuName: string) {
		openMenu = openMenu === menuName ? null : menuName;
	}

	async function minimizeWindow() {
		await backend.invoke('window_minimize');
	}

	async function maximizeWindow() {
		await backend.invoke('window_toggle_maximize');
	}

	async function closeWindow() {
		await backend.invoke('window_close');
	}
</script>

<div class="custom-titlebar" data-tauri-drag-region>
	<div class="titlebar-menus">
		<div class="menu">
			<button
				type="button"
				class="menu-button"
				class:active={openMenu === 'file'}
				onclick={() => toggleMenu('file')}
			>
				File
			</button>
			{#if openMenu === 'file'}
				<div class="menu-dropdown">
					<button type="button" class="menu-item" onclick={addLocalRepo}>
						Add Local Repository…
						<span class="shortcut">Ctrl+O</span>
					</button>
					<button type="button" class="menu-item" onclick={cloneRepo}>
						Clone Repository…
						<span class="shortcut">Ctrl+Shift+O</span>
					</button>
					<div class="menu-separator"></div>
					<button type="button" class="menu-item" onclick={createBranch}>
						Create Branch…
						<span class="shortcut">Ctrl+B</span>
					</button>
					<button type="button" class="menu-item" onclick={createDependentBranch}>
						Create Dependent Branch…
						<span class="shortcut">Ctrl+Shift+B</span>
					</button>
					<div class="menu-separator"></div>
					<button type="button" class="menu-item" onclick={quit}>Quit</button>
					<button type="button" class="menu-item" onclick={checkForUpdates}>
						Check for updates…
					</button>
				</div>
			{/if}
		</div>

		<!-- View Menu -->
		<div class="menu">
			<button
				type="button"
				class="menu-button"
				class:active={openMenu === 'view'}
				onclick={() => toggleMenu('view')}
			>
				View
			</button>
			{#if openMenu === 'view'}
				<div class="menu-dropdown">
					<button type="button" class="menu-item" onclick={switchTheme}>
						Switch Theme
						<span class="shortcut">Ctrl+T</span>
					</button>
					<button type="button" class="menu-item" onclick={toggleSidebar}>
						Toggle Unassigned
						<span class="shortcut">Ctrl+\</span>
					</button>
					<div class="menu-separator"></div>
					<button type="button" class="menu-item" onclick={zoomIn}>
						Zoom In
						<span class="shortcut">Ctrl+=</span>
					</button>
					<button type="button" class="menu-item" onclick={zoomOut}>
						Zoom Out
						<span class="shortcut">Ctrl+-</span>
					</button>
					<button type="button" class="menu-item" onclick={zoomReset}>
						Reset Zoom
						<span class="shortcut">Ctrl+0</span>
					</button>
					<div class="menu-separator"></div>
					<button type="button" class="menu-item" onclick={reload}>
						Reload View
						<span class="shortcut">Ctrl+R</span>
					</button>
				</div>
			{/if}
		</div>

		<!-- Project Menu -->
		<div class="menu">
			<button
				type="button"
				class="menu-button"
				class:active={openMenu === 'project'}
				onclick={() => toggleMenu('project')}
			>
				Project
			</button>
			{#if openMenu === 'project'}
				<div class="menu-dropdown">
					<button type="button" class="menu-item" onclick={goToHistory} disabled={!projectId}>
						Operations History
						<span class="shortcut">Ctrl+Shift+H</span>
					</button>
					<div class="menu-separator"></div>
					<button type="button" class="menu-item" onclick={openInEditor} disabled={!projectId}>
						Open in Editor
					</button>
					<button type="button" class="menu-item" onclick={showInFileManager} disabled={!projectId}>
						Show in File Manager
					</button>
					<div class="menu-separator"></div>
					<button
						type="button"
						class="menu-item"
						onclick={openProjectSettingsMenu}
						disabled={!projectId}
					>
						Project Settings
					</button>
				</div>
			{/if}
		</div>

		<!-- Help Menu -->
		<div class="menu">
			<button
				type="button"
				class="menu-button"
				class:active={openMenu === 'help'}
				onclick={() => toggleMenu('help')}
			>
				Help
			</button>
			{#if openMenu === 'help'}
				<div class="menu-dropdown">
					<button
						type="button"
						class="menu-item"
						onclick={() => openExternal('https://docs.gitbutler.com')}
					>
						Documentation
					</button>
					<button
						type="button"
						class="menu-item"
						onclick={() => openExternal('https://github.com/gitbutlerapp/gitbutler')}
					>
						Source Code
					</button>
					<button
						type="button"
						class="menu-item"
						onclick={() => openExternal('https://github.com/gitbutlerapp/gitbutler/releases')}
					>
						Release Notes
					</button>
					<div class="menu-separator"></div>
					<button type="button" class="menu-item" onclick={shareDebugInfo}>
						Share Debug Info…
					</button>
					<button
						type="button"
						class="menu-item"
						onclick={() =>
							openExternal('https://github.com/gitbutlerapp/gitbutler/issues/new/choose')}
					>
						Report an Issue…
					</button>
					<div class="menu-separator"></div>
					<button
						type="button"
						class="menu-item"
						onclick={() => openExternal('https://discord.com/invite/MmFkmaJ42D')}
					>
						Discord
					</button>
					<button
						type="button"
						class="menu-item"
						onclick={() => openExternal('https://www.youtube.com/@gitbutlerapp')}
					>
						YouTube
					</button>
					<button
						type="button"
						class="menu-item"
						onclick={() => openExternal('https://bsky.app/profile/gitbutler.com')}
					>
						Bluesky
					</button>
					<button
						type="button"
						class="menu-item"
						onclick={() => openExternal('https://x.com/gitbutler')}
					>
						X
					</button>
					<div class="menu-separator"></div>
					<button type="button" class="menu-item" disabled>
						Version {#await backend.currentVersion() then version}{version}{:catch}0.0.0{/await}
					</button>
				</div>
			{/if}
		</div>

		<!-- Settings Menu -->
		<div class="menu">
			<button
				type="button"
				class="menu-button"
				class:active={openMenu === 'settings'}
				onclick={() => toggleMenu('settings')}
			>
				Settings
			</button>
			{#if openMenu === 'settings'}
				<div class="menu-dropdown">
					<button type="button" class="menu-item" onclick={openGlobalSettingsMenu}>
						Settings
						<span class="shortcut">Ctrl+,</span>
					</button>
				</div>
			{/if}
		</div>
	</div>

	<!-- Window Controls -->
	<div class="window-controls">
		<button type="button" class="window-button" onclick={minimizeWindow} title="Minimize">
			<span>−</span>
		</button>
		<button type="button" class="window-button" onclick={maximizeWindow} title="Maximize">
			<span>□</span>
		</button>
		<button type="button" class="window-button close" onclick={closeWindow} title="Close">
			<span>×</span>
		</button>
	</div>
</div>

<!-- Close menu when clicking outside -->
{#if openMenu}
	<button
		type="button"
		class="menu-overlay"
		onclick={() => (openMenu = null)}
		aria-label="Close menu"
	></button>
{/if}

{#if projectId}
	<CreateBranchModal bind:this={createBranchModal} {projectId} />
{/if}
<ShareIssueModal bind:this={shareIssueModal} />

<style>
	.custom-titlebar {
		display: flex;
		align-items: center;
		justify-content: space-between;
		height: 32px;
		background: var(--clr-bg-1);
		border-bottom: 1px solid var(--clr-border-2);
		user-select: none;
		-webkit-app-region: drag;
	}

	.titlebar-menus {
		display: flex;
		gap: 0;
		-webkit-app-region: no-drag;
	}

	.menu {
		position: relative;
	}

	.menu-button {
		padding: 0 12px;
		height: 32px;
		background: none;
		border: none;
		color: var(--clr-text-1);
		font-size: 13px;
		cursor: pointer;
		transition: background-color 0.1s;
	}

	.menu-button:hover,
	.menu-button.active {
		background: var(--clr-bg-2);
	}

	.menu-dropdown {
		position: absolute;
		top: 100%;
		left: 0;
		min-width: 220px;
		background: var(--clr-bg-1);
		border: 1px solid var(--clr-border-2);
		border-radius: 6px;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
		padding: 4px 0;
		z-index: 10000;
	}

	.menu-item {
		width: 100%;
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 6px 12px;
		background: none;
		border: none;
		color: var(--clr-text-1);
		font-size: 13px;
		text-align: left;
		cursor: pointer;
		transition: background-color 0.1s;
	}

	.menu-item:hover:not(:disabled) {
		background: var(--clr-bg-2);
	}

	.menu-item:disabled {
		opacity: 0.5;
		cursor: default;
	}

	.menu-separator {
		height: 1px;
		background: var(--clr-border-2);
		margin: 4px 0;
	}

	.shortcut {
		font-size: 11px;
		color: var(--clr-text-3);
		margin-left: 24px;
	}

	.window-controls {
		display: flex;
		-webkit-app-region: no-drag;
	}

	.window-button {
		width: 46px;
		height: 32px;
		background: none;
		border: none;
		color: var(--clr-text-1);
		font-size: 16px;
		cursor: pointer;
		transition: background-color 0.1s;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.window-button:hover {
		background: var(--clr-bg-2);
	}

	.window-button.close:hover {
		background: #e81123;
		color: white;
	}

	.menu-overlay {
		position: fixed;
		inset: 0;
		background: transparent;
		border: none;
		cursor: default;
		z-index: 9999;
	}
</style>
