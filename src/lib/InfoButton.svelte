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

<script>
    import ExternalLink from "./ExternalLink.svelte";
    let showPopup = false;

    function togglePopup() {
        showPopup = !showPopup;
    }
</script>

<button on:click={togglePopup}>?</button>

{#if showPopup}
<div class="block-interact">
    <div class="popup">
        <button on:click={togglePopup} class="close-popup">X</button>
        <p>v1.0.2 - <ExternalLink url="https://github.com/Picorims/station-games/blob/main/CHANGELOGS.md" label="Changelogs"></ExternalLink></p>
        <h2>Welcome!</h2>
        <p>
            <strong>
                Your goal is to guess as much stations from the Île-De-France public
                transport network as you can!
            </strong>
            It is of course impossible to guess everything (or you are cheating ;)).
            But can you beat your friends and coworkers?
        </p>
        <p>
            <strong>
                This game is highly inspired from <ExternalLink url="https://memory.pour.paris" label="memory.pour.paris"></ExternalLink>,
                please check it out if you are more into metro stations, it is worth it!
            </strong>
        </p>
        <h3>How to play</h3>
        <p>
            Enter a station name in the input field. You can omit dashes ("-"), capital letters, accents and diacritics.
            Found stations will be shown on the map, with their name and corresponding line(s). They will be filled with
            the color of these lines.
            You can optionally show not found stations (in white) in the settings to help you find them.
        </p>
        <p>
            Because there are many stations, they are grouped for performance and readability.
            The color code is as follows:
        </p>
        <ul>
            <li><strong>green group:</strong> found;</li>
            <li><strong>white group:</strong> not found;</li>
            <li><strong>orange group:</strong> many stations not found.</li>
        </ul>
        <h3>Saving your progress</h3>
        <p>
            <strong>
                While your progress is saved to the browser's local storage, it is advised
                to export your save from time to time. Note that local data is NOT persistent
                if you enabled cache erasing on quit on your browser! The save does not contain
                any sensitive data, only your game progress.
            </strong>
        </p>
        <hr>
        <h2>Licenses</h2>
        <h3>Program</h3>
        <p>
            Copyright (C) 2023  Picorims - licensed under the GNU AGPL-3.0-or-later
        </p>
        <p>
            Source code available on GitHub: <ExternalLink url="https://github.com/Picorims/station-games"></ExternalLink>
        </p>
        <p>
            This game is not affiliated nor approved by Île-De-France Mobilités and its affiliates.
        </p>
        <h3>Data</h3>
        <p>
            Stations data (<code>stops_data.json</code>) is licensed under the ODbL
            and is a combination of the following sources:
        </p>
        <ul>
            <li>"Référentiel des arrêts : Arrêts", PRIM, snapshot from October 7th 2023, licensed under the Licence Ouverte v2.0 (Etalab)</li>
            <ul>
                <li><strong>link:</strong> <ExternalLink url="https://prim.iledefrance-mobilites.fr/fr/donnees-statiques/arrets"></ExternalLink></li>
                <li><strong>licence:</strong> <ExternalLink url="https://www.etalab.gouv.fr/wp-content/uploads/2017/04/ETALAB-Licence-Ouverte-v2.0.pdf"></ExternalLink></li>
            </ul>
            <li>"Référentiel des lignes de transport en commun d'île-de-France - lignes actives et prochainement actives", PRIM, snapshot from October 7th 2023, licensed under the ODbL (French version)</li>
            <ul>
                <li><strong>link:</strong> <ExternalLink url="https://prim.iledefrance-mobilites.fr/fr/donnees-statiques/referentiel-des-lignes"></ExternalLink></li>
                <li><strong>licence:</strong> <ExternalLink url="http://vvlibri.org/fr/licence/odbl-10/legalcode/unofficial"></ExternalLink></li>
            </ul>
            <li>"Arrêts et lignes associées", PRIM, snapshot from October 7th 2023, licensed under the ODbL (French version)</li>
            <ul>
                <li><strong>link:</strong> <ExternalLink url="https://prim.iledefrance-mobilites.fr/fr/donnees-statiques/arrets-lignes"></ExternalLink></li>
                <li><strong>licence:</strong> <ExternalLink url="http://vvlibri.org/fr/licence/odbl-10/legalcode/unofficial"></ExternalLink></li>
            </ul>
        </ul>
        <p>
            You can find the derived data and its processing code here: <ExternalLink url="https://github.com/Picorims/station-games-data"></ExternalLink>
        </p>
        <p>
            The map tiles come from OpenStreetMap.
        </p>
        <button on:click={togglePopup}>Close</button>
    </div>
</div>
{/if}

<style>
    div.block-interact {
        position: fixed;
        z-index: 10000;
        left: 0;
        top: 0;
        width: 100vw;
        height: 100vh;
        background-color: var(--c-black-transparent);
    }
    div.popup {
        position: absolute;
        left: 5vw;
        top: 8vh;
        width: 90vw;
        height: 84vh;
        padding: 2rem;
        padding-right: 6vw;
        overflow: auto;
        overflow-y: scroll;

        border: 5px solid var(--c-gray-0);
        border-radius: var(--p-large-radius);
        background-color: var(--c-gray-4);
    }
    button.close-popup {
        position: fixed;
        top: 10vh;
        right: 9vw;
        border: 4px solid var(--c-gray-4);
        border-radius: var(--p-medium-radius)
    }

    @media screen and (max-width: 640px) {
        div.popup {
            padding-right: 2rem;
        }
        button.close-popup {
            right: 15vw;
        }
    }
</style>