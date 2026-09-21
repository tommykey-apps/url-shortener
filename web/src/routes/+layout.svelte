<script lang="ts">
	import '../app.css';

	let { children } = $props();
</script>

<svelte:head>
	<title>URL 短縮</title>
</svelte:head>

<div class="min-h-screen">
	<header class="site-header">
		<div class="mx-auto max-w-6xl px-4">
			<a href="/" class="site-title dads-u-std-20B-150">URL 短縮</a>
		</div>
	</header>
	<main class="mx-auto max-w-6xl px-4 py-8">
		{@render children()}
	</main>
</div>

<style>
	.site-header {
		border-bottom: 1px solid var(--color-neutral-solid-gray-200);
		padding-top: 1rem;
		padding-bottom: 1rem;
	}

	.site-title {
		color: var(--color-neutral-solid-gray-900);
		text-decoration: none;
	}

	.site-title:hover {
		text-decoration: underline;
	}
</style>
