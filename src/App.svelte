<script>
  import { run } from 'svelte/legacy';

  import { forceSimulation, forceX, forceY, forceCollide } from 'd3-force';
  import { scaleLinear, scaleBand, scaleOrdinal, scaleSqrt } from 'd3-scale';
  import data from './data/data.js';
  import AxisX from './components/AxisX.svelte';
  import AxisY from './components/AxisY.svelte';
  import Tooltip from './components/Tooltip.svelte';

  let width = $state(400);
  let height = 600;

  const margin = { top: 0, right: 50, left: 30, bottom: 20 };

  let innerWidth = $derived(width - margin.left - margin.right);
  let innerHeight = height - margin.top - margin.bottom;

  let seasons = Array.from(new Set(data.map(d => d.season)));

  let xScale = $derived(scaleLinear().domain([6.5, 10]).range([0, innerWidth]));
  let yScale = $derived(
    scaleBand().domain(seasons).range([0, innerHeight]).paddingOuter(0.5)
  );

  let simulation = forceSimulation(data);
  let nodes = $state([]);
  let tickCount = $state(0);
  const maxTicks = 200; // Set this to your desired number of ticks

  simulation.on('tick', () => {
    nodes = simulation.nodes();
    tickCount++;
    if (tickCount >= maxTicks) {
      simulation.stop();
    }
  });

  let groupBySeason = $state(false);

  run(() => {
    tickCount = 0; // Reset tick count when forces change
    simulation
      .force(
        'x',
        forceX()
          .x(d => xScale(d.averageEpisodeRating))
          .strength(1)
      )
      .force(
        'y',
        forceY()
          .y(d => (groupBySeason ? yScale(d.season) : innerHeight / 2))
          .strength(groupBySeason ? 0.5 : 0.15)
      )
      .force('collide', forceCollide().radius(7))
      .alpha(0.1)
      .alphaDecay(0.00000000002)
      .restart();
  });

  let hovered = $state();
</script>

<h2>Average episode ratings for the series Frasier</h2>
<p style="margin: 0; font-size: 0.875rem;">
  Click on plot to view {!groupBySeason ? 'by season' : 'all episodes'}
</p>
<!-- svelte-ignore a11y_click_events_have_key_events -->
<div
  role="button"
  tabindex="0"
  class="chart-container"
  bind:clientWidth={width}
  onclick={() => (groupBySeason = !groupBySeason)}
>
  <svg {width} {height}>
    <g class="inner-chart" transform="translate({margin.left}, {margin.top})">
      <AxisY {yScale} {groupBySeason} />
      <AxisX {xScale} height={innerHeight} width={innerWidth} />
      {#each nodes as node, i}
        <circle
          role="presentation"
          aria-label="Data point description"
          cx={node.x}
          cy={node.y}
          r={5}
          fill={hovered === node ? 'orange' : '#f4f4f4'}
          stroke={'#555'}
          stroke-width={0.5}
          onmouseover={() => (hovered = node)}
          onfocus={() => (hovered = node)}
          onclick={event => {
            event.stopImmediatePropagation();
          }}
          onmouseleave={() => (hovered = null)}
        />
      {/each}
    </g>
  </svg>

  {#if hovered}
    <Tooltip data={hovered} />
  {/if}
  <p class="source">Source: IMDb</p>
</div>

<style>
  .chart-container {
    /* max-width: 1000px; */
    max-height: 1000px;
    font-size: 0.7rem;
    position: relative;
    height: 95vh;
    user-select: none;
    /* For Safari */
    -webkit-user-select: none;
    /* For Firefox */
    -moz-user-select: none;
  }

  h2 {
    margin-bottom: 0.5rem;
    font-size: 1.5rem;
    font-weight: 600;
  }

  circle {
    transition:
      stroke 300ms ease,
      opacity 300ms ease;
    cursor: pointer;
  }
</style>
