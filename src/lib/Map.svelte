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
    import * as La from "leaflet";
    import "leaflet/dist/leaflet.css";
    import * as Lb from "leaflet.markercluster";
    const L = {...La, ...Lb};
    import "leaflet.markercluster/dist/MarkerCluster.css";
    import "leaflet.markercluster/dist/MarkerCluster.Default.css";
    import dataI from "$lib/stops_data.json";

    const data = dataI as unknown as Record<string, {
        "route_id": string,
        "route_long_name": string,
        "stop_id": number,
        "stop_name": string,
        "stop_lon": number,
        "stop_lat": number,
        "area_id": number,
        "town": string,
        "postal_region": number,
        "route_short_name": string,
        "transport_mode": string,
        "transport_submode": string,
        "operator_name": string,
        "background_color": string,
        "text_color": string
    }>
    let progressBar: HTMLProgressElement;
    let notFoundMarkersCheckbox: HTMLInputElement;
    let stationInput: HTMLInputElement;

    let leaflet: {
        map: La.Map | null,
        markersCluster: La.MarkerClusterGroup | null,
        foundMarkersCluster: La.MarkerClusterGroup | null,
        markersCache: Record<string, {found: boolean, marker: La.Marker}>,
        foundMarkersCache: Record<string, La.Marker>,
    } = {
        map: null,
        markersCluster: null,
        foundMarkersCluster: null,
        markersCache: {},
        foundMarkersCache: {},
    }

    interface Save {
        version: 1,
        foundKeywords: string[],
        foundStations: string[], // ID list
    }
    let save: Save = {
        version: 1,
        foundKeywords: [],
        foundStations: [],
    }

    let nbFoundMarkers = 0;
    let nbMarkers = 0;
    let submitMsg = "Enter a value above";
    
    /**
     * Init leaflet map
     * @param node
     */
    function init(node: HTMLDivElement): void {
        console.time("init");
        node = node; // mute error;
        const parisCenter: L.LatLngTuple = [48.86296891769729, 2.3394514484316247];
        leaflet.map = L.map("map").setView(parisCenter, 13);
        L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            minZoom: 9,
            attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        }).addTo(leaflet.map);
        leaflet.map.setMaxBounds([[parisCenter[0]-0.8, parisCenter[1]-0.9],[parisCenter[0]+1, parisCenter[1]+0.8]]);

        leaflet.markersCluster = L.markerClusterGroup({chunkedLoading: true, chunkProgress: updateProgress});
        leaflet.foundMarkersCluster = L.markerClusterGroup({chunkedLoading: true, chunkProgress: updateProgress});

        let i = 0;
        for (const id in data) {
            leaflet.markersCache[id] = {marker: getMarker(id), found: false};
            i++;
        }

        nbMarkers = i; // update UI once

        leaflet.map.addLayer(leaflet.markersCluster);
        leaflet.map.addLayer(leaflet.foundMarkersCluster);
        updateNotFoundMarkers();
        console.timeEnd("init");
    }

    /**
     * Update loading progress of markers
     * @param processed
     * @param total
     */
    function updateProgress(processed: number, total: number/*, elapsed, layersArray*/): void {
        if (progressBar) progressBar.value = (processed/total) * 100;
    }

    /**
     * Create leaflet marker
     * @param id
     */
    function getMarker(id: string, blank: boolean = true): La.Marker {
        const name = data[id].stop_name + " - ligne " + data[id].route_long_name;

        const div = document.createElement("div");
        div.style.width = "100%";
        div.style.height = "100%";  
        if (!blank) div.style.backgroundColor = "#" + data[id].background_color;

        const marker = L.marker(L.latLng(data[id].stop_lat, data[id].stop_lon), {
            icon: L.divIcon({html: div}),
            title: blank ? "not found" : name,
        });
        if (!blank) marker.bindTooltip(name, {className: "station-tooltip"});

        return marker;
    }

    function updateNotFoundMarkers(): void {
        if (notFoundMarkersCheckbox?.checked) {
            leaflet.markersCluster?.addLayers(
                Object.values(leaflet.markersCache)
                    .filter(v => !v.found)
                    .map((v) => v.marker)
            );
        } else {
            leaflet.markersCluster?.clearLayers();
        }
    }

    /**
     * Search stations matching keywords and add them to the map
     * @param e
     */
    function searchStation(e: SubmitEvent) {
        console.log("searching " + e);
        e.preventDefault();
        
        // not search if already searched
        const keywords = stationInput.value;
        for (const search of save.foundKeywords) {
            if (stationsEqual(search, keywords)) {
                submitMsg = "Already found";
                return;
            }
        }
        
        // search
        let found = false;
        let firstFound = true;
        for (const id in data) {
            if (stationsEqual(data[id].stop_name, keywords)) {
                found = true;
                if (firstFound) {
                    save.foundKeywords.push(keywords);
                    submitMsg = "Found!";
                }
                addStation(id);

                firstFound = false;
            }
        }
        if (!found) submitMsg = "Not found";
        stationInput.value = "";
    }

    /**
     * Determine if two keywords are considered equivalent, by cleaning them up beforehand.
     * @param a
     * @param b
     */
    function stationsEqual(a: string, b: string): boolean {
        const transform = (v: string) => v.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase();
        return transform(a) === transform(b);
        //FIXME: optimize to cache data strings and only transform once input string
    }

    /**
     * Add a station as found to the save and the map
     * @param id
     */
    function addStation(id: string): void {
        for (const foundId of save.foundStations) {
            if (id === foundId) return; // already added
        }

        nbFoundMarkers++;
        save.foundStations.push(id);
        addFoundMarker(id);
    }

    /**
     * Convert a station to a found station on Leaflet
     */
    function addFoundMarker(id :string) {
        console.log("showing " + id + " as found");
        const marker = getMarker(id, false);
        leaflet.foundMarkersCache[id] = marker;
        leaflet.foundMarkersCluster?.addLayer(marker);
        
        leaflet.markersCache[id].found = true;
        leaflet.markersCluster?.removeLayer(leaflet.markersCache[id].marker);
    }

    /**
     * Export save to JSON
     */
    export function downloadSave() {
        console.log("downloading save");
        const blob = new Blob([JSON.stringify(save)], {type: "application/json"});
        const link = document.createElement("a");
        
        const date = new Date();
        const y = date.getFullYear();
        const m = date.getMonth();
        const d = date.getDay();
        const h = date.getHours();
        const mn = date.getMinutes();
        const s = date.getSeconds();
        
        link.download = `station_games_save_${y}_${m}_${d}_${h}_${mn}_${s}`;
        link.href = window.URL.createObjectURL(blob);
        link.dataset.downloadurl = ["application/json", link.download, link.href].join(":");

        const evt = new MouseEvent("click", {
            view: window,
            bubbles: true,
            cancelable: true,
        });

        link.dispatchEvent(evt);
        link.remove();
    }

    /**
     * load external json file
     */
    export function uploadSave(file: Blob) {
        console.log("loading save");
        file.text().then((text) => {
            const tmpSave: Save = JSON.parse(text);
            if (!tmpSave.foundKeywords) {
                showSaveLoadError("missing keywords");
            }
            if (!tmpSave.foundStations) {
                showSaveLoadError("missing station IDs");
            }
            if (save.version !== 1) {
                showSaveLoadError("unsupported version");
            }
            if (Object.keys(tmpSave).length > 3) {
                showSaveLoadError("unknown fields");
            }
            save = tmpSave;
            reloadMarkersFromSave();
        });
    }

    function showSaveLoadError(context: string = "unknown") {
        alert("Invalid save file! Make sure you uploaded the right file.\n\nContext: " + context + ".");
    }

    /**
     * Mark all markers as not found, then mark every
     * marker in the save as found and add them.
     */
    function reloadMarkersFromSave() {
        console.log("updating the map");
        nbFoundMarkers = save.foundStations.length;

        // mark all as not found
        leaflet.foundMarkersCluster?.clearLayers();
        leaflet.foundMarkersCache = {};
        for (let m in leaflet.markersCache) {
            leaflet.markersCache[m].found = false;
        }

        // show found stations
        for (const id of save.foundStations) {
            addFoundMarker(id);
        }
        console.log("map update done.");
    }
</script>

<progress bind:this={progressBar} max="100" value="0"></progress>

<label for="show-not-found">Show not found stations</label>
<input id="show-not-found" type="checkbox" on:change={updateNotFoundMarkers} bind:this={notFoundMarkersCheckbox}/>

<form action="" on:submit={searchStation}>
    <input type="text" name="station" id="station-input" bind:this={stationInput}>
    <button type="submit">Submit</button>
</form>

<p>{submitMsg} | Found stops: {nbFoundMarkers} / {nbMarkers}</p>

<div id="map" use:init></div>


<style>
    #map {
        height: 70vh;
    }
    :global(div.leaflet-div-icon) {
        border: 2px solid black;
    }
    :global(.station-tooltip) {
        font-size: 1rem;
    }
</style>