<script lang="ts">
	import '../app.css';
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	import { page } from '$app/stores';
	import { Toaster } from '$lib/components/ui/sonner';
	import AppSidebar from '$lib/components/app-sidebar.svelte';
	import ThemeToggle from '$lib/components/theme-toggle.svelte';
	import HostInfo from '$lib/components/host-info.svelte';
	import MainContent from '$lib/components/main-content.svelte';
	import CommandPalette from '$lib/components/CommandPalette.svelte';
	import WhatsNewModal from '$lib/components/WhatsNewModal.svelte';
	import { SidebarProvider, SidebarTrigger } from '$lib/components/ui/sidebar';
	import { connectSSE, disconnectSSE } from '$lib/stores/events';
	import { currentEnvironment, environments } from '$lib/stores/environment';
	import { licenseStore, daysUntilExpiry } from '$lib/stores/license';
	import { authStore } from '$lib/stores/auth';
	import { themeStore, applyTheme } from '$lib/stores/theme';
	import { gridPreferencesStore } from '$lib/stores/grid-preferences';
	import { shouldShowWhatsNew } from '$lib/utils/version';
	import { AlertTriangle, Search } from 'lucide-svelte';

	// 判断当前路由是否为登录页（登录页不需要侵边栏）
	const isLoginPage = $derived($page.url.pathname === '/login');

	let { children } = $props();
	let envId = $state<number | null>(null);
	let commandPaletteOpen = $state(false);

	// 新功能弹窗状态
	let showWhatsNewModal = $state(false);
	let changelog = $state<Array<{ version: string; date: string; changes: Array<{ type: string; text: string }> }>>([]);
	let lastSeenVersion = $state<string | null>(null);

	// 应用版本（构建时通过 git tag 注入）
	declare const __APP_VERSION__: string | null;
	const currentVersion = __APP_VERSION__;

	// 检测是否为 Mac（用于快捷键提示）
	const isMac = typeof navigator !== 'undefined' && navigator.platform?.toUpperCase().indexOf('MAC') >= 0;

	// 监听环境切换
	$effect(() => {
		const env = $currentEnvironment;
		envId = env?.id ?? null;
	});

	// 认证状态确认后初始化主题
	$effect(() => {
		if (!$authStore.loading) {
			const userId = $authStore.authEnabled && $authStore.user ? $authStore.user.id : undefined;
			themeStore.init(userId);
		}
	});

	// 用户登录后刷新环境列表（处理 OIDC 回调的服务端登录）
	let wasAuthenticated = $state(false);
	$effect(() => {
		if (!$authStore.loading && $authStore.authenticated && !wasAuthenticated) {
			environments.refresh();
		}
		if (!$authStore.loading) {
			wasAuthenticated = $authStore.authenticated;
		}
	});

	onMount(() => {
		// 立即应用主题（防止闪烁）
		applyTheme(themeStore.get());

		// 初始化表格布局偏好
		gridPreferencesStore.init();

		// 建立 SSE 连接，实时接收 Docker 事件
		connectSSE(envId);

		// 检查企业版许可证状态
		licenseStore.check();

		// 检查认证状态
		authStore.check();

		return () => {
			disconnectSSE();
		};
	});

	async function checkWhatsNew() {
		if (browser && currentVersion && currentVersion !== 'unknown') {
			lastSeenVersion = localStorage.getItem('dockhand-whats-new-version');
			if (shouldShowWhatsNew(currentVersion, lastSeenVersion)) {
				try {
					const res = await fetch('/api/changelog');
					if (res.ok) {
						changelog = await res.json();
						showWhatsNewModal = true;
					}
				} catch {
					// 静默失败，获取 changelog 失败时不弹窗
				}
			}
		}
	}

	function dismissWhatsNew() {
		showWhatsNewModal = false;
		if (browser && currentVersion) {
			localStorage.setItem('dockhand-whats-new-version', currentVersion);
		}
	}

	// 认证完成后再显示新功能（避免在登录页泄露版本信息 #717）
	let whatsNewChecked = false;
	$effect(() => {
		if (!whatsNewChecked && !$authStore.loading && (!$authStore.authEnabled || $authStore.authenticated)) {
			whatsNewChecked = true;
			checkWhatsNew();
		}
	});
</script>

<svelte:head>
	<link rel="icon" href="/logo_light.webp" />
	<title>Dockhand - Docker 管理面板</title>
</svelte:head>

{#if isLoginPage}
	<!-- 登录页：无侵边栏和顶栏 -->
	{@render children?.()}
	<Toaster richColors position="bottom-right" />
{:else}
	<!-- 主应用：带侵边栏的完整布局 -->
	<SidebarProvider>
		<AppSidebar />
		<MainContent>
			<header class="h-14 shrink-0 flex items-center justify-between gap-4 border-b bg-background px-4">
				<div class="flex items-center gap-2 min-w-0">
					<SidebarTrigger class="md:hidden shrink-0" />
					<HostInfo />
				</div>
				<div class="flex items-center gap-3 shrink-0">
					<button
						type="button"
						onclick={() => commandPaletteOpen = true}
						class="flex items-center gap-2 px-2.5 py-1.5 text-xs text-muted-foreground hover:text-foreground border rounded-md hover:bg-muted/50 transition-colors"
					>
						<Search class="w-3.5 h-3.5" />
						<span class="hidden sm:inline">搜索...</span>
						<kbd class="pointer-events-none hidden sm:inline-flex h-5 select-none items-center gap-1 rounded border bg-muted px-1.5 font-mono text-2xs font-medium text-muted-foreground">
							{#if isMac}
								<span class="text-xs">⌘</span>
							{:else}
								<span class="text-xs">Ctrl</span>
							{/if}
							K
						</kbd>
					</button>
					{#if $licenseStore.isEnterprise && $daysUntilExpiry !== null && $daysUntilExpiry <= 30}
						<a
							href="/settings?tab=license"
							class="flex items-center gap-1.5 px-2.5 py-1 rounded-md text-xs font-medium transition-colors
								{$daysUntilExpiry <= 7
									? 'bg-red-100 text-red-800 hover:bg-red-200 dark:bg-red-900/30 dark:text-red-400 dark:hover:bg-red-900/50'
									: 'bg-amber-100 text-amber-800 hover:bg-amber-200 dark:bg-amber-900/30 dark:text-amber-400 dark:hover:bg-amber-900/50'}"
						>
							<AlertTriangle class="w-3.5 h-3.5" />
							{#if $daysUntilExpiry <= 0}
								许可证已过期
							{:else if $daysUntilExpiry === 1}
								许可证明天到期
							{:else}
								许可证将在 {$daysUntilExpiry} 天后到期
							{/if}
						</a>
					{/if}
					<ThemeToggle />
				</div>
			</header>
			<div class="flex-1 min-h-0 h-[calc(100%-3.5rem)] overflow-auto py-2 px-3 flex flex-col">
				{@render children?.()}
			</div>
		</MainContent>
	</SidebarProvider>

	<Toaster richColors position="bottom-right" />
	<CommandPalette bind:open={commandPaletteOpen} />
{/if}

{#if showWhatsNewModal && currentVersion}
	<WhatsNewModal
		bind:open={showWhatsNewModal}
		version={currentVersion}
		{lastSeenVersion}
		{changelog}
		onDismiss={dismissWhatsNew}
	/>
{/if}
