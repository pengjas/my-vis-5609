<script lang="ts">
  import * as d3 from 'd3';
  import type { SvelteComponentTyped } from 'svelte';

  // 接收的props：年度Top3类型数据
  let { data = [] } = $props<{
    data: { year: number; genre: string; count: number; rank: number }[];
  }>();
  // 图表尺寸与边距
  const width = 800;
  const height = 400;
  const margin = { top: 20, right: 50, bottom: 50, left: 50 };
  const usableWidth = width - margin.left - margin.right;
  const usableHeight = height - margin.top - margin.bottom;

  // 提取所有类型（用于颜色映射）
  const genres = Array.from(new Set(data.map(d => d.genre)));
  // 颜色比例尺（分类颜色，区分不同类型）
  const color = d3.scaleOrdinal(d3.schemeCategory10).domain(genres);

  // X轴比例尺（年份，线性）
  const xScale = d3.scaleLinear()
    .domain(d3.extent(data, d => d.year) as [number, number])
    .range([0, usableWidth])
    .nice(); // 优化刻度为整数

  // Y轴比例尺（数量，线性）
  const yScale = d3.scaleLinear()
    .domain([0, d3.max(data, d => d.count) as number])
    .range([usableHeight, 0])
    .nice();

  // 折线生成器（连接当年进Top3的点）
  const line = d3.line()
    .x(d => xScale(d.year))
    .y(d => yScale(d.count))
    .curve(d3.curveMonotoneX); // 平滑曲线

  // 按类型分组数据（方便绘制折线）
  const genreData = d3.group(data, d => d.genre);

  // 坐标轴元素（用于绑定D3轴）
  let xAxisElem: SVGGElement | null = null;
  let yAxisElem: SVGGElement | null = null;

  // 当数据/比例尺变化时，更新坐标轴
  $effect(() => {
    if (xAxisElem) d3.select(xAxisElem).call(d3.axisBottom(xScale));
    if (yAxisElem) d3.select(yAxisElem).call(d3.axisLeft(yScale));
  });
</script>

<svg width={width} height={height}>
  <!-- 主绘图区域（扣除边距） -->
  <g transform={`translate(${margin.left}, ${margin.top})`}>
    <!-- 折线：每个类型1条 -->
    {#each genreData as [genre, points]}
      <path 
        d={line(points)} 
        stroke={color(genre)} 
        fill="none" 
        stroke-width={2}
        opacity={0.7}
      />
    {/each}

    <!-- 点+排名标签：每个Top3类型1个 -->
    {#each data as d}
      <circle 
        cx={xScale(d.year)} 
        cy={yScale(d.count)} 
        r={4} 
        fill={color(d.genre)}
        stroke="white"
        stroke-width={1}
        cursor="pointer"
      />
      <text 
        x={xScale(d.year)} 
        y={yScale(d.count) - 10} 
        text-anchor="middle" 
        font-size="12"
        fill={color(d.genre)}
      >
        {d.rank}
      </text>
    {/each}

    <!-- X轴（底部） -->
    <g 
      bind:this={xAxisElem}
      transform={`translate(0, ${usableHeight})`}
      class="axis"
    />

    <!-- Y轴（左侧） -->
    <g 
      bind:this={yAxisElem}
      class="axis"
    />

    <!-- X轴标签 -->
    <text 
      x={usableWidth/2} 
      y={usableHeight + 40} 
      text-anchor="middle" 
      font-size="14"
    >
      Year
    </text>

    <!-- Y轴标签 -->
    <text 
      x={-usableHeight/2} 
      y={-40} 
      transform="rotate(-90)"
      text-anchor="middle" 
      font-size="14"
    >
      Number of Movies
    </text>
  </g>
</svg>

<style>
  .axis path, .axis line {
    stroke: #ccc;
  }
  .axis text {
    font-size: 12px;
    fill: #666;
  }
</style>