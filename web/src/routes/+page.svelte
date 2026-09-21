<script lang="ts">
	import { onMount } from 'svelte';
	import {
		shortenUrl,
		listUrls,
		deleteUrl,
		summarizeUrl,
		getClickStats,
		type URL as ShortURL,
		type ShortenResponse,
		type ClickStats
	} from '$lib/api';

	// ---- 短縮フォーム ----
	let newUrl = $state('');
	let submitting = $state(false);
	let formError = $state('');
	let result = $state<ShortenResponse | null>(null);
	let copied = $state(false);

	// ---- 一覧 ----
	type ListState =
		| { kind: 'loading' }
		| { kind: 'error'; message: string }
		| { kind: 'ready'; items: ShortURL[] };
	let list = $state<ListState>({ kind: 'loading' });
	let listNotice = $state<{ type: 'error' | 'success'; message: string } | null>(null);

	type SortKey = 'code' | 'original_url' | 'clicks' | 'created_at';
	type SortDir = 'ascending' | 'descending';
	let filter = $state('');
	let sortKey = $state<SortKey>('created_at');
	let sortDir = $state<SortDir>('descending');
	let page = $state(1);
	const PAGE_SIZE = 20;

	// ---- 行内の詳細 (統計・AI 要約) と削除確認 ----
	type Detail =
		| { code: string; kind: 'stats'; status: 'loading' }
		| { code: string; kind: 'stats'; status: 'ready'; stats: ClickStats }
		| { code: string; kind: 'summary'; status: 'loading' }
		| { code: string; kind: 'summary'; status: 'ready'; summary: string }
		| { code: string; kind: 'stats' | 'summary'; status: 'error'; message: string };
	let detail = $state<Detail | null>(null);
	let confirmingDelete = $state<string | null>(null);
	let deleting = $state(false);

	const SORT_ICONS: Record<SortDir | 'none', string> = {
		none: 'M17 18.11L21.27 14L22 14.7L16.5 20L11 14.7L11.73 14L16 18.12V4H17V18.12ZM8 5.88L12.27 10L13 9.3L7.5 4L2 9.3L2.73 10L7 5.88V20H8V5.88Z',
		ascending:
			'M17 18.12L21.27 14L22 14.7L16.5 20L11 14.7L11.73 14L16 18.12V4H17V18.12ZM14 8.92L11.73 11L9 8.52V20H6V8.52L3.27 11L1 8.93L7.5 3L14 8.93Z',
		descending:
			'M7 5.88L2.73 10L2 9.3L7.5 4L13 9.3L12.27 10L8 5.88V20H7V5.88ZM10 15.08L12.27 13L15 15.48V4H18V15.48L20.73 13L23 15.07L16.5 21L10 15.07Z'
	};

	const COLUMNS: { key: SortKey; label: string }[] = [
		{ key: 'code', label: 'コード' },
		{ key: 'original_url', label: '元の URL' },
		{ key: 'clicks', label: 'クリック数' },
		{ key: 'created_at', label: '作成日' }
	];

	// このブラウザで作成したコードだけを一覧に出す (API は全件を返す)
	function getMyCodes(): string[] {
		try {
			const raw = localStorage.getItem('my_urls');
			return raw ? JSON.parse(raw) : [];
		} catch {
			return [];
		}
	}

	function addMyCode(code: string) {
		const codes = getMyCodes();
		codes.push(code);
		localStorage.setItem('my_urls', JSON.stringify(codes));
	}

	function removeMyCode(code: string) {
		const codes = getMyCodes().filter((c) => c !== code);
		localStorage.setItem('my_urls', JSON.stringify(codes));
	}

	async function loadUrls() {
		list = { kind: 'loading' };
		try {
			const all = (await listUrls()) ?? [];
			const myCodes = getMyCodes();
			list = { kind: 'ready', items: all.filter((u) => myCodes.includes(u.code)) };
		} catch (err) {
			list = { kind: 'error', message: err instanceof Error ? err.message : String(err) };
		}
	}

	async function handleSubmit(e: SubmitEvent) {
		e.preventDefault();
		if (!newUrl.trim()) return;

		submitting = true;
		formError = '';
		result = null;
		copied = false;

		try {
			result = await shortenUrl(newUrl);
			addMyCode(result.code);
			newUrl = '';
			await loadUrls();
		} catch (err) {
			formError = err instanceof Error ? err.message : String(err);
		} finally {
			submitting = false;
		}
	}

	async function copyToClipboard(text: string) {
		await navigator.clipboard.writeText(text);
		copied = true;
		setTimeout(() => (copied = false), 2000);
	}

	async function handleDelete(code: string) {
		deleting = true;
		listNotice = null;
		try {
			await deleteUrl(code);
			removeMyCode(code);
			if (detail?.code === code) detail = null;
			confirmingDelete = null;
			listNotice = { type: 'success', message: `コード ${code} を削除しました` };
			await loadUrls();
		} catch {
			listNotice = { type: 'error', message: `コード ${code} を削除できませんでした` };
		} finally {
			deleting = false;
		}
	}

	async function showStats(code: string) {
		if (detail?.code === code && detail.kind === 'stats') {
			detail = null;
			return;
		}
		detail = { code, kind: 'stats', status: 'loading' };
		try {
			const stats = await getClickStats(code);
			detail = { code, kind: 'stats', status: 'ready', stats };
		} catch {
			detail = { code, kind: 'stats', status: 'error', message: '統計を取得できませんでした' };
		}
	}

	async function showSummary(code: string) {
		if (detail?.code === code && detail.kind === 'summary') {
			detail = null;
			return;
		}
		detail = { code, kind: 'summary', status: 'loading' };
		try {
			const summary = await summarizeUrl(code);
			detail = { code, kind: 'summary', status: 'ready', summary };
		} catch {
			detail = { code, kind: 'summary', status: 'error', message: '要約を取得できませんでした' };
		}
	}

	function toggleSort(key: SortKey) {
		if (sortKey === key) {
			sortDir = sortDir === 'ascending' ? 'descending' : 'ascending';
		} else {
			sortKey = key;
			sortDir = key === 'created_at' || key === 'clicks' ? 'descending' : 'ascending';
		}
	}

	const filtered = $derived.by(() => {
		if (list.kind !== 'ready') return [];
		const q = filter.trim().toLowerCase();
		const items = q
			? list.items.filter(
					(u) => u.code.toLowerCase().includes(q) || u.original_url.toLowerCase().includes(q)
				)
			: list.items;
		const dir = sortDir === 'ascending' ? 1 : -1;
		return [...items].sort((a, b) => {
			const av = a[sortKey];
			const bv = b[sortKey];
			if (typeof av === 'number' && typeof bv === 'number') return (av - bv) * dir;
			return String(av).localeCompare(String(bv), 'ja') * dir;
		});
	});
	const totalPages = $derived(Math.max(1, Math.ceil(filtered.length / PAGE_SIZE)));
	const pageItems = $derived(filtered.slice((page - 1) * PAGE_SIZE, page * PAGE_SIZE));

	$effect(() => {
		// 絞り込みを変えたら 1 ページ目に戻す
		void filter;
		page = 1;
	});
	$effect(() => {
		if (page > totalPages) page = totalPages;
	});

	function formatDate(iso: string): string {
		return new Date(iso).toLocaleDateString('ja-JP', { year: 'numeric', month: 'long', day: 'numeric' });
	}

	onMount(async () => {
		// モバイルで表を横スクロールさせる公式部品 (customElements は二重定義できないので確認してから読み込む)
		if (!customElements.get('dads-scroll-shadow')) {
			await import('$lib/dads/scroll-shadow.js');
		}
		loadUrls();
	});
</script>

<section aria-labelledby="shorten-heading" class="mb-12">
	<h1 id="shorten-heading" class="dads-u-std-28B-150 mb-6">URL を短縮する</h1>

	<form onsubmit={handleSubmit} novalidate>
		<div class="dads-form-control-label" data-size="md">
			<label class="dads-form-control-label__label" for="url-input">
				短縮したい URL
				<span class="dads-form-control-label__requirement" data-required="true">※必須</span>
			</label>
			<p id="url-support" class="dads-form-control-label__support-text">
				http:// または https:// で始まる URL を入力してください。登録時に安全性を確認します。
			</p>
			<div class="flex flex-col gap-4 sm:flex-row sm:items-start">
				<span class="dads-input-text grow">
					<input
						id="url-input"
						class="dads-input-text__input w-full"
						type="url"
						name="url"
						data-size="lg"
						bind:value={newUrl}
						aria-required="true"
						autocomplete="off"
						aria-invalid={formError ? 'true' : undefined}
						aria-describedby={formError ? 'url-error url-support' : 'url-support'}
					/>
					{#if formError}
						<span id="url-error" class="dads-input-text__error-text" role="alert">{formError}</span>
					{/if}
				</span>
				<button
					class="dads-button shrink-0"
					type="submit"
					data-type="solid-fill"
					data-size="lg"
					disabled={submitting}
				>
					{submitting ? '作成中' : '短縮する'}
				</button>
			</div>
		</div>
	</form>

	{#if result}
		<div class="dads-notification-banner mt-6" data-style="standard" data-type="success" role="status">
			<h2 class="dads-notification-banner__heading">
				<svg class="dads-notification-banner__icon" width="24" height="24" viewBox="0 0 24 24" role="img" aria-label="成功">
					<circle cx="12" cy="12" r="10" fill="currentcolor" />
					<path d="m17.6 9.6-7 7-4.3-4.3L7.7 11l2.9 2.9 5.7-5.6 1.3 1.4Z" fill="Canvas" />
				</svg>
				<span class="dads-notification-banner__heading-text">短縮 URL を作成しました</span>
			</h2>
			<button class="dads-notification-banner__close" type="button" onclick={() => (result = null)}>
				<svg class="dads-notification-banner__close-icon" width="24" height="24" viewBox="0 0 24 24" aria-hidden="true">
					<path d="m6.4 18.6-1-1 5.5-5.6-5.6-5.6 1.1-1 5.6 5.5 5.6-5.6 1 1.1L13 12l5.6 5.6-1 1L12 13l-5.6 5.6Z" fill="currentcolor" />
				</svg>
				<span class="dads-notification-banner__close-label">閉じる</span>
			</button>
			<div class="dads-notification-banner__body">
				<p>
					<a class="dads-link break-all" href={result.short_url} target="_blank" rel="noopener">
						{result.short_url}
						<svg class="dads-link__icon" viewBox="0 0 48 48" role="img" aria-label="新規タブで開きます">
							<path d="M22 6V9H9V39H39V26H42V42H6V6H22ZM42 6V20H39V11.2L21 29L19 27L36.8 9H28V6H42Z" fill="currentcolor" />
						</svg>
					</a>
				</p>
			</div>
			<div class="dads-notification-banner__actions">
				<button
					class="dads-button"
					type="button"
					data-type="outline"
					data-size="md"
					onclick={() => copyToClipboard(result!.short_url)}
				>
					{copied ? 'コピーしました' : 'コピー'}
				</button>
			</div>
		</div>
	{/if}
</section>

<section aria-labelledby="list-heading">
	<h2 id="list-heading" class="dads-u-std-24B-150 mb-4">作成した URL</h2>

	{#if listNotice}
		<div
			class="dads-notification-banner mb-6"
			data-style="color-chip"
			data-type={listNotice.type}
			role={listNotice.type === 'error' ? 'alert' : 'status'}
		>
			<h3 class="dads-notification-banner__heading">
				{#if listNotice.type === 'error'}
					<svg class="dads-notification-banner__icon" width="24" height="24" viewBox="0 0 24 24" role="img" aria-label="エラー">
						<path d="M8.25 21 3 15.75v-7.5L8.25 3h7.5L21 8.25v7.5L15.75 21h-7.5Z" fill="currentcolor" />
						<path d="m12 13.4-2.85 2.85-1.4-1.4L10.6 12 7.75 9.15l1.4-1.4L12 10.6l2.85-2.85 1.4 1.4L13.4 12l2.85 2.85-1.4 1.4L12 13.4Z" fill="Canvas" />
					</svg>
				{:else}
					<svg class="dads-notification-banner__icon" width="24" height="24" viewBox="0 0 24 24" role="img" aria-label="成功">
						<circle cx="12" cy="12" r="10" fill="currentcolor" />
						<path d="m17.6 9.6-7 7-4.3-4.3L7.7 11l2.9 2.9 5.7-5.6 1.3 1.4Z" fill="Canvas" />
					</svg>
				{/if}
				<span class="dads-notification-banner__heading-text">{listNotice.message}</span>
			</h3>
			<button class="dads-notification-banner__close" type="button" onclick={() => (listNotice = null)}>
				<svg class="dads-notification-banner__close-icon" width="24" height="24" viewBox="0 0 24 24" aria-hidden="true">
					<path d="m6.4 18.6-1-1 5.5-5.6-5.6-5.6 1.1-1 5.6 5.5 5.6-5.6 1 1.1L13 12l5.6 5.6-1 1L12 13l-5.6 5.6Z" fill="currentcolor" />
				</svg>
				<span class="dads-notification-banner__close-label">閉じる</span>
			</button>
		</div>
	{/if}

	{#if list.kind === 'loading'}
		<p role="status">一覧を読み込んでいます。</p>
	{:else if list.kind === 'error'}
		<div class="dads-notification-banner" data-style="standard" data-type="error" role="alert">
			<h3 class="dads-notification-banner__heading">
				<svg class="dads-notification-banner__icon" width="24" height="24" viewBox="0 0 24 24" role="img" aria-label="エラー">
					<path d="M8.25 21 3 15.75v-7.5L8.25 3h7.5L21 8.25v7.5L15.75 21h-7.5Z" fill="currentcolor" />
					<path d="m12 13.4-2.85 2.85-1.4-1.4L10.6 12 7.75 9.15l1.4-1.4L12 10.6l2.85-2.85 1.4 1.4L13.4 12l2.85 2.85-1.4 1.4L12 13.4Z" fill="Canvas" />
				</svg>
				<span class="dads-notification-banner__heading-text">一覧を取得できませんでした</span>
			</h3>
			<div class="dads-notification-banner__body">
				<p>{list.message}</p>
			</div>
			<div class="dads-notification-banner__actions">
				<button class="dads-button" type="button" data-type="outline" data-size="md" onclick={loadUrls}>
					再読み込み
				</button>
			</div>
		</div>
	{:else if list.items.length === 0}
		<p>まだ URL がありません。上の入力欄に URL を入れて「短縮する」を押すと、ここに一覧が表示されます。</p>
	{:else}
		<div class="mb-4 flex flex-col gap-4 sm:flex-row sm:items-end sm:justify-between">
			<div class="dads-form-control-label" data-size="sm">
				<label class="dads-form-control-label__label" for="filter-input">絞り込み</label>
				<p id="filter-support" class="dads-form-control-label__support-text">コードまたは元の URL の一部</p>
				<span class="dads-input-text">
					<input
						id="filter-input"
						class="dads-input-text__input filter-input"
						type="search"
						data-size="sm"
						bind:value={filter}
						aria-describedby="filter-support"
					/>
				</span>
			</div>
			<p class="count" aria-live="polite">
				{#if filter.trim()}
					全 {list.items.length} 件中 {filtered.length} 件
				{:else}
					全 {list.items.length} 件
				{/if}
			</p>
		</div>

		{#if filtered.length === 0}
			<p>絞り込み条件に一致する URL はありません。条件を変えるか、絞り込みを空にしてください。</p>
		{:else}
			<dads-scroll-shadow style="--scroll-shadow-padding: 1rem;">
				<div class="dads-table w-full" data-size="dense">
					<table class="dads-table__table url-table" data-width="full" data-cell-border="bottom">
						<colgroup>
							<col class="col-code" />
							<col />
							<col class="col-clicks" />
							<col class="col-date" />
							<col class="col-actions" />
						</colgroup>
						<thead>
							<tr>
								{#each COLUMNS as col (col.key)}
									<th
										class="dads-table__sort-header"
										scope="col"
										aria-sort={sortKey === col.key ? sortDir : 'none'}
									>
										<div class="dads-table__sort-inner">
											<div class="dads-table__sort-label">
												<button class="dads-table__sort-button" type="button" onclick={() => toggleSort(col.key)}>
													{col.label}
													<span class="dads-table__sort-icon">
														<svg class="dads-table__sort-svg" width="24" height="24" fill="currentcolor" aria-hidden="true">
															<path d={SORT_ICONS[sortKey === col.key ? sortDir : 'none']} />
														</svg>
													</span>
												</button>
											</div>
										</div>
									</th>
								{/each}
								<th class="dads-table__col-header" scope="col">操作</th>
							</tr>
						</thead>
						<tbody>
							{#each pageItems as url (url.code)}
								<tr>
									<td>
										<a class="dads-link font-mono" href="/r/{url.code}" target="_blank" rel="noopener">
											{url.code}
											<svg class="dads-link__icon" viewBox="0 0 48 48" role="img" aria-label="新規タブで開きます">
												<path d="M22 6V9H9V39H39V26H42V42H6V6H22ZM42 6V20H39V11.2L21 29L19 27L36.8 9H28V6H42Z" fill="currentcolor" />
											</svg>
										</a>
									</td>
									<td class="url-cell">{url.original_url}</td>
									<td class="num-cell">{url.clicks.toLocaleString('ja-JP')}</td>
									<td><time datetime={url.created_at}>{formatDate(url.created_at)}</time></td>
									<td>
										{#if confirmingDelete === url.code}
											<div class="flex items-center gap-2">
												<span>削除しますか？</span>
												<button
													class="dads-button danger-button"
													type="button"
													data-type="solid-fill"
													data-size="sm"
													disabled={deleting}
													onclick={() => handleDelete(url.code)}
												>
													削除する
												</button>
												<button
													class="dads-button"
													type="button"
													data-type="text"
													data-size="sm"
													disabled={deleting}
													onclick={() => (confirmingDelete = null)}
												>
													キャンセル
												</button>
											</div>
										{:else}
											<div class="flex items-center gap-1">
												<button
													class="dads-button"
													type="button"
													data-type="text"
													data-size="sm"
													aria-expanded={detail?.code === url.code && detail.kind === 'stats'}
													onclick={() => showStats(url.code)}
												>
													統計
												</button>
												<button
													class="dads-button"
													type="button"
													data-type="text"
													data-size="sm"
													aria-expanded={detail?.code === url.code && detail.kind === 'summary'}
													onclick={() => showSummary(url.code)}
												>
													AI 要約
												</button>
												<button
													class="dads-button danger-button"
													type="button"
													data-type="text"
													data-size="sm"
													onclick={() => (confirmingDelete = url.code)}
												>
													削除
												</button>
											</div>
										{/if}
									</td>
								</tr>
								{#if detail?.code === url.code}
									<tr class="detail-row">
										<td colspan="5">
											{#if detail.status === 'loading'}
												<p role="status">
													{detail.kind === 'stats' ? '統計を取得しています。' : 'AI が要約しています。'}
												</p>
											{:else if detail.status === 'error'}
												<p role="alert" class="error-text">{detail.message}</p>
											{:else if detail.kind === 'stats'}
												<h3 class="dads-u-std-17B-170 mb-2">クリック統計 (コード {detail.stats.code})</h3>
												<p class="mb-4">過去 30 日の合計: {detail.stats.total_clicks.toLocaleString('ja-JP')} 回</p>
												{#if detail.stats.daily.length === 0}
													<p>過去 30 日にクリックはありません。</p>
												{:else}
													<div class="dads-table" data-size="dense">
														<table class="dads-table__table" data-cell-border="bottom">
															<thead>
																<tr>
																	<th class="dads-table__col-header" scope="col">日付</th>
																	<th class="dads-table__col-header num-cell" scope="col">クリック数</th>
																</tr>
															</thead>
															<tbody>
																{#each detail.stats.daily as d (d.date)}
																	<tr>
																		<td><time datetime={d.date}>{d.date}</time></td>
																		<td class="num-cell">{d.clicks.toLocaleString('ja-JP')}</td>
																	</tr>
																{/each}
															</tbody>
														</table>
													</div>
												{/if}
											{:else}
												<h3 class="dads-u-std-17B-170 mb-2">AI 要約 (コード {detail.code})</h3>
												<p class="summary-text">{detail.summary}</p>
											{/if}
										</td>
									</tr>
								{/if}
							{/each}
						</tbody>
					</table>
				</div>
			</dads-scroll-shadow>

			{#if totalPages > 1}
				<nav class="dads-page-navigation mt-6 justify-center" aria-label="ページ">
					<button
						class="dads-button"
						type="button"
						data-type="outline"
						data-size="md"
						data-control="prev"
						disabled={page <= 1}
						onclick={() => (page -= 1)}
					>
						<svg class="dads-button__icon" width="24" height="24" viewBox="0 0 24 24" aria-hidden="true">
							<path d="m7.9 12 8-8-1.4-1.4L5.1 12l9.4 9.4 1.4-1.4z" fill="currentcolor" />
						</svg>
						前のページ
					</button>
					<span class="dads-page-navigation__counter">{page} / {totalPages}</span>
					<button
						class="dads-button"
						type="button"
						data-type="outline"
						data-size="md"
						data-control="next"
						disabled={page >= totalPages}
						onclick={() => (page += 1)}
					>
						次のページ
						<svg class="dads-button__icon" width="24" height="24" viewBox="0 0 24 24" aria-hidden="true">
							<path d="M9 2.6 7.6 4l8 8-8 8L9 21.4l9.4-9.4z" fill="currentcolor" />
						</svg>
					</button>
				</nav>
			{/if}
		{/if}
	{/if}
</section>

<style>
	.filter-input {
		width: 20rem;
		max-width: 100%;
	}

	.count {
		color: var(--color-neutral-solid-gray-600);
	}

	.url-table {
		min-width: 60rem;
	}

	.url-table :is(th, td) {
		white-space: nowrap;
	}

	.url-table .url-cell,
	.url-table .detail-row > td {
		white-space: normal;
	}

	.col-code {
		width: 8rem;
	}

	.col-clicks {
		width: 7rem;
	}

	.col-date {
		width: 10rem;
	}

	.col-actions {
		width: 19rem;
	}

	.url-cell {
		overflow-wrap: anywhere;
	}

	.num-cell {
		text-align: right;
		font-variant-numeric: tabular-nums;
	}

	/* 破壊的操作は意味色 (error) で分ける。公式部品と同じ変数を上書きする */
	.danger-button {
		--button-color: var(--color-semantic-error-1);
		--button-hover-color: var(--color-primitive-red-1000);
		--button-active-color: var(--color-primitive-red-1200);
		--button-outline-hover-bg-color: var(--color-primitive-red-200);
		--button-outline-active-bg-color: var(--color-primitive-red-300);
	}

	@media (hover: hover) {
		.danger-button[data-type='text']:hover {
			background-color: var(--color-primitive-red-50);
		}
	}

	.danger-button[data-type='text']:active {
		background-color: var(--color-primitive-red-100);
	}

	.detail-row > td {
		background-color: var(--color-neutral-solid-gray-50);
	}

	.error-text {
		color: var(--color-semantic-error-1);
	}

	.summary-text {
		white-space: pre-wrap;
	}
</style>
