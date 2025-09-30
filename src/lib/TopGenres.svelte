<script lang="ts">
  import * as d3 from "d3";
  
  type TopGenreData = {
    year: number;
    topGenres: { genre: string; count: number; rank: number }[];
  };
  
  type Props = {
    data: TopGenreData[];
    width?: number;
    height?: number;
  };
  
  let { data, width = 800, height = 500 }: Props = $props();
  
  const margin = { top: 40, right: 120, bottom: 50, left: 60 };
  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;
  
  // Extract all genres
  const allGenres = $derived(Array.from(new Set(data.flatMap(d => 
    d.topGenres.map(g => g.genre)
  ))));
  
  // Color scale
  const colorScale = $derived(d3.scaleOrdinal(d3.schemeCategory10).domain(allGenres));
  
  // Line data
  const lineData = $derived(allGenres.map(genre => ({
    genre,
    values: data.map(d => {
      const genreData = d.topGenres.find(g => g.genre === genre);
      return {
        year: d.year,
        count: genreData ? genreData.count : 0,
        rank: genreData ? genreData.rank : null
      };
    })
  })));
  
  // Scales
  const xScale = $derived(d3.scaleLinear()
    .domain(d3.extent(data, d => d.year) as [number, number])
    .range([0, innerWidth])
    .nice());
  
  const yScale = $derived(d3.scaleLinear()
    .domain([0, d3.max(data, d => d3.max(d.topGenres, g => g.count)) || 0])
    .range([innerHeight, 0])
    .nice());
  
  // Line generator
  const lineGenerator = $derived(d3.line<{year: number; count: number}>()
    .x(d => xScale(d.year))
    .y(d => yScale(d.count))
    .curve(d3.curveMonotoneX));
</script>

<div class="chart-container">
  <h3>Annual Top 3 Genre Trends</h3>
  <svg {width} {height}>
    <g transform={`translate(${margin.left}, ${margin.top})`}>
      <!-- Grid lines -->
      <g class="grid">
        {#each yScale.ticks(5) as tick}
          <line x1={0} x2={innerWidth} y1={yScale(tick)} y2={yScale(tick)} stroke="#eee" />
          <text x={-5} y={yScale(tick)} text-anchor="end" dy="0.32em" font-size="10">{tick}</text>
        {/each}
      </g>
      
      <!-- Lines -->
      {#each lineData as line}
        <path 
          d={lineGenerator(line.values.filter(v => v.rank !== null))} 
          fill="none" 
          stroke={colorScale(line.genre)} 
          stroke-width={2}
        />
      {/each}
      
      <!-- Data points -->
      {#each lineData as line}
        {#each line.values as point}
          {#if point.rank !== null}
            <circle 
              cx={xScale(point.year)} 
              cy={yScale(point.count)} 
              r={4} 
              fill={colorScale(line.genre)}
              stroke="#fff" 
              stroke-width={1.5}
            />
            <!-- <text 
              x={xScale(point.year) + 5} 
              y={yScale(point.count) - 5} 
              font-size="10" 
              fill={colorScale(line.genre)}
            >
              Rank {point.rank}
            </text> -->
          {/if}
        {/each}
      {/each}
      
      <!-- Axes -->
      <g transform={`translate(0, ${innerHeight})`} class="x-axis">
        {#each xScale.ticks(10) as tick}
          <g transform={`translate(${xScale(tick)}, 0)`}>
            <line y2={6} stroke="#000" />
            <text y={20} text-anchor="middle" font-size="11">{tick}</text>
          </g>
        {/each}
        <text x={innerWidth / 2} y={40} text-anchor="middle" font-size="12">Year</text>
      </g>
      
      <!-- <text x={-30} y={innerHeight / 2} text-anchor="middle" transform="rotate(-90)" font-size="12">
        Number of Movies
      </text> -->
      <text x={-40} y={innerHeight / 2} text-anchor="middle" transform="rotate(-90, -40, {innerHeight / 2})" font-size="12">
  Number of Movies
</text>
      
      <!-- Legend -->
      <g class="legend" transform={`translate(${innerWidth + 10}, 0)`}>
        {#each allGenres as genre, i}
          <g transform={`translate(0, ${i * 20})`}>
            <rect width={12} height={12} fill={colorScale(genre)} />
            <text x={20} y={10} font-size="11">{genre}</text>
          </g>
        {/each}
      </g>
    </g>
  </svg>
</div>

<style>
  .chart-container {
    margin: 20px 0;
  }
  
  .grid text {
    fill: #666;
    font-size: 10px;
  }
  
  .x-axis text {
    fill: #333;
  }
  
  .legend text {
    fill: #333;
  }
</style>