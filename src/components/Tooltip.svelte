<script>
	/** @type {{data: any, x: number, y: number, chartWidth: number, chartHeight: number}} */
	let { data, x, y, chartWidth, chartHeight } = $props();

	let tooltipWidth = $state(220);
	let tooltipHeight = $state(56);
	const edgeGap = 8;
	const pointGap = 12;

	let xPosition = $derived(
		Math.max(
			edgeGap,
			Math.min(x + pointGap, chartWidth - tooltipWidth - edgeGap),
		),
	);
	let yPosition = $derived(
		y - tooltipHeight - pointGap > edgeGap
			? y - tooltipHeight - pointGap
			: Math.min(y + pointGap, chartHeight - tooltipHeight - edgeGap),
	);

</script>

<div
	class="tooltip"
	role="status"
	aria-live="polite"
	style="left:{xPosition}px; top:{yPosition}px;"
	bind:clientWidth={tooltipWidth}
	bind:clientHeight={tooltipHeight}
>
	<h1>S{data.season} E{data.episode}: {data.title}</h1>
	<div class="rating">
		{data.averageEpisodeRating}/10 stars
	</div>
</div>

<style>
	.tooltip {
		position: absolute;
		z-index: 1;
		width: max-content;
		max-width: min(220px, calc(100% - 16px));
		background: #fff;
		padding: 0.5rem 0.625rem;
		border: 1px solid #d7d7d7;
		border-radius: 0.375rem;
		box-shadow: 0 3px 10px rgb(0 0 0 / 14%);
		color: #222;
		pointer-events: none;
	}

	h1 {
		margin: 0;
		font-size: 0.75rem;
		font-weight: 600;
		line-height: 1.3;
	}

	.rating {
		margin-top: 0.125rem;
		font-size: 0.75rem;
		line-height: 1.3;
	}
</style>
