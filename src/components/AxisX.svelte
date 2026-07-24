<script>
  /** @type {{xScale: any, gridTop: number, gridBottom: number, seriesAverage: number}} */
  let { xScale, gridTop, gridBottom, seriesAverage } = $props();

  let ticks = $derived(xScale.ticks(4));
</script>

<g class="axis x">
  {#each ticks as tick}
    <g class="tick" transform="translate({xScale(tick)}, 0)">
      <line x1="0" x2="0" y1={gridTop} y2={gridBottom} stroke="hsla(215, 15%, 91%, 1)" />
      <text x="3" y={gridBottom + 17}>{tick} stars</text>
    </g>
  {/each}
  <g class="average" transform="translate({xScale(seriesAverage)}, 0)">
    <line x1="0" x2="0" y1={gridTop} y2={gridBottom} />
    <text x="5" y={gridTop - 8}>Series average · {seriesAverage}</text>
  </g>
</g>

<style>
  .average line {
    stroke: oklch(0.48 0 0);
    stroke-width: 1;
    stroke-dasharray: 3 4;
  }

  .average text {
    fill: oklch(0.4 0 0);
    font-weight: 600;
  }
</style>
