<script>
  /** @type {{xScale: any, gridTop: number, gridBottom: number}} */
  let { xScale, gridTop, gridBottom } = $props();

  let ticks = $derived(xScale.ticks(4));
</script>

<g class="axis x">
  {#each ticks as tick}
    <g class="tick" transform="translate({xScale(tick)}, 0)">
      <text x="3" y={gridBottom}>{tick} stars</text>
      <line x1="0" x2="0" y1={gridTop} y2={gridBottom} stroke="hsla(215, 15%, 91%, 1)" />
    </g>
  {/each}
</g>
