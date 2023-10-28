<!--
    Station games is a game about guessing stations from a public transport network.
    Copyright (C) 2023  Picorims<picorims.contact@gmail.com>

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU Affero General Public License as published
    by the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU Affero General Public License for more details.

    You should have received a copy of the GNU Affero General Public License
    along with this program.  If not, see <https://www.gnu.org/licenses/>.
-->

<script lang="ts">
    import InfoButton from "$lib/InfoButton.svelte";
    import Map from "$lib/Map.svelte"

    let mapComponent: Map;
    let hiddenInput: HTMLInputElement;
</script>

<h1>Station games</h1>
<InfoButton></InfoButton>

<button on:click={() => {console.log(mapComponent);mapComponent.downloadSave()}}>Save</button>
<input type="file" bind:this={hiddenInput} style="display: none;" on:change={() => {
    if (hiddenInput.files) mapComponent.uploadSave(hiddenInput.files[0]);
}}/>
<button on:click={() => {hiddenInput.click();}}>Open save</button>

<Map bind:this={mapComponent}></Map>

<style>
    :global(:root) {
        font-family: Verdana, Geneva, Tahoma, sans-serif;
        color: #222222;
    }
</style>