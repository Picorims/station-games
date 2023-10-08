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
    <div>
        <h2>Welcome!</h2>
        <p>
            Your goal is to guess as much stations from the Île-De-France public
            transport network as you can!
            It is of course impossible to guess everything (or you are cheating ;)).
            But can you beat your friends and coworkers?
        </p>
        <h2>Licenses</h2>
        <h3>Program</h3>
        <p>
            Copyright (C) 2023  Picorims - licensed under the GNU AGPL-3.0-or-later
        </p>
        <p>
            Source code available on GitHub: <ExternalLink url="https://github.com/Picorims/station-games"></ExternalLink>
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
{/if}