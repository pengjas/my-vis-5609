<script>
    let maxClick = $state(2);
    let cnt = $state(maxClick);
    
    function onClick() {
        if (cnt > 0) cnt -= 1;
    }

    $effect(() => {
        // Reset count when maxClick changes
        cnt = maxClick;
    });
</script>

<h1>Pengju Liu's VIS Site</h1>
<img
    width="200px"
    src="https://en.wikipedia.org/wiki/Minnesota_Golden_Gophers#/media/File:Minnesota_Golden_Gophers_logo.svg"
/>
<div>
    You can click up to
    <select bind:value={maxClick}>
        {#each [2, 4, 6] as optionNum}
            <option value={optionNum}>
                {optionNum}
            </option>
        {/each}
    </select>
    times
</div>
<button on:click={onClick}> Click Me </button>

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
        color: white;
        font-size: xx-large;
        padding: 10px 20px;
        border: none;
        cursor: pointer;
        border-radius: 5px;
    }
</style>