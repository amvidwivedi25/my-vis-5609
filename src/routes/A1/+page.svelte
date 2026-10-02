
<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { TMovie } from "../../types";
  import Bar from "$lib/Bar.svelte";
  import TopGenres from "$lib/TopGenres.svelte";
  import GenreMatrix from "$lib/GenreMatrix.svelte";


  // Reactive variable for storing the data
  let movies: TMovie[] = $state([]);

  // Function to load the CSV
  async function loadCsv() {
    try {
      const csvUrl = "/summer_movies.csv";
      movies = await d3.csv(csvUrl, (row) => {
        // TIP: in row, all values are strings, so we need to use a row conversion function here to format them
        return {
          tconst: row.tconst,
          title_type: row.title_type,
          primary_title: row.primary_title,
          original_title: row.original_title,
          year: new Date(Number(row.year), 0),
          runtime_minutes: Number(row.runtime_minutes),
          genres: row.genres ? row.genres.split(",") : [],
          average_rating: Number(row.average_rating),
          num_votes: Number(row.num_votes)
          // ...row, // spread syntax to copy all properties from row
          // num_votes: Number(row.num_votes),
          // year: new Date(row.year),
          // please also format the values for other non-string attributes. You can check the attributes in the CSV file
        };
      });

      console.log("Loaded CSV Data:", movies);
    } catch (error) {
      console.error("Error loading CSV:", error);
    }
  }
  // Call the loader when the component mounts
  onMount(loadCsv);
</script>

<h1>Summer Movies</h1>

<p>Here are {movies.length == 0 ? "..." : movies.length + " "} movies</p>
<Bar {movies} />
<TopGenres {movies} />
<GenreMatrix {movies} />