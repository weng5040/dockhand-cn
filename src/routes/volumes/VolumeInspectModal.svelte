<script lang="ts">
	import * as Dialog from '$lib/components/ui/dialog';
	import { Button } from '$lib/components/ui/button';
	import { Badge } from '$lib/components/ui/badge';
	import { Loader2, HardDrive } from 'lucide-svelte';
	import { currentEnvironment, appendEnvParam } from '$lib/stores/environment';
	import { formatDateTime } from '$lib/stores/settings';

	interface Props {
		open: boolean;
		volumeName: string;
	}

	let { open = $bindable(), volumeName }: Props = $props();

	let loading = $state(true);
	let error = $state('');
	let volumeData = $state<any>(null);

	$effect(() => {
		if (open && volumeName) {
			fetchVolumeInspect();
		}
	});

	async function fetchVolumeInspect() {
		loading = true;
		error = '';
		try {
			const envId = $currentEnvironment?.id ?? null;
			const response = await fetch(appendEnvParam(`/api/volumes/${encodeURIComponent(volumeName)}/inspect`, envId));
			if (!response.ok) {
				throw new Error('Failed to fetch volume details');
			}
			volumeData = await response.json();
		} catch (err: any) {
			error = err.message || 'Failed to load volume details';
			console.error('Failed to fetch volume inspect:', err);
		} finally {
			loading = false;
		}
	}

	function formatDate(dateString: string): string {
		if (!dateString) return 'N/A';
		return formatDateTime(dateString);
	}
</script>

<Dialog.Root bind:open>
	<Dialog.Content class="max-w-4xl max-h-[90vh] flex flex-col">
		<Dialog.Header class="shrink-0">
			<Dialog.Title class="flex items-center gap-2">
				<HardDrive class="w-5 h-5" />
				数据卷详情: <span class="text-muted-foreground font-normal break-all">{volumeName}</span>
			</Dialog.Title>
		</Dialog.Header>

		<div class="flex-1 overflow-auto space-y-4 min-h-0">
			{#if loading}
				<div class="flex items-center justify-center py-8">
					<Loader2 class="w-6 h-6 animate-spin text-muted-foreground" />
				</div>
			{:else if error}
				<div class="text-sm text-red-600 dark:text-red-400 p-3 bg-red-50 dark:bg-red-950 rounded">
					{error}
				</div>
			{:else if volumeData}
				<!-- Basic Info -->
				<div class="space-y-3">
					<h3 class="text-sm font-semibold">基本信息</h3>
					<div class="grid grid-cols-2 gap-3 text-sm">
						<div>
							<p class="text-muted-foreground">名称</p>
							<code class="text-xs break-all">{volumeData.Name}</code>
						</div>
						<div>
							<p class="text-muted-foreground">驱动</p>
							<Badge variant="outline">{volumeData.Driver}</Badge>
						</div>
						<div>
							<p class="text-muted-foreground">范围</p>
							<Badge variant="secondary">{volumeData.Scope}</Badge>
						</div>
						<div>
							<p class="text-muted-foreground">创建时间</p>
							<p class="text-xs">{formatDate(volumeData.CreatedAt)}</p>
						</div>
					</div>
				</div>

				<!-- Mountpoint -->
				<div class="space-y-2">
					<h3 class="text-sm font-semibold">挂载点</h3>
					<div class="p-2 bg-muted rounded">
						<code class="text-xs break-all">{volumeData.Mountpoint}</code>
					</div>
					<p class="text-xs text-muted-foreground">
						数据卷数据存储在主机上的位置
					</p>
				</div>

				<!-- Driver Options -->
				{#if volumeData.Options && Object.keys(volumeData.Options).length > 0}
					<div class="space-y-3">
						<h3 class="text-sm font-semibold">驱动选项</h3>
						<div class="space-y-1">
							{#each Object.entries(volumeData.Options) as [key, value]}
								<div class="flex justify-between text-sm p-2 bg-muted rounded">
									<code class="text-muted-foreground">{key}</code>
									<code class="break-all ml-2">{value}</code>
								</div>
							{/each}
						</div>
					</div>
				{:else}
					<div class="space-y-2">
						<h3 class="text-sm font-semibold">驱动选项</h3>
						<p class="text-sm text-muted-foreground">未配置驱动选项</p>
					</div>
				{/if}

				<!-- Labels -->
				{#if volumeData.Labels && Object.keys(volumeData.Labels).length > 0}
					<div class="space-y-3">
						<h3 class="text-sm font-semibold">标签</h3>
						<div class="space-y-1">
							{#each Object.entries(volumeData.Labels) as [key, value]}
								<div class="flex justify-between text-sm p-2 bg-muted rounded">
									<code class="text-muted-foreground">{key}</code>
									<code class="break-all ml-2">{value}</code>
								</div>
							{/each}
						</div>
					</div>
				{/if}

				<!-- Status -->
				{#if volumeData.Status}
					<div class="space-y-3">
						<h3 class="text-sm font-semibold">状态</h3>
						<div class="space-y-1">
							{#each Object.entries(volumeData.Status) as [key, value]}
								<div class="flex justify-between text-sm p-2 bg-muted rounded">
									<span class="text-muted-foreground">{key}</span>
									<span>{value}</span>
								</div>
							{/each}
						</div>
					</div>
				{/if}

				<!-- Usage Warning -->
				<div class="p-3 bg-yellow-50 dark:bg-yellow-950/30 border border-yellow-200 dark:border-yellow-800 rounded">
					<p class="text-xs text-yellow-800 dark:text-yellow-200">
						<strong>注意:</strong> 删除此数据卷将永久删除其中存储的所有数据。
						删除前请确保没有容器正在使用此数据卷。
					</p>
				</div>
			{/if}
		</div>

		<Dialog.Footer class="shrink-0">
			<Button variant="outline" onclick={() => (open = false)}>关闭</Button>
		</Dialog.Footer>
	</Dialog.Content>
</Dialog.Root>