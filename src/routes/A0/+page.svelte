<script>
    let maxClick = $state(4);
    let cnt = $state(4); // tip: https://svelte.dev/docs/svelte/$state

    function onClick() {
        // tip: Since DOM (i.e., the webpage content) will automatically update based on values, [<p id="info">Remaining Number of Clicks: {cnt}</p>]
        // we only need to change the cnt number here.
        if (cnt > 0) {
            cnt = cnt - 1;
        }
    }
</script>

<h1>Welcome to SvelteKit</h1>
<p>
    Visit <a href="https://svelte.dev/docs/kit">svelte.dev/docs/kit</a> to read the
    documentation
</p>

<h1>Amvi's VIS Site</h1>
<img
    width="200px"
    src="https://i.pinimg.com/736x/b7/18/b7/b718b792b6266d4b8ab25dfc630908d3.jpg"
    alt="Fav image"
/>
<div>
    You can click up to
    <select
        bind:value={maxClick}
        onchange={(event) => {
            cnt = Number(event.currentTarget.value);
        }}
    >
        {#each [2, 4, 6] as optionNum}
            <option value={optionNum}>
                {optionNum}
            </option>
        {/each}
    </select>
    times
</div>
<button onclick={onClick}> Click Me </button>

<!-- tip: use {#if...} template syntax here (https://svelte.dev/docs/svelte/if) so that the content below will automatically update when cnt value changes -->
<!-- `the content below should only show when cnt >0` -->
{#if cnt > 0}
    <p id="info">Remaining Number of Clicks: {cnt}</p>
{:else}
    <p>No more clicks allowed</p>
{/if}

<style>
    body {
        font-family: Arial, Helvetica, sans-serif;
    }
    button {
        background-color: #44aa66;
        /* background-color: blue; */
        color: white;
        font-size: xx-large;
        padding: 10px 20px;
        border: none;
        cursor: pointer;
        border-radius: 5px;
    }
</style>
