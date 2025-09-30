<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  // 组件props：接收movies数组，以及可选的尺寸、进度（默认100%）
  type Props = {
    movies: TMovie[];
    progress?: number;
    width?: number;
    height?: number;
  };
  let { movies, progress = 100, width = 500, height = 400 }: Props = $props();

  // 选中的类型（用于交互高亮）
  let selectedGenre: string | undefined = $state();

  // 计算电影的年份范围（最早→最晚）
  const yearRange = $derived(d3.extent(movies.map((d) => d.year)));

  // 根据progress计算当前时间线截止年份（progress=100对应最晚年份）
  function getUpYear(yearRange: [undefined, undefined] | [Date, Date]) {
    if (!yearRange[0] || !yearRange[1]) return new Date();
    const timeScale = d3.scaleTime().domain(yearRange).range([0, 100]);
    return timeScale.invert(progress);
  }
  const upYear: Date = $derived(getUpYear(yearRange!));

  // 统计截止到upYear的各类型电影数量（genre→数量）
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

  // 图表边距（防止坐标轴被截断）
  const margin = { top: 15, bottom: 50, left: 30, right: 10 };
  // 可用绘图区域（扣除边距后的坐标范围）
  const usableArea = $derived({
    left: margin.left,
    right: width - margin.right,
    top: margin.top,
    bottom: height - margin.bottom,
  });

  // -------------------------- 1. 补全X轴比例尺（分类数据：类型） --------------------------
  const xScale = $derived(
    d3.scaleBand()
      .domain(Object.keys(genreNums)) // X轴数据：所有电影类型
      .range([usableArea.left, usableArea.right]) // X轴像素范围：从左到右
      .padding(0.1) // 柱子之间的间距（占带宽的10%）
  );

  // -------------------------- 2. 补全Y轴比例尺（数值数据：电影数量） --------------------------
  const yScale = $derived(
    d3.scaleLinear()
      .domain([0, d3.max(Object.values(genreNums)) || 0]) // Y轴数据范围：0到最多类型的电影数
      .range([usableArea.bottom, usableArea.top]) // Y轴像素范围：从下到上（反转SVG默认的从上到下）
      .nice() // 自动调整刻度为整数（更友好）
  );

  // 柱子宽度（由X轴比例尺的带宽决定）
  const xBarWidth: number = $derived(xScale.bandwidth());

  // -------------------------- 3. 补全坐标轴更新逻辑 --------------------------
  // let xAxis: SVGGElement | null = $state(); // X轴容器
  // let yAxis: SVGGElement | null = $state(); // Y轴容器
   let xAxis: SVGGElement | null = $state(null);
  let yAxis: SVGGElement | null = $state(null);


  function updateAxis() {
    // 渲染X轴（底部）：旋转文字避免重叠
    if (xAxis) {
      d3.select(xAxis)
        .call(d3.axisBottom(xScale))
        .selectAll("text")
        .attr("transform", "rotate(45)")
        .style("text-anchor", "start")
        .style("font-size", "12px");
    }
    // 渲染Y轴（左侧）：补充Y轴逻辑
    if (yAxis) {
      d3.select(yAxis)
        .call(d3.axisLeft(yScale)) // Y轴使用左侧坐标轴
        .selectAll("text")
        .style("font-size", "12px");
    }
  }

  // 当比例尺/数据变化时，自动更新坐标轴
  $effect(() => {
    updateAxis();
  });
</script>

<h3>
  The Distribution of Genres (
  {yearRange[0]?.getFullYear() || "N/A"} - {yearRange[1]?.getFullYear() || "N/A"}
  )
</h3>

{#if movies.length > 0}
  <svg {width} {height}>
    <!-- 柱子容器 -->
    <g class="bars">
      <!-- 遍历所有类型：[genre(类型名), cnt(数量)] -->
      {#each Object.entries(genreNums) as [genre, cnt]}
        <g class={`genre-bar ${selectedGenre === genre ? "active" : ""}`}>
          <!-- -------------------------- 4. 补全柱子绘制逻辑 -------------------------- -->
          <rect
            class="bar"
            width={xBarWidth}
            height={usableArea.bottom - yScale(cnt)} 
            x={xScale(genre)!} 
            y={yScale(cnt)} 
            fill="#449900"
            opacity={selectedGenre === genre ? 1 : 0.7} 
            onmouseover={() => (selectedGenre = genre)} 
            onmouseout={() => (selectedGenre = undefined)} 
          />

          <!-- 柱子顶部的数量标签 -->
          <text
            x={xScale(genre)! + xBarWidth / 2}
            y={yScale(cnt) - 5}
            font-size="12"
            text-anchor="middle"
            fill="#333"
          >
            {cnt}
          </text>
        </g>
      {/each}
    </g>

    <!-- X轴容器：定位到底部 -->
    <g transform={`translate(0, ${usableArea.bottom})`} bind:this={xAxis} />
    <!-- Y轴容器：定位到左侧 -->
    <g transform={`translate(${usableArea.left}, 0)`} bind:this={yAxis} />
  </svg>
{/if}

<style>
  .bar {
    transition: opacity 0.2s ease, height 0.2s ease; /* 动画过渡 */
    cursor: pointer;
  }
  .bar:hover {
    fill: #2e6600; /*  hover时加深颜色 */
  }
  text {
    pointer-events: none; /* 文字不拦截鼠标事件 */
  }
</style>