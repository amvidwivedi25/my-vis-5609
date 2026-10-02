<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = {
    movies: TMovie[];
  };

  let { movies }: Props = $props();

  let hoveredPair: string | undefined = $state(undefined);
  let hoveredCount: number = $state(0);

  function getGenrePairs(movies: TMovie[]) {
    let counts: { [pair: string]: number } = {};

    movies.forEach((movie) => {
      const genres = movie.genres;

      for (let i = 0; i < genres.length; i++) {
        for (let j = i + 1; j < genres.length; j++) {
          const genre1 = genres[i];
          const genre2 = genres[j];

          const pair = [genre1, genre2].sort().join("|");

          counts[pair] = (counts[pair] || 0) + 1;
        }
      }
    });

    return counts;
  }

  const genrePairs = $derived(getGenrePairs(movies));

  const genres = $derived(
    Array.from(
      new Set(
        movies.flatMap((movie) => movie.genres)
      )
    ).sort()
  );

  const width = 800;
  const height = 800;

  const margin = {
    top: 120,
    right: 30,
    bottom: 30,
    left: 120
  };

  const xScale = $derived(
    d3
      .scaleBand()
      .domain(genres)
      .range([margin.left, width - margin.right])
      .padding(0.1)
  );

  const yScale = $derived(
    d3
      .scaleBand()
      .domain(genres)
      .range([margin.top, height - margin.bottom])
      .padding(0.1)
  );

  const maxCount = $derived(
    d3.max(Object.values(genrePairs)) || 0
  );

  const sizeScale = $derived(
    d3
      .scaleSqrt()
      .domain([0, maxCount])
      .range([0, 15])
  );
</script>

<h3>Genre Correlations</h3>

{#if hoveredPair}
  <p>{hoveredPair}: {hoveredCount} movies</p>
{/if}

<svg width={width} height={height}>
  {#each Object.entries(genrePairs) as [pair, count]}
    {@const parts = pair.split("|")}
    {@const genre1 = parts[0]}
    {@const genre2 = parts[1]}

    <circle
      cx={(xScale(genre1) ?? 0) + xScale.bandwidth() / 2}
      cy={(yScale(genre2) ?? 0) + yScale.bandwidth() / 2}
      r={sizeScale(count)}
      fill="steelblue"
      opacity="0.7"
      onmouseenter={() => {
        hoveredPair = `${genre1} + ${genre2}`;
        hoveredCount = count;
      }}
      onmouseleave={() => {
        hoveredPair = undefined;
      }}
    />

    <circle
      cx={(xScale(genre2) ?? 0) + xScale.bandwidth() / 2}
      cy={(yScale(genre1) ?? 0) + yScale.bandwidth() / 2}
      r={sizeScale(count)}
      fill="steelblue"
      opacity="0.7"
      onmouseenter={() => {
        hoveredPair = `${genre1} + ${genre2}`;
        hoveredCount = count;
      }}
      onmouseleave={() => {
        hoveredPair = undefined;
      }}
    />
  {/each}

  {#each genres as genre}
    <text
      x={(xScale(genre) ?? 0) + xScale.bandwidth() / 2}
      y={margin.top - 10}
      font-size="10"
      text-anchor="start"
      transform={`rotate(-45, ${(xScale(genre) ?? 0) + xScale.bandwidth() / 2}, ${margin.top - 10})`}
    >
      {genre}
    </text>

    <text
      x={margin.left - 10}
      y={(yScale(genre) ?? 0) + yScale.bandwidth() / 2 + 4}
      font-size="10"
      text-anchor="end"
    >
      {genre}
    </text>
  {/each}
</svg>