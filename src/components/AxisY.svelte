<script>


	import { fly } from "svelte/transition"
	import { cubicOut } from "svelte/easing"
	/** @type {{yScale: any, groupBySeason: any}} */
	let { yScale, groupBySeason } = $props();

	let ticks = $derived(yScale.domain())
</script>

<g class="axis-y">
	{#each ticks as tick, index} 
		{#if groupBySeason}
			<g
				class="tick"
				in:fly={{ x: -100, duration: 300, easing: cubicOut }}
				out:fly={{ x: -100, duration: 300, easing: cubicOut }}
			>
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
