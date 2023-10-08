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

    let save: {
        foundKeywords: string[],
        foundStations: string[], // ID list
    } = {
        foundKeywords: [], // TODO implement
        foundStations: [],
    }
    
    /**
     * Init leaflet map
     * @param node
     */
    function init(node) {
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

        leaflet.map.addLayer(leaflet.markersCluster);
        updateNotFoundMarkers();
    }

    /**
     * Update loading progress of markers
     * @param processed
     * @param total
     */
    function updateProgress(processed: number, total: number/*, elapsed, layersArray*/) {
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

    function updateNotFoundMarkers() {
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
</script>

<progress bind:this={progressBar} max="100" value="0"></progress>
<label for="show-not-found">Show not found stations</label>
<input id="show-not-found" type="checkbox" on:change={updateNotFoundMarkers} bind:this={notFoundMarkersCheckbox}/>
<div id="map" use:init></div>


<style>
    #map {
        height: 80vh;
    }
    :global(div.leaflet-div-icon) {
        border: 2px solid black;
    }
    :global(.station-tooltip) {
        font-size: 1rem;
    }
</style>