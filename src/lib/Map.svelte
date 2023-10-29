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
    let hideNotFoundMarkers: boolean = true;
    let stopsFusioned: boolean = false;
    let stationInput: HTMLInputElement;

    let leaflet: {
        map: La.Map | null,
        markersCluster: La.MarkerClusterGroup | null,
        foundMarkersCluster: La.MarkerClusterGroup | null,
        /**
         * key is ID
        */
        markersCache: Record<string, {
            found: boolean,
            /**
             * if fusioned marker, multiple chache entries can map to the same marker.
            */
            marker: La.Marker | null
        }>,
        /**
         * key is ID in NON fusioned mode, coordinates otherwise
        */
        foundMarkersCache: Record<string, La.Marker>,
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
    let save: Save = {
        version: 1,
        foundKeywords: [],
        foundStations: [],
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
        leaflet.map.setMaxBounds([[parisCenter[0]-1.5, parisCenter[1]-1.5],[parisCenter[0]+1.5, parisCenter[1]+1.5]]);

        const zoomFn = (zoom: number) => (zoom >= 16)? 3 : (zoom >= 15)? 10 : (zoom >= 11)? 50 : 60;
        leaflet.markersCluster = L.markerClusterGroup({
            chunkedLoading: true,
            chunkProgress: updateProgress,
            maxClusterRadius: zoomFn,
            iconCreateFunction: (cluster) => getClusterDivIcon(cluster, false),
        });
        leaflet.foundMarkersCluster = L.markerClusterGroup({
            chunkedLoading: true,
            chunkProgress: updateProgress,
            maxClusterRadius: zoomFn,
            iconCreateFunction: (cluster) => getClusterDivIcon(cluster, true),
        });
        leaflet.map.on("zoom", () => {console.log("zoom: " + leaflet.map?.getZoom())})

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
    }

    function regenMarkerCache() {
        for (const id in data) {
            leaflet.markersCache[id].marker = null; // necessary to force getFusionedMarker to recreate a marker
            leaflet.markersCache[id].marker = stopsFusioned? getFusionedMarker(id) : getMarker(id);
        }
    }

    /**
     * Update loading progress of markers
     * @param processed
     * @param total
     */
    function updateProgress(processed: number, total: number/*, elapsed, layersArray*/): void {
        if (progressBar) progressBar.value = (processed/total) * 100;
    }

    function getClusterDivIcon(cluster: La.MarkerCluster, found: boolean): L.DivIcon {
        const count = cluster.getChildCount();
        const size = (count > 500)? "large" : (count > 120)? "medium" : "small"

        return new L.DivIcon({
            html: `<span>${count}</span>`,
            className: `m-cluster m-${size} m-${found? "found" : "not-found"}`,
            iconSize: new L.Point(40, 20),
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


        //FIXME: optimize div creation
        const div = document.createElement("div"); //4.9%
        div.style.display = "flex"; // 11%
        div.style.width = "100%"; // 8.9%
        div.style.height = "100%"; // 4.9%
        div.style.backgroundColor = "white"; //12%
        if (!blank) {
            if (stopsFusioned) {
                for (const stopID of matchingIDs) {
                    addColor(div, "#" + data[stopID].background_color);
                }
            } else {
                if (!blank) addColor(div, "#" + data[id].background_color);
            }
        }


        const marker = L.marker(L.latLng(data[id].stop_lat, data[id].stop_lon), {
            icon: L.divIcon({
                html: div,
                iconSize: stopsFusioned? L.point(12 + (4*(matchingIDs.length-1)), 12) : L.point(12,12),
            }),
            title: name,
        });
        marker.bindTooltip(name, {className: "station-tooltip"}); //FIXME 23% = bottleneck on not found stations ?

        return marker;
    }

    /**
     * Add a color to the div of a marker
    */
    function addColor(div: HTMLDivElement, color: string) {
        const child = document.createElement("div");
        child.style.backgroundColor = color;
        child.style.flex = "1 1 auto";
        div.appendChild(child);
    }

    /**
     * Returns the marker corresponding to the coordinates associated
     * with the provided id.
     * @param id
     * @param blank
     */
    function getFusionedMarker(id: string, blank: boolean = true): La.Marker {
        const posID = getCoordID(id);
        if (blank) {
            if (leaflet.markersCache[id] && leaflet.markersCache[id].marker !== null) {
                return leaflet.markersCache[id].marker as La.Marker<any>;
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

    /**
     * Convert a classical ID to a lon-lat ID.
    */
    function getCoordID(id: string) {
        return `${data[id].stop_lat}-${data[id].stop_lon}`;
    }

    /**
     * Regenerate not found markers on the map
     */
    function updateNotFoundMarkers(hide: boolean) {
        if (hide) {
            console.log("hide not found stations");
            leaflet.markersCluster?.clearLayers();
        } else {
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
                .map((v) => v[1].marker as La.Marker<any>)
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
        console.log("searching " + e);
        e.preventDefault();
        
        // not search if already searched
        const keywords = stationInput.value;
        for (const search of save.foundKeywords) {
            if (stationsEqual(search, keywords)) {
                submitMsg = "Already found";
                submitStatus = SubmitStatus.ERROR;
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
     * Convert a station to a found station on the map
     */
    function addFoundMarker(id :string) {
        console.log("showing " + id + " as found");
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
            leaflet.markersCluster?.removeLayer(leaflet.markersCache[id].marker as La.Marker<any>);
        } else {
            throw new Error("addFoundMarker: didn't expect a null marker when reading marker cache.");
        }
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
        if (!initDone) {
            console.log("attempted updating the map");
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
        for (const id of save.foundStations) {
            addFoundMarker(id);
        }
        updateNotFoundMarkers(hideNotFoundMarkers);
        console.log("map update done.");
    }
    $: {
        const triggerList = [stopsFusioned];
        reloadMarkersFromSave();
    }

    // TODO: markercluster grouping too quickly
</script>




<div class="container">
    <div>
        <form action="" on:submit={searchStation}>
            <input type="text" name="station" id="station-input" placeholder="Station name" bind:this={stationInput}>
            <button type="submit">Submit</button>
            <details>
                <summary>Settings</summary>
                <div class="checkbox">
                    <input id="show-not-found" type="checkbox" bind:checked={hideNotFoundMarkers}/>
                    <label for="show-not-found">Hide not found stations</label>
                </div>
                <div class="checkbox">
                    <input id="fusion-stops" type="checkbox" bind:checked={stopsFusioned}/>
                    <label for="fusion-stops">Show one point for overlapping stops</label>
                </div>
            </details>
        </form>
        
        <output class="info">
            <span class="status" style="background-color: {submitStatus}">{submitMsg}</span>
            <span>{nbFoundMarkers} / {nbMarkers} ({Math.round(nbFoundMarkers/nbMarkers * 1000)/1000}%)</span>
        </output>
    </div>
    
    <div id="map" use:init></div>
</div>

<progress bind:this={progressBar} max="100" value="0"></progress>



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
    :global(div.leaflet-div-icon) {
        border: 2px solid black;
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
        z-index: 5000;
        bottom: 0px;
        left: 5px;
        width: 50vw;
    }
</style>