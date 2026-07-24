<script>
  import { fly } from 'svelte/transition';
  import { cubicOut } from 'svelte/easing';

  /** @type {{yScale: any, groupBySeason: boolean, motionDuration: number}} */
  let { yScale, groupBySeason, motionDuration } = $props();

  let ticks = $derived(yScale.domain());
</script>

<g class="axis-y">
  {#each ticks as tick}
    {#if groupBySeason}
      <g
        class="tick"
        in:fly={{ x: -100, duration: motionDuration, easing: cubicOut }}
        out:fly={{ x: -100, duration: motionDuration, easing: cubicOut }}
      >
        <text
          x="-8"
          y={yScale(tick) + yScale.bandwidth() / 2}
          dominant-baseline="middle">Season {tick}</text
        >
      </g>
    {/if}
  {/each}
</g>

<style>
  .tick {
    text-anchor: end;
  }
</style>
