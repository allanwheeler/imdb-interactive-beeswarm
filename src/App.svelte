<script>
  import { onDestroy, onMount, untrack } from 'svelte';
  import { cubicOut } from 'svelte/easing';

  import { forceSimulation, forceX, forceY, forceCollide } from 'd3-force';
  import { scaleLinear, scaleBand } from 'd3-scale';
  import data from './data/data.js';
  import AxisX from './components/AxisX.svelte';
  import AxisY from './components/AxisY.svelte';
  import Tooltip from './components/Tooltip.svelte';

  let width = $state(400);
  let height = 600;

  const margin = { top: 0, right: 50, left: 30, bottom: 20 };

  /* ─────────────────────────────────────────────────────────
   * ANIMATION STORYBOARD
   *
   *    0ms   toggle selects a precomputed stable layout
   *  300ms   dots arrive at their new positions with cubic ease-out
   * ───────────────────────────────────────────────────────── */
  const MOTION = {
    duration: 300,
    layoutTicks: 300,
    collisionRadius: 7,
  };

  let innerWidth = $derived(width - margin.left - margin.right);
  let innerHeight = height - margin.top - margin.bottom;

  let seasons = Array.from(new Set(data.map(d => d.season)));

  let xScale = $derived(scaleLinear().domain([6.5, 10]).range([0, innerWidth]));
  let yScale = $derived(
    scaleBand().domain(seasons).range([0, innerHeight]).paddingOuter(0.5),
  );

  let nodes = $state([]);
  let groupBySeason = $state(false);
  let layouts = $state();
  let hoveredIndex = $state();
  let animationFrame;
  let prefersReducedMotion = false;

  function calculateLayout(grouped, currentXScale, currentYScale) {
    const layoutNodes = data.map((datum, index) => ({ ...datum, layoutIndex: index }));

    forceSimulation(layoutNodes)
      .force(
        'x',
        forceX()
          .x(d => currentXScale(d.averageEpisodeRating))
          .strength(1),
      )
      .force(
        'y',
        forceY()
          .y(d => (grouped ? currentYScale(d.season) : innerHeight / 2))
          .strength(0.5),
      )
      .force('collide', forceCollide().radius(MOTION.collisionRadius))
      .stop()
      .tick(MOTION.layoutTicks);

    return layoutNodes;
  }

  function animateTo(targetNodes) {
    cancelAnimationFrame(animationFrame);

    if (nodes.length === 0 || prefersReducedMotion) {
      nodes = targetNodes;
      return;
    }

    const startNodes = nodes.map(node => ({ x: node.x, y: node.y }));
    const startTime = performance.now();

    function frame(time) {
      const progress = Math.min((time - startTime) / MOTION.duration, 1);
      const easedProgress = cubicOut(progress);

      nodes = targetNodes.map((target, index) => ({
        ...target,
        x: startNodes[index].x + (target.x - startNodes[index].x) * easedProgress,
        y: startNodes[index].y + (target.y - startNodes[index].y) * easedProgress,
      }));

      if (progress < 1) animationFrame = requestAnimationFrame(frame);
    }

    animationFrame = requestAnimationFrame(frame);
  }

  $effect(() => {
    const currentXScale = xScale;
    const currentYScale = yScale;

    layouts = {
      combined: calculateLayout(false, currentXScale, currentYScale),
      grouped: calculateLayout(true, currentXScale, currentYScale),
    };
  });

  $effect(() => {
    const targetNodes = groupBySeason ? layouts?.grouped : layouts?.combined;
    if (targetNodes) untrack(() => animateTo(targetNodes));
  });

  onMount(() => {
    const motionPreference = window.matchMedia('(prefers-reduced-motion: reduce)');
    const updateMotionPreference = () => {
      prefersReducedMotion = motionPreference.matches;
    };

    updateMotionPreference();
    motionPreference.addEventListener('change', updateMotionPreference);
    return () => motionPreference.removeEventListener('change', updateMotionPreference);
  });

  onDestroy(() => cancelAnimationFrame(animationFrame));
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
      {#each nodes as node, i (node.layoutIndex)}
        <circle
          role="presentation"
          aria-label="Data point description"
          cx={node.x}
          cy={node.y}
          r={5}
          fill={hoveredIndex === i ? 'orange' : '#f4f4f4'}
          stroke={'#555'}
          stroke-width={0.5}
          onmouseover={() => (hoveredIndex = i)}
          onfocus={() => (hoveredIndex = i)}
          onclick={event => {
            event.stopImmediatePropagation();
          }}
          onmouseleave={() => (hoveredIndex = null)}
        />
      {/each}
    </g>
  </svg>

  {#if hoveredIndex !== undefined && hoveredIndex !== null}
    <Tooltip data={nodes[hoveredIndex]} />
  {/if}
  <p class="source">Source: IMDb</p>
</div>

<style>
  .chart-container {
    max-width: 800px;
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
      stroke 100ms ease-out,
      opacity 100ms ease-out;
    cursor: pointer;
  }
</style>
