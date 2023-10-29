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
    import * as Lf from "leaflet";
    import "leaflet/dist/leaflet.css";
    import * as Lm from "leaflet.markercluster";
    // const L = {...Lf, ...Lm};
    import "leaflet.markercluster/dist/MarkerCluster.css";
    import "leaflet.markercluster/dist/MarkerCluster.Default.css";
    // import dataI from "$lib/stops_data.json";

    let data: Record<string, {
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
    }> = {};
    let progressBar: HTMLProgressElement;
    let hideNotFoundMarkers = true;
    let stopsFusioned = true;
    let stationInput: HTMLInputElement;
    let loading = true;

    let leaflet: {
        map: Lf.Map | null,
        markersCluster: Lf.MarkerClusterGroup | null,
        foundMarkersCluster: Lf.MarkerClusterGroup | null,
        /**
         * key is ID
        */
        markersCache: Record<string, {
            found: boolean,
            /**
             * if fusioned marker, multiple chache entries can map to the same marker.
            */
            marker: Lf.Marker | null
        }>,
        /**
         * key is ID in NON fusioned mode, coordinates otherwise
        */
        foundMarkersCache: Record<string, Lf.Marker>,
        /**
         * coordinate to array of IDs
         */
        overlappingIDs: Record<string, Array<string>>
    } = {
        map: null,
        markersCluster: null,
        foundMarkersCluster: null,
        markersCache: {},
        foundMarkersCache: {},
        overlappingIDs: {},
    }

    

    interface Save {
        version: 1,
        foundKeywords: string[],
        foundStations: string[], // ID list
    }
    let save: Save = emptySave();
    function emptySave(): Save {
        return {
            version: 1,
            foundKeywords: [],
            foundStations: [],
        };
    }

    let nbFoundMarkers = 0;
    let nbMarkers = 0;
    let submitMsg = "Enter a value above";
    enum SubmitStatus {
        OK = "var(--c-green)",
        ERROR = "var(--c-red)",
        NEUTRAL = "var(--c-gray-4)"
    }
    let submitStatus: SubmitStatus = SubmitStatus.NEUTRAL;

    let initDone = false;
    
    /**
     * Init leaflet map
     * @param node
     */
    async function init(node: HTMLDivElement) {
        node = node; // mute error;

        console.time("init");
        console.log("init");
        loading = true;

        console.log("loading data");
        let jsonFetch = await fetch("/stops_data.json"); // dev url
        if (jsonFetch.status >= 400 && jsonFetch.status < 600) {
            jsonFetch = await fetch("/station-games/stops_data.json"); // prod url
        }
        data = await jsonFetch.json();

        console.log("setup leaflet");
        const parisCenter: L.LatLngTuple = [48.86296891769729, 2.3394514484316247];
        leaflet.map = Lf.map("map").setView(parisCenter, 13);
        Lf.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            minZoom: 9,
            attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        }).addTo(leaflet.map);
        leaflet.map.setMaxBounds([[parisCenter[0]-1.5, parisCenter[1]-1.5],[parisCenter[0]+1.5, parisCenter[1]+1.5]]);

        const zoomFn = (zoom: number) => (zoom >= 16)? 3 : (zoom >= 15)? 10 : (zoom >= 13)? 60 : (zoom >= 15)? 70 : 80;
        leaflet.markersCluster = Lf.markerClusterGroup({
            chunkedLoading: true,
            chunkProgress: updateProgress,
            maxClusterRadius: zoomFn,
            iconCreateFunction: (cluster) => getClusterDivIcon(cluster, false),
        });
        leaflet.foundMarkersCluster = Lf.markerClusterGroup({
            chunkedLoading: true,
            chunkProgress: updateProgress,
            maxClusterRadius: zoomFn,
            iconCreateFunction: (cluster) => getClusterDivIcon(cluster, true),
        });
        leaflet.map.on("zoom", () => {console.log("zoom: " + leaflet.map?.getZoom())})

        console.log("build cache");
        let i = 0;
        for (const id in data) {
            leaflet.markersCache[id] = {marker: null, found: false};

            // cache overlapping stops
            const posID = getCoordID(id);
            if (leaflet.overlappingIDs[posID]) {
                leaflet.overlappingIDs[posID].push(id);
            } else {
                leaflet.overlappingIDs[posID] = [id];
            }
            i++;
        }
        regenMarkerCache();

        nbMarkers = i; // update UI once

        leaflet.map.addLayer(leaflet.markersCluster);
        leaflet.map.addLayer(leaflet.foundMarkersCluster);
        updateNotFoundMarkers(hideNotFoundMarkers);
        console.timeEnd("init");
        initDone = true;
        loading = false;

        // restore save if exists
        const successLocal = restoreSaveFromLocalStorage();
        if (successLocal) console.log("restored local storage save");
        else console.log("No valid local storage");
    }

    function regenMarkerCache() {
        console.time("regen marker cache");
        console.log("regen marker cache");
        for (const id in data) {
            leaflet.markersCache[id].marker = null; // necessary to force getFusionedMarker to recreate a marker
            leaflet.markersCache[id].marker = stopsFusioned? getFusionedMarker(id) : getMarker(id);
        }
        console.timeEnd("regen marker cache");
    }

    /**
     * load new game
     */
    export function reset() {
        if (confirm("Are you sure?")) {
            save = emptySave();
            nbFoundMarkers = 0;
            reloadMarkersFromSave();
            submitMsg = "Enter a value above";
            submitStatus = SubmitStatus.NEUTRAL;
        }
    }


    /**
     * Update loading progress of markers
     * @param processed
     * @param total
     */
    function updateProgress(processed: number, total: number/*, elapsed, layersArray*/): void {
        if (progressBar) {
            progressBar.value = (total === 0)? 100 : (processed/total) * 100;
            if (processed === total) {
                progressBar.style.display = "none";
                loading = false;
            } else {
                loading = true;
                progressBar.style.display = "initial";
            }
        }
    }

    function getClusterDivIcon(cluster: Lf.MarkerCluster, found: boolean): L.DivIcon {
        const count = cluster.getChildCount();
        const size = (count > 500)? "large" : (count > 120)? "medium" : "small"

        return new Lf.DivIcon({
            html: `<span>${count}</span>`,
            className: `m-cluster m-${size} m-${found? "found" : "not-found"}`,
            iconSize: new Lf.Point(40, 20),
        });
    }

    /**
     * Create leaflet marker
     * @param id
     */
    function getMarker(id: string, blank: boolean = true): L.Marker {
        const posID = getCoordID(id);
        const matchingIDs = leaflet.overlappingIDs[posID];
        
        let name: string;
        if (stopsFusioned) {
            if (blank) {
                name = `not found (x${matchingIDs.length})`;
            } else {
                name = data[id].stop_name + " - ligne(s) ";
    
                for (let i = 0; i < matchingIDs.length; i++) {
                    name += data[matchingIDs[i]].route_long_name;
                    if (i < matchingIDs.length - 1) name += ", ";
                }
            }
        } else {
            name = (blank)? "not found" : data[id].stop_name + " - ligne " + data[id].route_long_name;
        }


        const div = document.createElement("div");
        div.className = "station-marker " + ((blank)? "s-blank" : "s-filled");
        if (!blank) {
            if (stopsFusioned) {
                for (const stopID of matchingIDs) {
                    addColor(div, "#" + data[stopID].background_color);
                }
            } else {
                if (!blank) addColor(div, "#" + data[id].background_color);
            }
        }


        const marker = Lf.marker(Lf.latLng(data[id].stop_lat, data[id].stop_lon), {
            icon: Lf.divIcon({
                html: div,
                iconSize: stopsFusioned? Lf.point(12 + (4*(matchingIDs.length-1)), 12) : Lf.point(12,12),
            }),
            title: name,
        });
        if (!blank) marker.bindTooltip(name, {className: "station-tooltip"});

        return marker;
    }

    /**
     * Add a color to the div of a marker
    */
    function addColor(div: HTMLDivElement, color: string) {
        const child = document.createElement("div");
        child.style.backgroundColor = color;
        div.appendChild(child);
    }

    /**
     * Returns the marker corresponding to the coordinates associated
     * with the provided id.
     * @param id
     * @param blank
     */
    function getFusionedMarker(id: string, blank: boolean = true): Lf.Marker {
        const posID = getCoordID(id);
        if (blank) {
            if (leaflet.markersCache[id] && leaflet.markersCache[id].marker !== null) {
                return leaflet.markersCache[id].marker as Lf.Marker<any>;
            } else {
                return getMarker(id, blank);
            }
        } else {
            if (leaflet.foundMarkersCache[posID]) {
                return leaflet.foundMarkersCache[posID];
            } else {
                return getMarker(id, blank);
            }
        }
    }

    let coordCache: Record<string, string> = {};
    /**
     * Convert a classical ID to a lon-lat ID.
    */
    function getCoordID(id: string) {
        const cache = coordCache[id];
        if (cache) {
            return cache;
        }
        else {
            const str = `${data[id].stop_lat}-${data[id].stop_lon}`;
            coordCache[id] = str;
            return str;
        }
    }

    /**
     * Regenerate not found markers on the map
     */
    function updateNotFoundMarkers(hide: boolean) {
        if (hide) {
            loading = true; 
            console.log("hide not found stations");
            leaflet.markersCluster?.clearLayers();
            loading = false;
        } else {
            loading = true; 
            console.log("show not found stations");
            leaflet.markersCluster?.clearLayers();
            leaflet.markersCluster?.addLayers(
                Object.entries(leaflet.markersCache)
                .filter(v => (
                    !v[1].found
                    && v[1].marker !== null
                    // only the first ID of the overlapping list loads the marker
                    && (leaflet.overlappingIDs[getCoordID(v[0])].indexOf(v[0]) === 0 || !stopsFusioned)
                ))
                .map((v) => v[1].marker as Lf.Marker<any>)
            );
        }
    }
    // subscribe to changes
    $: updateNotFoundMarkers(hideNotFoundMarkers);

    /**
     * Search stations matching keywords and add them to the map
     * @param e
     */
    function searchStation(e: SubmitEvent) {
        loading = true;
        console.log("searching " + stationInput.value);
        console.time("searching");
        e.preventDefault();
        
        // not search if already searched
        const keywords = stationInput.value;
        for (const search of save.foundKeywords) {
            if (stationsEqual(search, keywords)) {
                submitMsg = "Already found";
                submitStatus = SubmitStatus.ERROR;
                saveToLocalStorage();
                loading = false;
                console.timeEnd("searching");
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
                    submitStatus = SubmitStatus.OK;
                }
                addStation(id);

                firstFound = false;
            }
        }
        if (!found) {
            submitMsg = "Not found";
            submitStatus = SubmitStatus.ERROR;
        }
        stationInput.value = "";
        saveToLocalStorage();
        loading = false;
        console.timeEnd("searching");
    }

    let transformCache: Record<string, string> = {};
    /**
     * Determine if two keywords are considered equivalent, by cleaning them up beforehand.
     * @param a
     * @param b
     */
    function stationsEqual(a: string, b: string): boolean {
        const transform = (v: string) => {
            const cache = transformCache[v];
            if (cache) {
                return cache;
            } else {
                const transform = v.normalize("NFD").replaceAll(/[\u0300-\u036f]/g, "").toLowerCase().replaceAll("-"," ");
                transformCache[v] = transform;
                return transform;
            }
            
        };
        return transform(a) === transform(b);
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
     * Convert a station to a found station on the map
     */
    function addFoundMarker(id :string) {
        // console.log("showing " + id + " as found");
        const posID = getCoordID(id);
        const marker = stopsFusioned ? getFusionedMarker(id, false) : getMarker(id, false);

        leaflet.foundMarkersCache[stopsFusioned? posID : id] = marker;
        leaflet.foundMarkersCluster?.addLayer(marker);
        
        if (stopsFusioned) {
            for (const stopID of leaflet.overlappingIDs[posID]) {
                leaflet.markersCache[stopID].found = true;
            }
        } else {
            leaflet.markersCache[id].found = true;
        }
        if (leaflet.markersCache[id].marker !== null) {
            leaflet.markersCluster?.removeLayer(leaflet.markersCache[id].marker as Lf.Marker<any>);
        } else {
            throw new Error("addFoundMarker: didn't expect a null marker when reading marker cache.");
        }
    }

    /**
     * Export save to JSON
     */
    export function downloadSave() {
        loading = true;
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
        loading = false;
    }

    function saveToLocalStorage() {
        console.log("saving to local storage");
        localStorage.setItem("station-games-save", JSON.stringify(save));
    }

    /**
     * Also returns if succeeded
     */
    function restoreSaveFromLocalStorage(): boolean {
        console.log("loading from local storage");
        const data = localStorage.getItem("station-games-save");
        if (data === null) return false;
        loadSave(JSON.parse(data));
        console.log("loading from local storage - OK");
        return true;
    }

    /**
     * load external json file
     */
    export function uploadSave(file: Blob) {
        console.log("uploading save");
        file.text().then((text) => {
            const tmpSave: Save = JSON.parse(text);
            loadSave(tmpSave, false);
        });
    }

    function loadSave(json: Save, silent = true) {
        console.log("loading save");
        if (!json.foundKeywords) {
            showSaveLoadError("missing keywords", silent);
        }
        if (!json.foundStations) {
            showSaveLoadError("missing station IDs", silent);
        }
        if (save.version !== 1) {
            showSaveLoadError("unsupported version", silent);
        }
        if (Object.keys(json).length > 3) {
            showSaveLoadError("unknown fields", silent);
        }
        save = json;
        reloadMarkersFromSave();
    }

    function showSaveLoadError(context: string = "unknown", silent = false) {
        if (!silent) alert("Invalid save file! Make sure you uploaded the right file.\n\nContext: " + context + ".");
        else console.log("load save error: " + context);
    }

    /**
     * Mark all markers as not found, then mark every
     * marker in the save as found and add them.
     */
    function reloadMarkersFromSave() {
        loading = true;
        if (!initDone) {
            console.warn("attempted updating the map");
            return;
        };
        console.log("updating the map");
        nbFoundMarkers = save.foundStations.length;

        // mark all as not found
        leaflet.foundMarkersCluster?.clearLayers();
        leaflet.foundMarkersCache = {};
        for (let m in leaflet.markersCache) {
            leaflet.markersCache[m].found = false;
        }
        regenMarkerCache();

        // show found stations
        console.log("reload found stations")
        let i = 0;
        for (const id of save.foundStations) {
            addFoundMarker(id);
            i++;
        }
        console.log("loaded " + i + " stations")
        updateNotFoundMarkers(hideNotFoundMarkers);
        console.log("map update done.");
        loading = false;
    }
    $: {
        const triggerList = [stopsFusioned];
        reloadMarkersFromSave();
    }
</script>




<div class="container">
    <div>
        <form action="" on:submit={searchStation}>
            <input type="text" name="station" id="station-input" placeholder="Station name" bind:this={stationInput} disabled={loading}>
            <button type="submit">Submit</button>
            <details>
                <summary>Settings</summary>
                <div class="checkbox">
                    <input id="show-not-found" type="checkbox" bind:checked={hideNotFoundMarkers} disabled={loading}/>
                    <label for="show-not-found">Hide not found stations</label>
                </div>
                <div class="checkbox">
                    <input id="fusion-stops" type="checkbox" bind:checked={stopsFusioned} disabled={loading}/>
                    <label for="fusion-stops">Show one point for overlapping stops</label>
                </div>
            </details>
        </form>
        
        <output class="info">
            <span class="status" style="background-color: {submitStatus}">{submitMsg}</span>
            {#if initDone}
                <span>{nbFoundMarkers} / {nbMarkers} ({Math.round(nbFoundMarkers/nbMarkers * 1000)/1000}%)</span>
            {/if}
        </output>
    </div>
    
    <div id="map" use:init></div>
</div>

<progress bind:this={progressBar} max="100" value="0"></progress>
<progress class:visible={loading}></progress>



<style>
    div.container {
        display: flex;
        flex-direction: column;
        width: 100%;
        height: 100%;
    }
    #map {
        height: 100%;
    }

    :global(.leaflet-div-icon) {
        border: none;
    }
    :global(.station-marker) {
        border: 2px solid black;
        display: flex;
        width: 100%;
        height: 100%;
        background-color: white;
    }
    :global(.station-marker > div) {
        flex: 1 1 auto;
    }
    :global(.station-marker.s-blank) {
        content: "?";
    }
    :global(.station-tooltip) {
        font-size: 1rem;
    }
    :global(.m-cluster) {
        display: inline-block;
        width: auto !important;
        height: auto !important;
        padding: 0.4em;
        font-size: 0.9rem;
        line-height: 1em;
        font-weight: bold;
        text-align: center;
        border-radius: 50px;
    }
    :global(.m-cluster.m-not-found.m-small, .m-cluster.m-not-found.m-medium) {
        background-color: var(--c-white);
        border: 1px solid var(--c-gray-3);
    }
    :global(.m-cluster.m-not-found.m-large) {
        background-color: var(--c-orange);
    }
    :global(.m-cluster.m-found) {
        background-color: var(--c-green);
        border: 1px solid var(--c-green-dark);
    }

    form {
        display: flex;
        flex-wrap: wrap;
        justify-content: center;
        align-items: center;
    }

    details {
        margin: 0.5rem 1rem;
        border-radius: var(--p-small-radius);
        border: 2px solid var(--c-blue-3);
        overflow: hidden;
        min-width: 50%;
    }

    @media screen and (max-width: 640px) {
        details {width: 100%;}
    }
    
    details summary {
        font-weight: bold;
        font-size: 1.2rem;
        padding: 0.2rem 0.5rem;
        background-color: var(--c-blue-3);
        cursor: pointer;
    }

    div.checkbox {
        margin: 0.2rem;
    }

    div.checkbox > * {
        cursor: pointer;
    }
    div.checkbox > label:hover {
        font-weight: bold;
    }

    output.info {
        display: flex;
        flex-wrap: wrap;
        align-items: center;
        gap: 0.5rem;
        padding: 0.5rem 1rem;
    }

    div.info > * {
        flex: 0 0 auto;
    }

    @media screen and (max-width: 420px) {
        div.info {
            justify-content: center;
        }
    }

    span.status {
        font-weight: bold;
        padding: 0.2em 0.5em;
        border-radius: var(--p-small-radius);
    }

    progress {
        position: fixed;
        display: none;
        z-index: 5000;
        bottom: 10px;
        left: 5px;
        width: 50vw;
    }
    progress.visible {
        display: initial;
    }
    progress:indeterminate {
        bottom: 0;
    }
</style>