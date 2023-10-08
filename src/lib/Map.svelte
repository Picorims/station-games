<script lang="ts">
    import * as La from "leaflet";
    import "leaflet/dist/leaflet.css";
    import * as Lb from "leaflet.markercluster";
    const L = {...La, ...Lb};
    import "leaflet.markercluster/dist/MarkerCluster.css";
    import "leaflet.markercluster/dist/MarkerCluster.Default.css";
    import data from "$lib/stops_data.json";
    
    function init(node) {
        const parisCenter: L.LatLngTuple = [48.86296891769729, 2.3394514484316247];
        const map = L.map("map").setView(parisCenter, 13);
        L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            minZoom: 9,
            attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        }).addTo(map);
        map.setMaxBounds([[parisCenter[0]-0.8, parisCenter[1]-0.9],[parisCenter[0]+1, parisCenter[1]+0.8]]);

        let markers = L.markerClusterGroup();

        let i = 0;
        for (const id in data) {
            const div = document.createElement("div");
            div.style.width = "100%";
            div.style.height = "100%";  
            div.style.backgroundColor = "#" + data[id].ColourWeb_hexa;

            const name = data[id].stop_name + " - ligne " + data[id].route_long_name;

            markers.addLayer(L.marker(L.latLng(data[id].stop_lat, data[id].stop_lon), {
                icon: L.divIcon({html: div}),
                title: name,
            }).bindTooltip(name, {className: "station-tooltip"}));
            i++;
        }

        map.addLayer(markers);
    }
</script>

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