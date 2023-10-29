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

<header>
    <h1>Station games</h1>

    <div class="buttons">
        <InfoButton></InfoButton>
        <button on:click={() => {console.log(mapComponent);mapComponent.downloadSave()}}>Save</button>
        <input type="file" bind:this={hiddenInput} style="display: none;" on:change={() => {
            if (hiddenInput.files) mapComponent.uploadSave(hiddenInput.files[0]);
        }}/>
        <button on:click={() => {hiddenInput.click();}}>Open save</button>
    </div>
</header>

<main>
    <Map bind:this={mapComponent}></Map>
</main>

<style>
    :global(:root) {
        font-family: Verdana, Geneva, Tahoma, sans-serif;
        --c-black: #02111B;
        --c-black-transparent: #02111BBB;
        --c-gray-0: #2C3840;
        --c-gray-1: #555F66;
        --c-gray-2: #7F878C;
        --c-gray-3: #A9AEB1;
        --c-gray-4: #D2D5D7;
        --c-white: #FCFCFC;

        --c-blue-0: #134752;
        --c-blue-1: #247D8A;
        --c-blue-2: #35B3C1;
        --c-blue-3: #77CBD5;
        --c-blue-4: #BAE4E8;

        --c-orange: #FF8C42;
        --c-red: #eb6851;
        --c-green: #70ca75;
        --c-green-dark: #327736;
        --p-small-radius: 4px;
        --p-medium-radius: 8px;
        --p-large-radius: 16px;
        color: var(--c-black);
    }

    :global(body) {
        margin: 0;
        padding: 0;
        width: 100vw;
        height: 100vh;
        max-width: 100vw;
        max-height: 100vh;
        overflow: hidden;
        background-color: (--c-white);
        display: flex;
        flex-direction: column;
    }

    :global(*) {
        box-sizing: border-box;
    }

    :global(button) {
        margin: 0.2em;
        padding: 0.2em 0.5em;
        font-size: 1.2rem;
        font-weight: bold;
        color: var(--c-black);
        background-color: var(--c-blue-3);
        border: none;
        border-radius: var(--p-small-radius);
    }
    :global(button:hover) {
        cursor: pointer;
        background-color: var(--c-blue-2);
    }

    :global(input[type="text"]) {
        margin: 0.2em;
        padding: 0.2em 0.5em;
        font-size: 1.2rem;
        color: var(--c-black);
        background-color: var(--c-white);
        border: 4px solid var(--c-gray-0);
        border-radius: var(--p-medium-radius);
    }
    :global(input[type="text"][disabled]) {
        border-color: var(--c-gray-2);
    }

    header {
        width: 100vw;
        display: flex;
        flex-wrap: wrap;
        justify-content: space-between;
        align-items: center;
        background-color: var(--c-black);
        padding: 0.5rem;
    }

    h1 {
        color: var(--c-white);
        margin: 0;
        margin-right: 0.5rem;
    }

    div.buttons {
        display: flex;
        justify-content: center;
        align-items: center;
    }

    @media screen and (max-width: 640px) {
        h1 {
            width: 100%;
            text-align: center;
        }
        div.buttons {
            width: 100%;
        }
    }

    div.buttons > button {
        flex: 0 0 auto;
    }

    main {
        flex: 1 1 auto;
    }
</style>