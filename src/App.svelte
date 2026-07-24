<script>
  import { onDestroy, onMount, untrack } from 'svelte';
  import { cubicOut } from 'svelte/easing';
  import { draw, fade } from 'svelte/transition';

  import { forceSimulation, forceX, forceY, forceCollide } from 'd3-force';
  import { scaleLinear, scaleBand } from 'd3-scale';
  import data from './data/data.js';
  import AxisX from './components/AxisX.svelte';
  import AxisY from './components/AxisY.svelte';
  import Tooltip from './components/Tooltip.svelte';

  let width = $state(400);
  let viewportHeight = $state(0);
  let expandedChartHeight = $derived(
    width < 600 && viewportHeight
      ? Math.max(420, Math.round(viewportHeight * 0.88))
      : Math.min(600, Math.max(420, Math.round(width * 0.75))),
  );
  let pointRadius = $derived(width < 480 ? 4 : 5);
  let collisionRadius = $derived(pointRadius + 2);

  const margin = { top: 24, right: 44, left: 64, bottom: 56 };
  const seriesAverage = data[0].avgSeriesRating;

  /* ─────────────────────────────────────────────────────────
   * ANIMATION STORYBOARD
   *
   *    0ms   toggle selects a precomputed stable layout
   *  300ms   dots arrive at their new positions with cubic ease-out
   * ───────────────────────────────────────────────────────── */
  const MOTION = {
    duration: 300,
    feedbackDuration: 120,
    comparisonDuration: 150,
    layoutTicks: 300,
    gridPadding: 32,
  };
  const LAYOUT = {
    minimumCombinedHeight: 220,
    sourceOffset: 38,
    chartEndPadding: 18,
  };
  const INTERACTION = {
    hitRadius: 24,
  };

  let innerWidth = $derived(width - margin.left - margin.right);
  let innerHeight = $derived(expandedChartHeight - margin.top - margin.bottom);

  let seasons = Array.from(new Set(data.map(d => d.season)));

  let xScale = $derived(scaleLinear().domain([6.5, 10]).range([0, innerWidth]));
  let yScale = $derived(
    scaleBand().domain(seasons).range([0, innerHeight]).paddingOuter(0.5),
  );

  let nodes = $state([]);
  let gridTop = $state(0);
  let gridBottom = $state(0);
  let chartHeight = $state(0);
  let viewIndicatorPosition = $state(0);
  let groupBySeason = $state(false);
  let layouts = $state();
  let hoveredIndex = $state();
  let selectedIndex = $state();
  let activeIndex = $derived(hoveredIndex ?? selectedIndex);
  let animationFrame;
  let prefersReducedMotion = $state(false);
  let interactionHeight = $derived(
    Math.min(
      innerHeight,
      Math.max(gridBottom, chartHeight - margin.top - margin.bottom),
    ),
  );
  let comparisonRoute = $derived.by(() => {
    if (groupBySeason || selectedIndex === undefined || !nodes[selectedIndex]) {
      return undefined;
    }

    const selectedNode = nodes[selectedIndex];
    const averageX = xScale(seriesAverage);
    const routeY = Math.min(
      gridBottom - 8,
      Math.max(...nodes.map(node => node.y)) + pointRadius + 7,
    );

    return {
      path: `M ${averageX} ${routeY} H ${selectedNode.x} V ${selectedNode.y}`,
      labelX: (averageX + selectedNode.x) / 2,
      labelY: routeY + 13,
    };
  });

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
          .y(d =>
            grouped
              ? currentYScale(d.season) + currentYScale.bandwidth() / 2
              : innerHeight / 2,
          )
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
      chartHeight: targetChartHeight,
      viewIndicatorPosition: targetViewIndicatorPosition,
    } = targetLayout;

    if (nodes.length === 0 || prefersReducedMotion) {
      nodes = targetNodes;
      gridTop = targetGridTop;
      gridBottom = targetGridBottom;
      chartHeight = targetChartHeight;
      viewIndicatorPosition = targetViewIndicatorPosition;
      return;
    }

    const startNodes = nodes.map(node => ({ x: node.x, y: node.y }));
    const startGridTop = gridTop;
    const startGridBottom = gridBottom;
    const startChartHeight = chartHeight;
    const startViewIndicatorPosition = viewIndicatorPosition;
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
      chartHeight =
        startChartHeight +
        (targetChartHeight - startChartHeight) * easedProgress;
      viewIndicatorPosition =
        startViewIndicatorPosition +
        (targetViewIndicatorPosition - startViewIndicatorPosition) *
          easedProgress;

      if (progress < 1) animationFrame = requestAnimationFrame(frame);
    }

    animationFrame = requestAnimationFrame(frame);
  }

  function findNearestNode(event) {
    const bounds = event.currentTarget.getBoundingClientRect();
    const pointerX =
      ((event.clientX - bounds.left) / bounds.width) * innerWidth;
    const pointerY =
      ((event.clientY - bounds.top) / bounds.height) * interactionHeight;
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

  function formatRatingDifference(rating) {
    const difference = rating - seriesAverage;
    const sign = difference > 0 ? '+' : difference < 0 ? '−' : '';
    return `${sign}${Math.abs(difference).toFixed(1)} difference`;
  }

  function describeRatingDifference(rating) {
    const difference = rating - seriesAverage;
    if (difference === 0) return 'matches the series average';
    return `${Math.abs(difference).toFixed(1)} stars ${
      difference > 0 ? 'above' : 'below'
    } the series average`;
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
    hoveredIndex = undefined;
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
    const combinedGridBottom = Math.min(
      innerHeight,
      Math.max(...combinedYValues) + MOTION.gridPadding,
    );
    const groupedGridBottom = Math.min(
      innerHeight,
      Math.max(...groupedYValues) + 56,
    );

    layouts = {
      combined: {
        nodes: combinedNodes,
        viewIndicatorPosition: 0,
        gridTop: Math.max(0, Math.min(...combinedYValues) - MOTION.gridPadding),
        gridBottom: combinedGridBottom,
        chartHeight: Math.min(
          expandedChartHeight,
          Math.max(
            LAYOUT.minimumCombinedHeight,
            margin.top +
              combinedGridBottom +
              LAYOUT.sourceOffset +
              LAYOUT.chartEndPadding,
          ),
        ),
      },
      grouped: {
        nodes: groupedNodes,
        viewIndicatorPosition: 1,
        gridTop: Math.max(0, Math.min(...groupedYValues) - MOTION.gridPadding),
        gridBottom: groupedGridBottom,
        chartHeight: expandedChartHeight,
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

<main class="project" style:--feedback-duration="{MOTION.feedbackDuration}ms">
  <h1 id="chart-title">Average episode ratings for the series Frasier</h1>
  <div class="view-controls" role="group" aria-label="Chart view">
    <span
      class="view-indicator"
      aria-hidden="true"
      style:transform="translateX({viewIndicatorPosition * 100}%)"
    ></span>
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
  <p class="instructions" id="chart-instructions">
    Hover or tap a point for episode details. After selecting a point, use the
    arrow keys to cycle through episodes chronologically.
  </p>
  <!-- svelte-ignore a11y_no_noninteractive_tabindex -->
  <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
  <div
    role="region"
    tabindex="0"
    class="chart-container"
    aria-labelledby="chart-title"
    aria-describedby="chart-instructions"
    bind:clientWidth={width}
    onfocus={() => (hoveredIndex = undefined)}
    onkeydown={handleChartKeydown}
  >
    <svg {width} height={chartHeight} aria-hidden="true" focusable="false">
      <g class="inner-chart" transform="translate({margin.left}, {margin.top})">
        <AxisY
          {yScale}
          {groupBySeason}
          motionDuration={prefersReducedMotion ? 0 : MOTION.duration}
        />
        <AxisX {xScale} {gridTop} {gridBottom} {seriesAverage} />
        {#if selectedIndex !== undefined && nodes[selectedIndex]}
          {#key `${selectedIndex}-${groupBySeason}`}
            {#if comparisonRoute}
              <path
                class="selection-comparison-line"
                d={comparisonRoute.path}
                in:draw={{
                  duration: prefersReducedMotion
                    ? 0
                    : MOTION.comparisonDuration,
                  easing: cubicOut,
                }}
              />
            {:else}
              <line
                class="selection-comparison-line"
                x1={xScale(seriesAverage)}
                x2={nodes[selectedIndex].x}
                y1={nodes[selectedIndex].y}
                y2={nodes[selectedIndex].y}
                in:draw={{
                  duration: prefersReducedMotion
                    ? 0
                    : MOTION.comparisonDuration,
                  easing: cubicOut,
                }}
              />
            {/if}
          {/key}
        {/if}
        {#each nodes as node, i (node.layoutIndex)}
          <circle
            class="data-point"
            class:hovered={hoveredIndex === i}
            class:selected={selectedIndex === i}
            cx={node.x}
            cy={node.y}
            r={pointRadius}
          />
        {/each}
        {#if selectedIndex !== undefined && nodes[selectedIndex]}
          <circle
            class="selection-ring"
            cx={nodes[selectedIndex].x}
            cy={nodes[selectedIndex].y}
            r={pointRadius + 4}
          />
          {#key selectedIndex}
            <text
              class="selection-comparison-label"
              x={comparisonRoute?.labelX ??
                (xScale(seriesAverage) + nodes[selectedIndex].x) / 2}
              y={comparisonRoute?.labelY ??
                nodes[selectedIndex].y - pointRadius - 7}
              in:fade={{
                duration: prefersReducedMotion ? 0 : MOTION.comparisonDuration,
                easing: cubicOut,
              }}
              >{formatRatingDifference(
                nodes[selectedIndex].averageEpisodeRating,
              )}</text
            >
          {/key}
        {/if}
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <!-- svelte-ignore a11y_no_static_element_interactions -->
        <rect
          class="interaction-layer"
          class:near-point={hoveredIndex !== undefined}
          width={innerWidth}
          height={interactionHeight}
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
        {chartHeight}
        {seriesAverage}
      />
    {/if}
    <p class="sr-only" role="status">
      {#if selectedIndex !== undefined && nodes[selectedIndex]}
        Season {nodes[selectedIndex].season}, episode
        {nodes[selectedIndex].episode}: {nodes[selectedIndex].title},
        {nodes[selectedIndex].averageEpisodeRating} out of 10,
        {describeRatingDifference(nodes[selectedIndex].averageEpisodeRating)}.
      {/if}
    </p>
    <p
      class="source"
      style="top:{gridBottom +
        margin.top +
        LAYOUT.sourceOffset}px; left:{margin.left}px;"
    >
      Source: IMDb
    </p>
  </div>
</main>

<style>
  .project {
    --color-accent: oklch(0.72 0.17 55);
    --color-accent-soft: oklch(0.92 0.05 70);
    --color-accent-strong: oklch(0.56 0.16 50);
    --color-point: oklch(0.965 0 0);
    --color-point-stroke: oklch(0.45 0 0);
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

  h1 {
    margin-bottom: 0.5rem;
    font-size: 1.5rem;
    font-weight: 600;
  }

  .view-controls {
    position: relative;
    display: inline-grid;
    grid-template-columns: repeat(2, 1fr);
    margin-top: 0.75rem;
    padding: 2px;
    border: 1px solid rgb(200, 200, 200, 0.1);
    border-radius: 0.5rem;
    background: #f2f2f2;
  }

  .view-indicator {
    position: absolute;
    z-index: 0;
    inset: 2px auto 2px 2px;
    width: calc(50% - 2px);
    border-radius: 0.375rem;
    background: #fff;
    box-shadow: 0 1px 2px rgb(0 0 0 / 12%);
    pointer-events: none;
    will-change: transform;
  }

  .view-controls button {
    position: relative;
    z-index: 1;
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
  }

  .view-controls button:focus-visible,
  .chart-container:focus-visible {
    outline: 2px solid;
    outline-offset: 2px;
  }

  .instructions {
    margin-top: 0.625rem;
    margin-bottom: 0.75rem;
    color: oklch(0.45 0 0);
    font-size: 0.875rem;
    line-height: 1.4;
  }

  .source {
    position: absolute;
    margin: 0;
    color: oklch(0.45 0 0);
  }

  .data-point {
    fill: var(--color-point);
    stroke: var(--color-point-stroke);
    stroke-width: 0.5;
    pointer-events: none;
  }

  .data-point.hovered {
    fill: var(--color-accent-soft);
    stroke: var(--color-accent-strong);
    stroke-width: 1;
  }

  .data-point.selected {
    fill: var(--color-accent);
    stroke: var(--color-accent-strong);
    stroke-width: 1.5;
  }

  .selection-ring {
    fill: none;
    stroke: var(--color-accent-strong);
    stroke-width: 1.5;
    pointer-events: none;
  }

  .selection-comparison-line {
    fill: none;
    stroke: var(--color-accent-strong);
    stroke-width: 1;
    stroke-dasharray: 3 4;
    stroke-linecap: round;
    stroke-linejoin: round;
    opacity: 0.55;
    pointer-events: none;
  }

  .selection-comparison-label {
    fill: var(--color-accent-strong);
    stroke: white;
    stroke-width: 3;
    paint-order: stroke;
    font-weight: 600;
    text-anchor: middle;
    pointer-events: none;
  }

  .interaction-layer {
    fill: transparent;
    touch-action: manipulation;
  }

  .interaction-layer.near-point {
    cursor: pointer;
  }

  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0 0 0 0);
    clip-path: inset(50%);
    white-space: nowrap;
    border: 0;
  }

  @media (prefers-reduced-motion: no-preference) {
    .data-point {
      transition:
        fill var(--feedback-duration) ease,
        stroke var(--feedback-duration) ease,
        stroke-width var(--feedback-duration) ease;
    }
  }
</style>
