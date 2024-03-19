<script lang="ts">
	import App from '$lib/components/App.svelte';
</script>

<div>
	<a href="https://github.com/edsunman/threlte-particles" target="_blank">github</a>
	<App />
</div>

<style>
	:global(body) {
		margin: 0;
	}

	div {
		width: 100vw;
		height: 100vh;
		background: rgb(13, 19, 32);
		background: linear-gradient(180deg, rgba(13, 19, 32, 1) 0%, rgba(8, 12, 21, 1) 100%);
	}

	a {
		position: absolute;
		color: white;
		text-decoration: none;
		margin: 30px;
		font-family: monospace;
	}
</style>
