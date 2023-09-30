<script lang="ts">
	// import Header from '$lib/header/Header.svelte';
	// import '../app.css';
	import "carbon-components-svelte/css/all.css";
	import { Theme } from "carbon-components-svelte";
	let theme = "g90";
</script>
<Theme bind:theme />
<main>
	<slot />
</main>
