<script>
	export let data;
	// export let width;

	let tooltipWidth;

	$: xPosition = data.x + 40
	$: yPosition = data.y - 10

</script>

<div
	class="tooltip"
	style="left:{xPosition}px; top:{yPosition}px;"
	bind:clientWidth={tooltipWidth}
>
	<h1>S{data.season} E{data.episode}: {data.title}</h1>
	<div
		style="font-size: 0.75"
	>
		{data.averageEpisodeRating}/10 stars
	</div>
</div>

<style>
	.tooltip {
    position: absolute;
		background: #fff;
		padding: .3rem;
		opacity: 0.75;
	}

	 h1 {
    margin: 0;
    font-size: 0.75rem;
    font-weight: 600;
		/* stroke: white; */
		/* stroke-width: 2px; */
  }
</style>