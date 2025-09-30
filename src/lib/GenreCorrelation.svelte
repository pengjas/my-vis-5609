<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = { movies: TMovie[] };
  let { movies }: Props = $props();

  const width = 600;
  const height = 600;
  const margin = { top: 100, right: 20, bottom: 20, left: 100 };

  const correlationData = $derived(() => {
    if (movies.length === 0) return { genres: [], matrix: [] };

    // 1. Get all unique genres
    const genreSet = new Set<string>();
    for (const movie of movies) {
        const genres = movie.genres.split(',').map(g => g.trim());
        for (const genre of genres) {
            if (genre) genreSet.add(genre);
        }
    }
    const allGenres = Array.from(genreSet).sort();
    
    // 2. Create a co-occurrence matrix
    const matrix = new Map<string, Map<string, number>>();
    for (const g1 of allGenres) {
        matrix.set(g1, new Map());
        for (const g2 of allGenres) {
            matrix.get(g1)!.set(g2, 0);
        }
    }

    // 3. Populate the matrix
    for (const movie of movies) {
        const genres = movie.genres.split(',').map(g => g.trim()).filter(g => g);
        for (let i = 0; i < genres.length; i++) {
            for (let j = i + 1; j < genres.length; j++) {
                const g1 = genres[i];
                const g2 = genres[j];
                matrix.get(g1)!.set(g2, matrix.get(g1)!.get(g2)! + 1);
                matrix.get(g2)!.set(g1, matrix.get(g2)!.get(g1)! + 1);
            }
        }
    }
    
    // 4. Flatten for Svelte's #each loop and find max value
    let flatData = [];
    let maxCount = 0;
    for (const g1 of allGenres) {
        for (const g2 of allGenres) {
            if (g1 <= g2) { // Only need one half of the matrix
                const count = matrix.get(g1)!.get(g2)!;
                flatData.push({ g1, g2, count });
                if (count > maxCount && g1 !== g2) {
                    maxCount = count;
                }
            }
        }
    }

    return { genres: allGenres, matrix: flatData, maxCount };
  });

  const xScale = $derived(d3.scaleBand()
    .domain(correlationData.genres)
    .range([margin.left, width - margin.right])
    .padding(0.05));

  const yScale = $derived(d3.scaleBand()
    .domain(correlationData.genres)
    .range([margin.top, height - margin.bottom])
    .padding(0.05));

  const colorScale = $derived(d3.scaleSequential(d3.interpolateGreens)
    .domain([0, correlationData.maxCount]));

</script>

<h4>Q2: What are the correlations between different genres?</h4>
<svg {width} {height}>
  {#each correlationData.matrix as d}
    {@const x = xScale(d.g1)}
    {@const y = yScale(d.g2)}
    {@const x_rev = xScale(d.g2)}
    {@const y_rev = yScale(d.g1)}
    
    {#if x !== undefined && y !== undefined}
      <rect {x} {y} width={xScale.bandwidth()} height={yScale.bandwidth()} fill={colorScale(d.count)}>
        <title>{d.g1} & {d.g2}: {d.count}</title>
      </rect>
    {/if}
    {#if x_rev !== undefined && y_rev !== undefined && d.g1 !== d.g2}
       <rect x={x_rev} y={y_rev} width={xScale.bandwidth()} height={yScale.bandwidth()} fill={colorScale(d.count)}>
         <title>{d.g1} & {d.g2}: {d.count}</title>
       </rect>
    {/if}
  {/each}
  
  <g transform="translate({margin.left}, 0)">
    {#each yScale.domain() as tick}
      <text y={yScale(tick) + yScale.bandwidth() / 2} x="-5" text-anchor="end" dominant-baseline="middle" font-size="10">
        {tick}
      </text>
    {/each}
  </g>
  
  <g transform="translate(0, {margin.top})">
     {#each xScale.domain() as tick}
      <text x={xScale(tick) + xScale.bandwidth() / 2} y="-5" text-anchor="middle" dominant-baseline="auto" font-size="10" transform="rotate(-45, {xScale(tick) + xScale.bandwidth() / 2}, -5)">
        {tick}
      </text>
    {/each}
  </g>
</svg>
<p><b>Answer:</b> This heatmap shows the number of times two genres appear in the same movie. Darker green squares indicate a stronger correlation. By finding "Comedy" on an axis, you can see it frequently co-occurs with <b>Drama</b> and <b>Romance</b>, as indicated by the dark cells at their intersections. Similarly, <b>Action</b> has strong correlations with <b>Adventure</b> and <b>Thriller</b>.</p>