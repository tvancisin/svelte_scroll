<script>
    import { onMount, onDestroy, createEventDispatcher } from "svelte";
    import mapboxgl from "mapbox-gl";

    // import * as turf from "turf";

    // const dispatch = createEventDispatcher();
    export let mygeojson;

    export let map;

    onMount(() => {
        mapboxgl.accessToken =
            "pk.eyJ1Ijoic2FzaGFnYXJpYmFsZHkiLCJhIjoiY2xyajRlczBlMDhqMTJpcXF3dHJhdTVsNyJ9.P_6mX_qbcbxLDS1o_SxpFg";

        map = new mapboxgl.Map({
            container: map,
            style: "mapbox://styles/sashagaribaldy/cm0az6qe200pf01phd16v6qm0",
            center: [50.224518, 22.213995],
            zoom: 2.5,
            maxZoom: 5,
        });
        // console.log(map);

        map.on("load", () => {

            map.addSource("countries", {
                type: "geojson",
                data: mygeojson,
                generateId: true, //This ensures that all features have unique IDs
            });

            map.addLayer({
                id: "countries-layer",
                type: "fill",
                source: "countries",
                paint: {
                    "fill-color": {
                        property: "number",
                        // stops: [
                        //   [1, "#feda00"],
                        //   [100, "red"],
                        //   [1000, "#400000"],
                        // ],
                        stops: [
                            [1, "#BEBEBE"],
                            [100, "#989898"],
                            [1000, "#484840"],
                        ],
                    },
                    "fill-opacity": [
                        "case",
                        ["<", ["get", "number"], 5],
                        0,
                        [">=", ["get", "number"], 5],
                        1,
                        1,
                    ],
                },
            });

            // Add outline layer
            map.addLayer({
                id: `borders`,
                type: "line",
                source: "countries",
                paint: {
                    "line-color": "#000",
                    "line-width": [
                        "case",
                        ["boolean", ["feature-state", "hover"], false],
                        1.5,
                        0,
                    ],
                },
            });
        });
    });
</script>

<div id="map" bind:this={map} style="width: 100%; height: 100vh;"></div>

<style>
    div {
        top: 0;
        bottom: 0;
    }
    #map {
        pointer-events: none;
    }
</style>
