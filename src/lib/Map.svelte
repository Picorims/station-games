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
    import data from "$lib/stops_data.json";

    let progressBar;
    
    function init(node) {
        const parisCenter: L.LatLngTuple = [48.86296891769729, 2.3394514484316247];
        const map = L.map("map").setView(parisCenter, 13);
        L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            minZoom: 9,
            attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        }).addTo(map);
        map.setMaxBounds([[parisCenter[0]-0.8, parisCenter[1]-0.9],[parisCenter[0]+1, parisCenter[1]+0.8]]);

        let markers = L.markerClusterGroup({chunkedLoading: true, chunkProgress: updateProgress});
        let markersCache: La.Layer[] = [];

        let i = 0;
        for (const id in data) {
            const div = document.createElement("div");
            div.style.width = "100%";
            div.style.height = "100%";  
            div.style.backgroundColor = "#" + data[id].ColourWeb_hexa;

            const name = data[id].stop_name + " - ligne " + data[id].route_long_name;

            markersCache.push(L.marker(L.latLng(data[id].stop_lat, data[id].stop_lon), {
                icon: L.divIcon({html: div}),
                title: name,
            }).bindTooltip(name, {className: "station-tooltip"}));
            i++;
        }

        markers.addLayers(markersCache, );
        map.addLayer(markers);
    }

    function updateProgress(processed, total/*, elapsed, layersArray*/) {
        if (progressBar) progressBar.value = `${(processed/total) * 100}`;
    }
</script>

<progress bind:this={progressBar} max="100" value="0"></progress>
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