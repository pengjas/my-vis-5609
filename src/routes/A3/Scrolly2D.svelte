<script lang="ts">
  import type { TMovie } from "../../types";

  // define the props of this component
  type Props = {
    movies: TMovie[];
  };
  let { movies }: Props = $props();

  import { Scroll, Bar, RankBar } from "$lib"; // <-- 1. 在这里导入 RankBar
  import * as d3 from "d3";
  import { get } from "svelte/store";

  let myProgress = $state(0); // reactive variable for tracking the progress of the scrollytelling

  let yearRange = $derived(d3.extent(movies.map((d) => d.year))); // derive the year range from the movies data

  function getYearsBetweenDates(startDate?: Date, endDate?: Date) {
    if (startDate && endDate) {
      return Array.from(
        { length: endDate.getFullYear() - startDate.getFullYear() + 1 },
        (_, i) => {
          // Create a Date object for the first day of each year in the range
          return new Date(startDate.getFullYear() + i, 0, 1); // January 1st of the year
        },
      );
    }
    return [];
  }
  let years: Date[] = $derived(
    getYearsBetweenDates(yearRange[0], yearRange[1]),
  ); // derive the years from the year range

  // function getHighestRatedMovieEachYear(
  //   movies: TMovie[],
  //   years: Date[] | undefined,
  // ): TMovie[] {
  //   let highestRatedMovieEachYear = years
  //     ? years.map((currentYear) => {
  //         let moviesThisYear = movies.filter(
  //           (d) => d.year.getFullYear() == currentYear.getFullYear(),
  //         );
  //         let highestRatedMovie = moviesThisYear.reduce((max, current) => {
  //           return current.average_rating >= max.average_rating ? current : max;
  //         }, moviesThisYear[0]);
  //         return highestRatedMovie;
  //       })
  //     : [];

  //   highestRatedMovieEachYear = highestRatedMovieEachYear.filter((d) => d); // make sure no empty elements in array
  //   return highestRatedMovieEachYear;
  // }

  // let highestRatedMovieEachYear = $derived(
  //   getHighestRatedMovieEachYear(movies, years),
  // );

  const barChaertHeight = 500;
</script>

<h2>Scrolly with 2D visualization</h2>
<p>
  First, we demonstrate how to use the Scrolly element to update the genre
  distribution based on year, which is estimated based on the scrolly y position
  of the left text panel
</p>

<Scroll bind:progress={myProgress}>
            {#each years as date}
    <div class={date.getFullYear().toString()}>
      <h3 class="year">{date.getFullYear()}</h3>
            {#each movies.filter((d) => d.year.getFullYear() == date.getFullYear()) as movie}
                <b class="movie-title">{movie.primary_title}:</b>
        {movie.genres.join(" | ")}
        <br />
              {/each}
          </div>
  {/each}

    <div slot="viz">
    <Bar {movies} progress={myProgress} height={barChaertHeight} width={600} />
        <RankBar {movies} progress={myProgress} height={barChaertHeight} width={600} />
  </div>
</Scroll>

<style>
  .movie-title {
    color: #449900;
  }
  h3.year {
    margin-bottom: 0px;
  }
</style>