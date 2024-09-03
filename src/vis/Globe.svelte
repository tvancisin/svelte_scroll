<script>
    import { onMount, onDestroy, createEventDispatcher } from "svelte";
    import mapboxgl from "mapbox-gl";
    import { filter } from "d3";

    const dispatch = createEventDispatcher();
    export let mygeojson;
    export let map_array;
    export let map;
    let current_zoom

    $: console.log(map_array);
    $: console.log(mygeojson);

    function adjustMapForWindowSize() {
        let centerCoordinates = map.getCenter();
        if (window.innerWidth <= 768) {
            current_zoom = 1.4
            map.flyTo({
                center: [centerCoordinates.lng, centerCoordinates.lat],
                zoom: 1.4,
            });
        } else if (window.innerWidth <= 1000) {
            current_zoom = 2.2 
            map.flyTo({
                center: [centerCoordinates.lng, centerCoordinates.lat],
                zoom: 2.2,
            });
        } else {
            current_zoom = 2.5
            map.flyTo({
                center: [centerCoordinates.lng, centerCoordinates.lat],
                zoom: 2.5,
            });
        }
    }

    $: if (mygeojson && map_array && map) {
        // Ensure this block runs only after the map has fully loaded
        map.on("load", () => {
            // Check if the source already exists
            if (!map.getSource("countries")) {
                // Add the source if it doesn't already exist
                map.addSource("countries", {
                    type: "geojson",
                    data: mygeojson,
                    generateId: true, // Ensures all features have unique IDs
                });

                // Add the layer with the filtered countries
                map.addLayer({
                    id: "countries-layer",
                    type: "fill",
                    source: "countries",
                    paint: {
                        "fill-color": "gray",
                    },
                    filter: ["in", ["get", "ADMIN"], ["literal", map_array]],
                });
            } else {
                // If the source exists, update the data and filter
                map.getSource("countries").setData(mygeojson);
                map.setFilter("countries-layer", [
                    "in",
                    ["get", "ADMIN"],
                    ["literal", map_array],
                ]);
            }
            dispatch("mapLoaded");
        });
    }

    onMount(() => {
        window.scrollTo(0, 0);

        mapboxgl.accessToken =
            "pk.eyJ1Ijoic2FzaGFnYXJpYmFsZHkiLCJhIjoiY2xyajRlczBlMDhqMTJpcXF3dHJhdTVsNyJ9.P_6mX_qbcbxLDS1o_SxpFg";

        // Initialize the Mapbox map
        map = new mapboxgl.Map({
            container: "map", // Replace with your map container's ID or DOM element
            style: "mapbox://styles/sashagaribaldy/cm0az6qe200pf01phd16v6qm0",
            center: [50, 22],
            // zoom: 2.5,
            maxZoom: 5,
        });

        adjustMapForWindowSize()
        // window.addEventListener("load", adjustMapForWindowSize);
        window.addEventListener("resize", adjustMapForWindowSize);
    });

    $: if (map && map_array && map.getSource("countries")) {
        // Ensure the map and source are loaded before updating the filter
        map.setFilter("countries-layer", [
            "in",
            ["get", "ADMIN"],
            ["literal", map_array],
        ]);
    }

    // const countriesToRemove = [
    //     "France",
    //     "United Kingdom",
    //     "United States of America",
    // ];
    // $: filteredCountries = map_array.filter(
    //     (country) => !countriesToRemove.includes(country),
    // );

    // onMount(() => {
    //     window.scrollTo(0, 0);

    //     mapboxgl.accessToken =
    //         "pk.eyJ1Ijoic2FzaGFnYXJpYmFsZHkiLCJhIjoiY2xyajRlczBlMDhqMTJpcXF3dHJhdTVsNyJ9.P_6mX_qbcbxLDS1o_SxpFg";

    //     map = new mapboxgl.Map({
    //         container: map,
    //         style: "mapbox://styles/sashagaribaldy/cm0az6qe200pf01phd16v6qm0",
    //         center: [50, 22],
    //         zoom: 2.5,
    //         maxZoom: 5,
    //     });
    //     // console.log(map);

    //     map.on("load", () => {
    //         map.addSource("countries", {
    //             type: "geojson",
    //             data: mygeojson,
    //             generateId: true, //This ensures that all features have unique IDs
    //         });

    //         map.addLayer({
    //             id: "countries-layer",
    //             type: "fill",
    //             source: "countries",
    //             paint: {
    //                 "fill-color": "gray",
    //             },
    //         });

    //         // Add outline layer
    //         // map.addLayer({
    //         //     id: `borders`,
    //         //     type: "line",
    //         //     source: "countries",
    //         //     paint: {
    //         //         "line-color": "#000",
    //         //         "line-width": [
    //         //             "case",
    //         //             ["boolean", ["feature-state", "hover"], false],
    //         //             1.5,
    //         //             0,
    //         //         ],
    //         //     },
    //         // });
    //     });

    //     dispatch("mapLoaded");
    // });
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
