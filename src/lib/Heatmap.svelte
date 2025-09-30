<script lang="ts">
  import * as d3 from "d3";
  
  type HeatmapData = {
    topGenres: string[];
    matrix: number[][];
  };
  
  type Props = {
    data: HeatmapData;
    width?: number;
    height?: number;
  };
  
  let { data, width = 700, height = 600 }: Props = $props();
  
  const margin = { top: 100, right: 20, bottom: 100, left: 100 };
  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;
  
  // Color scale
  const colorScale = $derived(d3.scaleSequential(d3.interpolateBlues)
    .domain([0, d3.max(data.matrix.flat()) || 1]));
  
  // Cell size
  const cellSize = $derived(Math.min(
    innerWidth / data.topGenres.length,
    innerHeight / data.topGenres.length
  ));
</script>

<div class="chart-container">
  <h3>Genre Co-occurrence Heatmap</h3>
  <svg {width} {height}>
    <g transform={`translate(${margin.left}, ${margin.top})`}>
      <!-- Heatmap cells -->
      {#each data.topGenres as genreY, i}
        {#each data.topGenres as genreX, j}
          <rect 
            x={j * cellSize} 
            y={i * cellSize} 
            width={cellSize} 
            height={cellSize} 
            fill={colorScale(data.matrix[i][j])}
            stroke="#fff"
          />
          {#if data.matrix[i][j] > 0}
            <text 
              x={j * cellSize + cellSize / 2} 
              y={i * cellSize + cellSize / 2} 
              text-anchor="middle" 
              dy="0.35em" 
              font-size="10"
              fill={data.matrix[i][j] > (d3.max(data.matrix.flat()) || 0) / 2 ? "#fff" : "#000"}
            >
              {data.matrix[i][j]}
            </text>
          {/if}
        {/each}
      {/each}
      
      <!-- Y-axis labels -->
      {#each data.topGenres as genre, i}
        <text 
          x={-5} 
          y={i * cellSize + cellSize / 2} 
          text-anchor="end" 
          dy="0.35em" 
          font-size="11"
        >
          {genre}
        </text>
      {/each}
      
      <!-- X-axis labels -->
      {#each data.topGenres as genre, i}
        <text 
          x={i * cellSize + cellSize / 2} 
          y={innerHeight + 15} 
          text-anchor="end" 
          font-size="11"
          transform={`rotate(-45, ${i * cellSize + cellSize / 2}, ${innerHeight + 15})`}
        >
          {genre}
        </text>
      {/each}
      
      <!-- Title -->
      <text x={innerWidth / 2} y={-20} text-anchor="middle" font-size="14" font-weight="bold">
        Genre Co-occurrence Matrix
      </text>
      <text x={innerWidth / 2} y={-5} text-anchor="middle" font-size="11" fill="#666">
        (Cell values show number of movies with both genres)
      </text>
      
      <!-- Color legend -->
      <g transform={`translate(${innerWidth + 10}, 0)`}>
        {#each [0, 0.25, 0.5, 0.75, 1] as t}
          <rect 
            y={t * 100} 
            width={15} 
            height={25} 
            fill={colorScale(t * (d3.max(data.matrix.flat()) || 1))}
          />
          <text 
            x={25} 
            y={t * 100 + 12} 
            font-size="10"
          >
            {Math.round(t * (d3.max(data.matrix.flat()) || 0))}
          </text>
        {/each}
      </g>
    </g>
  </svg>
</div>

<style>
  .chart-container {
    margin: 20px 0;
  }
  
  text {
    user-select: none;
  }
</style>