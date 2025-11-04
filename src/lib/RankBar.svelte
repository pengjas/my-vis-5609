<script lang="ts">
  import { fade } from "svelte/transition";
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  // define the props of the Bar component
  type Props = {
    movies: TMovie[];
    progress?: number;
    width?: number;
    height?: number;
  };
  // progress is 100 by default unless specified
  let { movies, progress = 100, width = 500, height = 400 }: Props = $props();

  let selectedGenre: string = $state('');

  // processing the data; $derived is used to create a reactive variable that updates whenever the dependent variables change
  const yearRange = $derived(d3.extent(movies.map((d) => d.year)));

  function getUpYear(yearRange: [undefined, undefined] | [Date, Date]) {
    if (!yearRange[0]) return new Date();
    const timeScale = d3.scaleTime().domain(yearRange).range([0, 100]);
    return timeScale.invert(progress);
  }
  const upYear: Date = $derived(getUpYear(yearRange!));

  function getGenreNums(movies: TMovie[], upYear: Date) {
    let res: { [genre: string]: number } = {};
    movies
      .filter((movie) => movie.year <= upYear)
      .forEach((movie) => {
        movie.genres.forEach((genre: string) => {
          res[genre] = (res[genre] || 0) + 1;
        });
      });
    return res;
  }

  const genreNums = $derived(getGenreNums(movies, upYear));
  const genreRank = $derived(Object.entries(genreNums));

  // drawing the bar chart
  const margin = {
    top: 0,
    bottom: 100,
    left: 70,
    right: 20,
  };

  let usableArea = {
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left,
  };

  const yScale = $derived(
    d3.scaleBand()
      .range([usableArea.top, usableArea.bottom])
      .domain(genreRank.sort((a, b) => b[1] - a[1]).map(([genre, _]) => genre))
      .padding(0.1),
  );

  const xScale = $derived(
    d3.scaleLinear()
      .range([0, usableArea.right - usableArea.left])
      .domain([0, d3.max(Object.values(genreNums)) || 0]),
  );

  // D3 坐标轴逻辑
  let xAxis: SVGGElement;
  let yAxis: SVGGElement;

 // 在 runes 模式中使用 $effect 来响应式地运行副作用（并避免 SSR 报错）
$effect(() => {
  // 只在浏览器环境运行 DOM 操作（避免 SSR 报错）
  if (typeof window === "undefined") return;

  if (!xAxis) return;
  // 如果 xScale 是响应式/derived 值，$effect 会在它变化时重新执行
  const axisBottom = d3.axisBottom(xScale);
  d3.select(xAxis).call(axisBottom);
});

$effect(() => {
  if (typeof window === "undefined") return;

  if (!yAxis) return;
  const axisLeft = d3.axisLeft(yScale);
  d3.select(yAxis).call(axisLeft);
});


</script>

<h3>
  The Ranking of Genres {yearRange[0]?.getFullYear()} - {upYear?.getFullYear()}
</h3>

{#if movies.length > 0}
  <svg {width} {height}>
    <g class="bars" transform="translate({usableArea.left}, 0)">
      {#each genreRank as [genre, cnt] (genre)}
        <g 
          class={`${genre} genre`}
          transform="translate(0, {yScale(genre)})"
          in:fade={{ delay: 300, duration: 300 }}
          out:fade={{ duration: 300 }}
        >
          <rect 
            y={0}
            width={xScale(cnt)} 
            height={yScale.bandwidth()} 
            fill={selectedGenre === genre ? "#ff8800" : "#449900"} 
            class="bar" 
            opacity={selectedGenre === genre ? 1 : 0.8} 
            on:mouseover={() => { selectedGenre = genre }} 
            on:mouseout={() => { selectedGenre = "" }}
          />

        <text
          x={5}
          y={yScale.bandwidth() / 2}
          font-size="12"
          fill={selectedGenre === genre ? "white" : "black"}
          text-anchor="start"
          dominant-baseline="middle"
          style="pointer-events: none;"
        >
          {genre}
        </text>
                  <text
            x={xScale(cnt) + 5}
            y={yScale.bandwidth() / 2}
            font-size="12"
            text-anchor="start"
            dominant-baseline="middle"
            fill="black"
          >
            {cnt} 
          </text>
        </g>
      {/each}
    </g>

    <g transform="translate({usableArea.left}, {usableArea.bottom})" bind:this={xAxis} />
    <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
  </svg>
{/if}

<style>
  .genre {
    /** 为 transform 添加 CSS 过渡，以实现平滑的位置变化 */
    transition: transform 0.3s ease;
  }
  .bar {
    transition:
      y 0.1s ease,
      height 0.1s ease,
      width 0.1s ease; /* Smooth transition for height */
  }
</style>