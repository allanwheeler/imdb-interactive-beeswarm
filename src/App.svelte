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
  const SEARCH_RESULT_LIMIT = 5;

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
  let searchQuery = $state('');
  let isSearchOpen = $state(false);
  let activeSuggestionIndex = $state(0);
  let searchInput;
  let activeIndex = $derived(hoveredIndex ?? selectedIndex);
  let normalizedSearchQuery = $derived(searchQuery.trim().toLocaleLowerCase());
  let matchingEpisodes = $derived.by(() => {
    if (!normalizedSearchQuery) return [];

    return data
      .map((episode, index) => ({ ...episode, index }))
      .filter(episode =>
        episode.title.toLocaleLowerCase().includes(normalizedSearchQuery),
      );
  });
  let episodeSuggestions = $derived(
    matchingEpisodes.slice(0, SEARCH_RESULT_LIMIT),
  );
  let activeSuggestionId = $derived(
    isSearchOpen && episodeSuggestions[activeSuggestionIndex]
      ? `episode-search-option-${episodeSuggestions[activeSuggestionIndex].index}`
      : undefined,
  );
  let searchAnnouncement = $derived.by(() => {
    if (!isSearchOpen || !normalizedSearchQuery) return '';

    if (matchingEpisodes.length === 0) {
      return `No episodes match ${searchQuery.trim()}.`;
    }

    const resultCount = episodeSuggestions.length;
    const resultLabel = resultCount === 1 ? 'suggestion' : 'suggestions';
    const shownLabel =
      matchingEpisodes.length > SEARCH_RESULT_LIMIT
        ? `${resultCount} of ${matchingEpisodes.length}`
        : `${resultCount}`;

    return `${shownLabel} ${resultLabel} available. Use the up and down arrow keys to review them.`;
  });
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

  function handleSearchInput(event) {
    searchQuery = event.currentTarget.value;
    activeSuggestionIndex = 0;
    isSearchOpen = searchQuery.trim().length > 0;
  }

  function handleSearchFocus() {
    if (normalizedSearchQuery) isSearchOpen = true;
  }

  function handleSearchBlur(event) {
    const search = event.currentTarget.closest('.episode-search');
    if (!search?.contains(event.relatedTarget)) isSearchOpen = false;
  }

  function selectEpisodeSuggestion(suggestion) {
    hoveredIndex = undefined;
    selectedIndex = suggestion.index;
    searchQuery = suggestion.title;
    activeSuggestionIndex = 0;
    isSearchOpen = false;
    searchInput?.focus();
  }

  function clearSearch() {
    searchQuery = '';
    activeSuggestionIndex = 0;
    isSearchOpen = false;
    searchInput?.focus();
  }

  function handleSearchKeydown(event) {
    const suggestionCount = episodeSuggestions.length;

    if (event.key === 'ArrowDown') {
      if (suggestionCount === 0) return;
      event.preventDefault();

      if (!isSearchOpen) {
        isSearchOpen = true;
        activeSuggestionIndex = 0;
      } else {
        activeSuggestionIndex = (activeSuggestionIndex + 1) % suggestionCount;
      }
    } else if (event.key === 'ArrowUp') {
      if (suggestionCount === 0) return;
      event.preventDefault();

      if (!isSearchOpen) {
        isSearchOpen = true;
        activeSuggestionIndex = suggestionCount - 1;
      } else {
        activeSuggestionIndex =
          (activeSuggestionIndex - 1 + suggestionCount) % suggestionCount;
      }
    } else if (event.key === 'Home' && isSearchOpen && suggestionCount > 0) {
      event.preventDefault();
      activeSuggestionIndex = 0;
    } else if (event.key === 'End' && isSearchOpen && suggestionCount > 0) {
      event.preventDefault();
      activeSuggestionIndex = suggestionCount - 1;
    } else if (
      event.key === 'Enter' &&
      isSearchOpen &&
      episodeSuggestions[activeSuggestionIndex]
    ) {
      event.preventDefault();
      selectEpisodeSuggestion(episodeSuggestions[activeSuggestionIndex]);
    } else if (event.key === 'Escape' && isSearchOpen) {
      event.preventDefault();
      isSearchOpen = false;
    } else if (event.key === 'Tab') {
      isSearchOpen = false;
    }
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
  <div class="control-bar">
    <div class="episode-search">
      <label class="sr-only" for="episode-search-input">Find an episode</label>
      <div class="search-input-shell">
        <svg
          class="search-icon"
          viewBox="0 0 20 20"
          aria-hidden="true"
          focusable="false"
        >
          <circle cx="8.5" cy="8.5" r="5.5"></circle>
          <path d="m12.5 12.5 4.5 4.5"></path>
        </svg>
        <input
          id="episode-search-input"
          class="search-input"
          type="search"
          name="episode-search"
          placeholder="Search episode titles"
          autocomplete="off"
          spellcheck="false"
          role="combobox"
          aria-autocomplete="list"
          aria-expanded={isSearchOpen}
          aria-controls="episode-search-listbox"
          aria-activedescendant={activeSuggestionId}
          aria-describedby="episode-search-status"
          value={searchQuery}
          bind:this={searchInput}
          oninput={handleSearchInput}
          onfocus={handleSearchFocus}
          onblur={handleSearchBlur}
          onkeydown={handleSearchKeydown}
        />
        {#if searchQuery}
          <button
            class="search-clear"
            type="button"
            aria-label="Clear search"
            onclick={clearSearch}
          >
            <span aria-hidden="true">×</span>
          </button>
        {/if}
      </div>
      {#if isSearchOpen && normalizedSearchQuery}
        <ul
          class="search-results"
          id="episode-search-listbox"
          role="listbox"
          aria-label="Episode suggestions"
        >
          {#each episodeSuggestions as suggestion, index (suggestion.index)}
            <li role="presentation">
              <button
                id="episode-search-option-{suggestion.index}"
                class="search-result"
                class:active={index === activeSuggestionIndex}
                type="button"
                role="option"
                tabindex="-1"
                aria-selected={selectedIndex === suggestion.index}
                onpointerenter={() => (activeSuggestionIndex = index)}
                onclick={() => selectEpisodeSuggestion(suggestion)}
              >
                <span class="search-result-title">{suggestion.title}</span>
                <span class="search-result-meta"
                  >Season {suggestion.season}, episode
                  {suggestion.episode}</span
                >
              </button>
            </li>
          {:else}
            <li class="search-empty" role="presentation">
              No episodes match “{searchQuery.trim()}”.
            </li>
          {/each}
        </ul>
      {/if}
      <p class="sr-only" id="episode-search-status" aria-live="polite">
        {searchAnnouncement}
      </p>
    </div>
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
        <AxisX
          {xScale}
          {gridTop}
          {gridBottom}
          {seriesAverage}
          referenceTop={-margin.top}
        />
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
        {#if comparisonRoute}
          {#key `${selectedIndex}-${groupBySeason}`}
            <path
              class="selection-comparison-line"
              d={comparisonRoute.path}
              in:draw={{
                duration: prefersReducedMotion ? 0 : MOTION.comparisonDuration,
                easing: cubicOut,
              }}
            />
          {/key}
        {/if}
        {#if groupBySeason && selectedIndex !== undefined && nodes[selectedIndex]}
          {#key `${selectedIndex}-${groupBySeason}`}
            <line
              class="selection-comparison-line"
              x1={xScale(seriesAverage)}
              x2={nodes[selectedIndex].x}
              y1={nodes[selectedIndex].y}
              y2={nodes[selectedIndex].y}
              in:draw={{
                duration: prefersReducedMotion ? 0 : MOTION.comparisonDuration,
                easing: cubicOut,
              }}
            />
          {/key}
        {/if}
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
    width: min(100%, 700px);
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

  .control-bar {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    flex-wrap: wrap;
    gap: 0.75rem 1rem;
    margin-top: 0.75rem;
  }

  .view-controls {
    position: relative;
    display: inline-grid;
    grid-template-columns: repeat(2, 1fr);
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

  .episode-search {
    position: relative;
    flex: 1 1 17rem;
    width: min(100%, 22rem);
    max-width: 22rem;
  }

  .search-input-shell {
    position: relative;
  }

  .search-icon {
    position: absolute;
    z-index: 1;
    inset-block-start: 50%;
    inset-inline-start: 0.75rem;
    width: 1rem;
    height: 1rem;
    color: oklch(0.45 0 0);
    fill: none;
    stroke: currentColor;
    stroke-width: 1.5;
    stroke-linecap: round;
    transform: translateY(-50%);
    pointer-events: none;
  }

  .search-input {
    box-sizing: border-box;
    width: 100%;
    min-height: 2.5rem;
    padding: 0.4375rem 2.75rem 0.4375rem 2.375rem;
    border: 1px solid oklch(0.76 0 0);
    border-radius: 0.5rem;
    color: #111;
    background: #fff;
    font: inherit;
    font-size: 1rem;
  }

  .search-input::placeholder {
    color: oklch(0.5 0 0);
  }

  .search-input::-webkit-search-cancel-button {
    appearance: none;
  }

  .search-clear {
    position: absolute;
    inset-block: 0;
    inset-inline-end: 0;
    width: 2.5rem;
    padding: 0;
    border: 0;
    border-radius: 0.375rem;
    color: oklch(0.4 0 0);
    background: transparent;
    font: inherit;
    font-size: 1.25rem;
    line-height: 1;
    cursor: pointer;
  }

  .search-results {
    position: absolute;
    z-index: 10;
    inset: calc(100% + 0.25rem) 0 auto;
    max-height: min(18rem, 50vh);
    margin: 0;
    padding: 0.25rem;
    overflow-y: auto;
    border: 1px solid oklch(0.82 0 0);
    border-radius: 0.625rem;
    background: #fff;
    box-shadow:
      0 1px 2px oklch(0 0 0 / 8%),
      0 8px 24px oklch(0 0 0 / 10%);
    list-style: none;
  }

  .search-result {
    display: grid;
    width: 100%;
    gap: 0.125rem;
    padding: 0.5rem 0.625rem;
    border: 0;
    border-radius: 0.375rem;
    color: #222;
    background: transparent;
    font: inherit;
    text-align: start;
    cursor: pointer;
  }

  .search-result.active {
    background: var(--color-accent-soft);
  }

  .search-result[aria-selected='true'] .search-result-title {
    color: var(--color-accent-strong);
    font-weight: 600;
  }

  .search-result-title {
    overflow-wrap: break-word;
    font-size: 0.875rem;
    line-height: 1.25;
  }

  .search-result-meta,
  .search-empty {
    color: oklch(0.45 0 0);
    font-size: 0.75rem;
    line-height: 1.35;
  }

  .search-empty {
    padding: 0.625rem;
  }

  .view-controls button:focus-visible,
  .search-input:focus-visible,
  .search-clear:focus-visible,
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

  @media (min-width: 30rem) {
    .search-input {
      font-size: 0.875rem;
    }
  }
</style>
