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
  let viewportHeight = $state(0);
  let height = $derived(
    width < 600 && viewportHeight
      ? Math.max(420, Math.round(viewportHeight * 0.88))
      : Math.min(600, Math.max(420, Math.round(width * 0.75))),
  );
  let pointRadius = $derived(width < 480 ? 4 : 5);
  let collisionRadius = $derived(pointRadius + 2);

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
    gridPadding: 32,
  };
  const INTERACTION = {
    hitRadius: 24,
  };

  let innerWidth = $derived(width - margin.left - margin.right);
  let innerHeight = $derived(height - margin.top - margin.bottom);

  let seasons = Array.from(new Set(data.map(d => d.season)));

  let xScale = $derived(scaleLinear().domain([6.5, 10]).range([0, innerWidth]));
  let yScale = $derived(
    scaleBand().domain(seasons).range([0, innerHeight]).paddingOuter(0.5),
  );

  let nodes = $state([]);
  let gridTop = $state(0);
  let gridBottom = $state(0);
  let groupBySeason = $state(false);
  let layouts = $state();
  let hoveredIndex = $state();
  let selectedIndex = $state();
  let activeIndex = $derived(hoveredIndex ?? selectedIndex);
  let animationFrame;
  let prefersReducedMotion = false;

  function calculateLayout(grouped, currentXScale, currentYScale) {
    const layoutNodes = data.map((datum, index) => ({
      ...datum,
      layoutIndex: index,
    }));

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
      .force('collide', forceCollide().radius(collisionRadius))
      .stop()
      .tick(MOTION.layoutTicks);

    return layoutNodes;
  }

  function animateTo(targetLayout) {
    cancelAnimationFrame(animationFrame);
    const {
      nodes: targetNodes,
      gridTop: targetGridTop,
      gridBottom: targetGridBottom,
    } = targetLayout;

    if (nodes.length === 0 || prefersReducedMotion) {
      nodes = targetNodes;
      gridTop = targetGridTop;
      gridBottom = targetGridBottom;
      return;
    }

    const startNodes = nodes.map(node => ({ x: node.x, y: node.y }));
    const startGridTop = gridTop;
    const startGridBottom = gridBottom;
    const startTime = performance.now();

    function frame(time) {
      const progress = Math.min((time - startTime) / MOTION.duration, 1);
      const easedProgress = cubicOut(progress);

      nodes = targetNodes.map((target, index) => ({
        ...target,
        x:
          startNodes[index].x +
          (target.x - startNodes[index].x) * easedProgress,
        y:
          startNodes[index].y +
          (target.y - startNodes[index].y) * easedProgress,
      }));
      gridTop = startGridTop + (targetGridTop - startGridTop) * easedProgress;
      gridBottom =
        startGridBottom + (targetGridBottom - startGridBottom) * easedProgress;

      if (progress < 1) animationFrame = requestAnimationFrame(frame);
    }

    animationFrame = requestAnimationFrame(frame);
  }

  function findNearestNode(event) {
    const bounds = event.currentTarget.getBoundingClientRect();
    const pointerX =
      ((event.clientX - bounds.left) / bounds.width) * innerWidth;
    const pointerY =
      ((event.clientY - bounds.top) / bounds.height) * innerHeight;
    let nearestIndex;
    let nearestDistanceSquared = INTERACTION.hitRadius ** 2;

    nodes.forEach((node, index) => {
      const distanceSquared =
        (node.x - pointerX) ** 2 + (node.y - pointerY) ** 2;
      if (distanceSquared < nearestDistanceSquared) {
        nearestDistanceSquared = distanceSquared;
        nearestIndex = index;
      }
    });

    return nearestIndex;
  }

  function handlePointerMove(event) {
    if (event.pointerType === 'mouse') hoveredIndex = findNearestNode(event);
  }

  function handleChartClick(event) {
    selectedIndex = findNearestNode(event);
  }

  function handleChartKeydown(event) {
    const currentIndex = selectedIndex ?? 0;
    let nextIndex = currentIndex;

    if (event.key === 'ArrowRight' || event.key === 'ArrowDown') {
      nextIndex = (currentIndex + 1) % nodes.length;
    } else if (event.key === 'ArrowLeft' || event.key === 'ArrowUp') {
      nextIndex = (currentIndex - 1 + nodes.length) % nodes.length;
    } else if (event.key === 'Home') {
      nextIndex = 0;
    } else if (event.key === 'End') {
      nextIndex = nodes.length - 1;
    } else if (event.key === 'Escape') {
      selectedIndex = undefined;
      return;
    } else {
      return;
    }

    event.preventDefault();
    selectedIndex = nextIndex;
  }

  $effect(() => {
    const currentXScale = xScale;
    const currentYScale = yScale;
    const combinedNodes = calculateLayout(false, currentXScale, currentYScale);
    const groupedNodes = calculateLayout(true, currentXScale, currentYScale);
    const combinedOffset =
      MOTION.gridPadding - Math.min(...combinedNodes.map(node => node.y));

    combinedNodes.forEach(node => {
      node.y += combinedOffset;
    });

    const combinedYValues = combinedNodes.map(node => node.y);
    const groupedYValues = groupedNodes.map(node => node.y);

    layouts = {
      combined: {
        nodes: combinedNodes,
        gridTop: Math.max(0, Math.min(...combinedYValues) - MOTION.gridPadding),
        gridBottom: Math.min(
          innerHeight,
          Math.max(...combinedYValues) + MOTION.gridPadding,
        ),
      },
      grouped: {
        nodes: groupedNodes,
        gridTop: Math.max(0, Math.min(...groupedYValues) + 32),
        gridBottom: Math.min(innerHeight, Math.max(...groupedYValues) + 56),
      },
    };
  });

  $effect(() => {
    const targetLayout = groupBySeason ? layouts?.grouped : layouts?.combined;
    if (targetLayout) untrack(() => animateTo(targetLayout));
  });

  onMount(() => {
    const motionPreference = window.matchMedia(
      '(prefers-reduced-motion: reduce)',
    );
    const updateMotionPreference = () => {
      prefersReducedMotion = motionPreference.matches;
    };
    const viewport = window.visualViewport;
    const updateViewportHeight = () => {
      viewportHeight = viewport?.height ?? window.innerHeight;
    };

    updateMotionPreference();
    updateViewportHeight();
    motionPreference.addEventListener('change', updateMotionPreference);
    window.addEventListener('resize', updateViewportHeight);
    viewport?.addEventListener('resize', updateViewportHeight);

    return () => {
      motionPreference.removeEventListener('change', updateMotionPreference);
      window.removeEventListener('resize', updateViewportHeight);
      viewport?.removeEventListener('resize', updateViewportHeight);
    };
  });

  onDestroy(() => cancelAnimationFrame(animationFrame));
</script>

<main class="project">
  <h2>Average episode ratings for the series Frasier</h2>
  <div class="view-controls" aria-label="Chart view">
    <button
      type="button"
      class:active={!groupBySeason}
      aria-pressed={!groupBySeason}
      onclick={() => (groupBySeason = false)}>All episodes</button
    >
    <button
      type="button"
      class:active={groupBySeason}
      aria-pressed={groupBySeason}
      onclick={() => (groupBySeason = true)}>By season</button
    >
  </div>
  <p class="instructions">
    Tap a point for episode details. Use arrow keys when the chart is focused.
  </p>
  <!-- svelte-ignore a11y_no_noninteractive_tabindex -->
  <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
  <div
    role="application"
    tabindex="0"
    class="chart-container"
    aria-label="Episode ratings chart. Use arrow keys to explore episodes."
    bind:clientWidth={width}
    onkeydown={handleChartKeydown}
  >
    <svg {width} {height}>
      <g class="inner-chart" transform="translate({margin.left}, {margin.top})">
        <AxisY {yScale} {groupBySeason} />
        <AxisX {xScale} {gridTop} {gridBottom} />
        {#each nodes as node, i (node.layoutIndex)}
          <circle
            class="data-point"
            aria-hidden="true"
            cx={node.x}
            cy={node.y}
            r={pointRadius}
            fill={activeIndex === i ? 'orange' : '#f4f4f4'}
            stroke={'#555'}
            stroke-width={0.5}
          />
        {/each}
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <!-- svelte-ignore a11y_no_static_element_interactions -->
        <rect
          class="interaction-layer"
          class:near-point={hoveredIndex !== undefined}
          width={innerWidth}
          height={innerHeight}
          onpointermove={handlePointerMove}
          onpointerleave={() => (hoveredIndex = undefined)}
          onclick={handleChartClick}
        />
      </g>
    </svg>

    {#if activeIndex !== undefined && nodes[activeIndex]}
      <Tooltip
        data={nodes[activeIndex]}
        x={nodes[activeIndex].x + margin.left}
        y={nodes[activeIndex].y + margin.top}
        chartWidth={width}
        chartHeight={height}
      />
    {/if}
    <p
      class="source"
      style="top:{gridBottom + margin.top + 22}px; left:{margin.left}px;"
    >
      Source: IMDb
    </p>
  </div>
</main>

<style>
  .project {
    width: min(100%, 800px);
    margin-inline: auto;
  }

  .chart-container {
    width: 100%;
    font-size: 0.7rem;
    position: relative;
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

  .view-controls {
    display: inline-flex;
    margin-top: 0.75rem;
    padding: 2px;
    border: 1px solid #c8c8c8;
    border-radius: 0.5rem;
    background: #f2f2f2;
  }

  .view-controls button {
    min-height: 2rem;
    padding: 0.25rem 0.625rem;
    border: 0;
    border-radius: 0.375rem;
    color: #444;
    background: transparent;
    font: inherit;
    font-size: 0.8125rem;
    cursor: pointer;
  }

  .view-controls button.active {
    color: #111;
    background: #fff;
    box-shadow: 0 1px 2px rgb(0 0 0 / 12%);
  }

  .view-controls button:focus-visible,
  .chart-container:focus-visible {
    outline: 2px solid #d56700;
    outline-offset: 2px;
  }

  .instructions {
    margin-top: 0.625rem;
    color: #555;
    font-size: 0.875rem;
    line-height: 1.4;
  }

  .source {
    position: absolute;
    margin: 0;
  }

  .data-point {
    transition:
      stroke 100ms ease-out,
      opacity 100ms ease-out;
    pointer-events: none;
  }

  .interaction-layer {
    fill: transparent;
    touch-action: manipulation;
  }

  .interaction-layer.near-point {
    cursor: pointer;
  }
</style>
