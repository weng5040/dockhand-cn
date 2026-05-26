<script lang="ts">
	import { Button } from '$lib/components/ui/button';
	import * as Dialog from '$lib/components/ui/dialog';
	import { Label } from '$lib/components/ui/label';
	import { Input } from '$lib/components/ui/input';
	import { QrCode, RefreshCw, ShieldCheck, TriangleAlert, Copy, Download, Check, XCircle } from 'lucide-svelte';
	import * as Tooltip from '$lib/components/ui/tooltip';
	import { copyToClipboard } from '$lib/utils/clipboard';
	import * as Alert from '$lib/components/ui/alert';
	import { focusFirstInput } from '$lib/utils';

	interface Props {
		open: boolean;
		qrCode: string;
		secret: string;
		userId: number;
		onClose: () => void;
		onSuccess: () => void;
	}

	let { open = $bindable(), qrCode, secret, userId, onClose, onSuccess }: Props = $props();

	let token = $state('');
	let loading = $state(false);
	let error = $state('');
	let backupCodes = $state<string[]>([]);
	let showBackupCodes = $state(false);
	let copied = $state<'ok' | 'error' | null>(null);

	function resetForm() {
		token = '';
		error = '';
		backupCodes = [];
		showBackupCodes = false;
		copied = null;
	}

	async function verifyAndEnableMfa() {
		if (!token) {
			error = '请输入验证码';
			return;
		}

		loading = true;
		error = '';

		try {
			const response = await fetch(`/api/users/${userId}/mfa`, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ action: 'verify', token })
			});

			const data = await response.json();

			if (response.ok) {
				backupCodes = data.backupCodes || [];
				showBackupCodes = true;
			} else {
				error = data.error || '验证码无效';
			}
		} catch (e) {
			error = 'MFA验证失败';
		} finally {
			loading = false;
		}
	}

	function formatBackupCodes(): string {
		return backupCodes.map((code, i) => `${i + 1}. ${code}`).join('\n');
	}

	async function copyBackupCodes() {
		const ok = await copyToClipboard(formatBackupCodes());
		copied = ok ? 'ok' : 'error';
		setTimeout(() => copied = null, 2000);
	}

	function downloadBackupCodes() {
		const content = `Dockhand MFA 备用码\n${'='.repeat(30)}\n\n如果无法访问身份验证器应用，可以使用这些代码登录。\n每个代码只能使用一次。\n\n${formatBackupCodes()}\n\n生成时间: ${new Date().toISOString()}`;
		const blob = new Blob([content], { type: 'text/plain' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = 'dockhand-backup-codes.txt';
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
		URL.revokeObjectURL(url);
	}

	function handleDone() {
		onSuccess();
		onClose();
	}

</script>

<Dialog.Root bind:open onOpenChange={(o) => { if (o) { resetForm(); focusFirstInput(); } else if (!showBackupCodes) onClose(); }}>
	<Dialog.Content class="max-w-md">
		<Dialog.Header>
			<Dialog.Title class="flex items-center gap-2">
				{#if showBackupCodes}
					<ShieldCheck class="w-5 h-5 text-green-500" />
					MFA启用成功
				{:else}
					<QrCode class="w-5 h-5" />
					设置双因素认证
				{/if}
			</Dialog.Title>
		</Dialog.Header>

		{#if showBackupCodes}
			<!-- 备用码视图 -->
			<div class="space-y-4">
				<Alert.Root>
					<TriangleAlert class="h-4 w-4" />
					<Alert.Description>
						请将这些备用码保存在安全的地方。如果无法访问身份验证器应用，每个代码只能使用一次进行登录。
					</Alert.Description>
				</Alert.Root>

				<div class="grid grid-cols-2 gap-2 p-3 bg-muted rounded-lg font-mono text-sm">
					{#each backupCodes as code, i}
						<div class="flex items-center gap-2">
							<span class="text-muted-foreground w-4">{i + 1}.</span>
							<span>{code}</span>
						</div>
					{/each}
				</div>

				<div class="flex gap-2">
					<Button variant="outline" class="flex-1" onclick={copyBackupCodes}>
						{#if copied === 'error'}
							<Tooltip.Root open>
								<Tooltip.Trigger>
									<XCircle class="w-4 h-4 text-red-500" />
								</Tooltip.Trigger>
								<Tooltip.Content>复制需要HTTPS</Tooltip.Content>
							</Tooltip.Root>
							失败
						{:else if copied === 'ok'}
							<Check class="w-4 h-4" />
							已复制！
						{:else}
							<Copy class="w-4 h-4" />
							复制代码
						{/if}
					</Button>
					<Button variant="outline" class="flex-1" onclick={downloadBackupCodes}>
						<Download class="w-4 h-4" />
						下载
					</Button>
				</div>
			</div>
			<Dialog.Footer>
				<Button onclick={handleDone}>
					<ShieldCheck class="w-4 h-4" />
					完成
				</Button>
			</Dialog.Footer>
		{:else}
			<!-- 设置视图 -->
			<div class="space-y-4">
				{#if error}
					<Alert.Root variant="destructive">
						<TriangleAlert class="h-4 w-4" />
						<Alert.Description>{error}</Alert.Description>
					</Alert.Root>
				{/if}

				<p class="text-sm text-muted-foreground">
					使用您的身份验证器应用（Google Authenticator、Authy等）扫描此二维码
				</p>

				{#if qrCode}
					<div class="flex justify-center p-4 bg-white rounded-lg">
						<img src={qrCode} alt="MFA二维码" class="w-48 h-48" />
					</div>
				{/if}

				<div class="space-y-2">
					<Label class="text-xs text-muted-foreground">或者手动输入此代码：</Label>
					<code class="block p-2 bg-muted rounded text-sm font-mono break-all">{secret}</code>
				</div>

				<div class="space-y-2">
					<Label>验证码</Label>
					<Input
						bind:value={token}
						name="totp"
						placeholder="输入6位数字代码"
						maxlength={6}
						autocomplete="one-time-code"
					/>
					<p class="text-xs text-muted-foreground">
						输入身份验证器应用中的代码以验证设置
					</p>
				</div>
			</div>
			<Dialog.Footer>
				<Button variant="outline" onclick={onClose}>取消</Button>
				<Button onclick={verifyAndEnableMfa} disabled={loading || !token}>
					{#if loading}
						<RefreshCw class="w-4 h-4 animate-spin" />
					{:else}
						<ShieldCheck class="w-4 h-4" />
					{/if}
					启用MFA
				</Button>
			</Dialog.Footer>
		{/if}
	</Dialog.Content>
</Dialog.Root>