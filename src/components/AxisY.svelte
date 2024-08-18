<script>
	export let yScale;
	export let groupBySeason;

	$: ticks = yScale.domain()

	import { fade } from "svelte/transition"
	import { fly } from "svelte/transition"

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