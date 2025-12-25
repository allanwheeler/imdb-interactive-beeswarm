<script>


	import { fade } from "svelte/transition"
	import { fly } from "svelte/transition"
	/** @type {{yScale: any, groupBySeason: any}} */
	let { yScale, groupBySeason } = $props();

	let ticks = $derived(yScale.domain())
</script>

<g class="axis-y">
	{#each ticks as tick, index} 
		{#if groupBySeason}
			<g class="tick" in:fly={{x: -100, duration: 1000}} out:fly={{x: -100, duration: 1000}}> 
				<text y={yScale(tick)}>Season {tick}</text>
			</g>
		{/if}
	{/each}
</g>


<style>
	.tick {
		text-anchor: middle;
	}
</style>