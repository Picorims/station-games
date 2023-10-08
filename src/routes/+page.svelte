<script lang="ts">
    import Map from "$lib/Map.svelte"
    import { onMount } from "svelte";

    onMount(() => {
        // init();
    });
</script>

<h1>Station games</h1>

<Map></Map>

<style>
    :global(:root) {
        font-family: Verdana, Geneva, Tahoma, sans-serif;
        color: #222222;
    }
</style>