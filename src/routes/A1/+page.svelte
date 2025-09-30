<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { TMovie } from "../../types";
  import Bar from "$lib/Bar.svelte";
  import TopGenres from "$lib/TopGenres.svelte";
  import Heatmap from "$lib/Heatmap.svelte";
  
  let movies: TMovie[] = [];
  let top3PerYear: ReturnType<typeof processTop3PerYear> = [];
  let cooccurrenceData: ReturnType<typeof processGenreCooccurrence> = { topGenres: [], matrix: [] };

  // Process annual top 3 genres
  function processTop3PerYear(movies: TMovie[]) {
    const years = d3.group(movies, d => d.year.getFullYear());
    
    const result = [];
    for (const [year, yearMovies] of years) {
      const genreCounts: {[genre: string]: number} = {};
      yearMovies.forEach(movie => {
        movie.genres.forEach(genre => {
          genreCounts[genre] = (genreCounts[genre] || 0) + 1;
        });
      });
      
      const top3 = Object.entries(genreCounts)
        .sort((a, b) => b[1] - a[1])
        .slice(0, 3)
        .map(([genre, count], index) => ({
          genre,
          count,
          rank: index + 1
        }));
      
      result.push({
        year: year,
        topGenres: top3
      });
    }
    
    return result.sort((a, b) => a.year - b.year);
  }

  // Process genre co-occurrence matrix
  function processGenreCooccurrence(movies: TMovie[], topN: number = 15) {
    const genreCounts: {[genre: string]: number} = {};
    movies.forEach(movie => {
      movie.genres.forEach(genre => {
        genreCounts[genre] = (genreCounts[genre] || 0) + 1;
      });
    });
    
    const topGenres = Object.entries(genreCounts)
      .sort((a, b) => b[1] - a[1])
      .slice(0, topN)
      .map(([genre]) => genre);
    
    const matrix = topGenres.map(genreA => 
      topGenres.map(genreB => {
        let cooccurrence = 0;
        movies.forEach(movie => {
          const hasA = movie.genres.includes(genreA);
          const hasB = movie.genres.includes(genreB);
          if (hasA && hasB) cooccurrence++;
        });
        return cooccurrence;
      })
    );
    
    return { topGenres, matrix };
  }

  async function loadCsv() {
    try {
      const csvUrl = "./summer_movies.csv";
      movies = await d3.csv(csvUrl, (row) => {
        return {
          tconst: row.tconst,
          title_type: row.title_type,
          primary_title: row.primary_title,
          original_title: row.original_title,
          runtime_minutes: row.runtime_minutes ? Number(row.runtime_minutes) : 0,
          average_rating: row.average_rating ? Number(row.average_rating) : 0,
          num_votes: row.num_votes ? Number(row.num_votes) : 0,
          genres: row.genres ? row.genres.split(",") : [],
          year: row.year ? new Date(Number(row.year), 0, 1) : new Date(),
        };
      });

      top3PerYear = processTop3PerYear(movies);
      cooccurrenceData = processGenreCooccurrence(movies, 15);

      console.log("Data loaded successfully");
    } catch (error) {
      console.error("Error loading CSV:", error);
    }
  }

  onMount(loadCsv);
</script>

<h1>Summer Movies Data Analysis</h1>
<p>Dataset contains {movies.length === 0 ? "..." : `${movies.length} `} movies</p>

<!-- Genre Distribution -->
<h2>1. Genre Distribution</h2>
<Bar {movies} />

<!-- Q1: Annual Top 3 Genre Trends -->
<h2>2. Annual Top 3 Genre Trends</h2>
{#if top3PerYear.length > 0}
  <TopGenres data={top3PerYear} />
  <div class="insight">
    <h4>Analysis Insights:</h4>
    <p>The line chart shows that Drama genre dominates in most years, while Comedy maintains strong presence in summer movies. Some genres like Romance show significant fluctuations in specific years.</p>
  </div>
{/if}

<!-- Q2: Genre Correlation -->
<h2>3. Genre Correlation Analysis</h2>
{#if cooccurrenceData.topGenres.length > 0}
  <Heatmap data={cooccurrenceData} />
  <div class="insight">
    <h4>Analysis Insights:</h4>
    <p>The heatmap reveals strong co-occurrence between Comedy and Romance/Drama genres. Darker diagonal cells indicate single-genre movie counts, while off-diagonal cells show genre combination popularity.</p>
  </div>
{/if}

<style>
  h2 {
    border-bottom: 2px solid #449900;
    padding-bottom: 5px;
    margin-top: 40px;
  }
  
  .insight {
    background: #f9f9f9;
    padding: 15px;
    border-left: 4px solid #449900;
    margin: 20px 0;
  }
  
  .insight h4 {
    margin: 0 0 10px 0;
    color: #2e6600;
  }
</style>