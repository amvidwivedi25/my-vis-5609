<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = {
    movies: TMovie[];
  };

  let { movies }: Props = $props();

  function getTopGenres(movies: TMovie[]) {
    let data: {
      year: number;
      genre: string;
      rank: number;
      count: number;
    }[] = [];

    const years = Array.from(
      new Set(movies.map((movie) => movie.year.getFullYear()))
    ).sort();

    years.forEach((year) => {
      let genreCounts: { [genre: string]: number } = {};

      movies
        .filter((movie) => movie.year.getFullYear() === year)
        .forEach((movie) => {
          movie.genres.forEach((genre) => {
            genreCounts[genre] = (genreCounts[genre] || 0) + 1;
          });
        });

      const top3 = Object.entries(genreCounts)
        .sort((a, b) => b[1] - a[1])
        .slice(0, 3);

      top3.forEach(([genre, count], index) => {
        data.push({
          year: year,
          genre: genre,
          rank: index + 1,
          count: count
        });
      });
    });

    return data;
  }

  const topGenres = $derived(getTopGenres(movies));
const width = 950;
const height = 400;

const margin = {
  top: 30,
  right: 30,
  bottom: 50,
  left: 50
};

const years = $derived(
  Array.from(new Set(topGenres.map((d) => d.year))).sort()
);

const xScale = $derived(
  d3
    .scalePoint<number>()
    .domain(years)
    .range([margin.left, 780])
);

const yScale = d3
  .scaleLinear()
  .domain([1, 3])
  .range([margin.top, height - margin.bottom]); 
const genres = $derived(
  Array.from(new Set(topGenres.map((d) => d.genre)))
    .filter((genre) => {
      return topGenres.filter((d) => d.genre === genre).length >= 5;
    })
);

function getPointsForGenre(genre: string) {
  return topGenres.filter((d) => d.genre === genre);
}

const colorScale = d3
  .scaleOrdinal<string>()
  .domain(genres)
  .range(d3.schemeTableau10);
</script>

<h3>Top 3 Genres Over Time</h3>

<svg width={width} height={height}>
  <g transform={`translate(0, ${height - margin.bottom})`}>
  {#each years as year, i}
    {#if i % 5 === 0}
      <text
        x={xScale(year)}
        y="20"
        font-size="10"
        text-anchor="middle"
      >
        {year}
      </text>
    {/if}
  {/each}
</g>

  {#each [1, 2, 3] as rank}
    <text
      x={margin.left - 20}
      y={yScale(rank) + 4}
      font-size="12"
      text-anchor="middle"
    >
      {rank}
    </text>
  {/each}
{#each genres as genre}
  {@const points = getPointsForGenre(genre)}

  {#each points as point, i}
    <circle
      cx={xScale(point.year)}
      cy={yScale(point.rank)}
      r="3"
      fill={colorScale(genre)}
    />

    {#if i < points.length - 1 && points[i + 1].year === point.year + 1}
      <line
        x1={xScale(point.year)}
        y1={yScale(point.rank)}
        x2={xScale(points[i + 1].year)}
        y2={yScale(points[i + 1].rank)}
        stroke={colorScale(genre)}
        stroke-width="2"
      />
    {/if}
  {/each}
{/each}

<g transform="translate(810, 30)">
 {#each genres as genre, i}
    <circle
      cx="0"
      cy={i * 18}
      r="4"
      fill={colorScale(genre)}
    />

    <text
      x="10"
      y={i * 18 + 4}
      font-size="11"
    >
      {genre}
    </text>
  {/each}
</g>
<text
  x={width / 2}
  y={height - 15}
  text-anchor="middle"
  font-size="14"
>
  Year
</text>

<text
  x="15"
  y={height / 2}
  text-anchor="middle"
  font-size="14"
  transform={`rotate(-90, 15, ${height / 2})`}
>
  Rank
</text>
</svg>
