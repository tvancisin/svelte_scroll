<script>
    import * as d3 from "d3";

    export let donut_data;

    let width = 400,
        height = 800;

    const margin = {
        top: 0,
        right: 0,
        bottom: 20,
        left: 0,
    };

    $: innerWidth = width - margin.left - margin.right;
    let innerHeight = height - margin.top - margin.bottom;

    $: radius = Math.min(innerWidth, innerHeight) / 2;

    // Compute the position of each group on the pie:
    const pie = d3
        .pie()
        .sort(null) // Do not sort group by size
        .value((d) => d[1][1].length);

    const data_ready = pie(Object.entries(donut_data));
    
    // The arc generator
    $: arc = d3
        .arc()
        .innerRadius(radius * 0.5) // This is the size of the donut hole
        .outerRadius(radius * 0.8);

    // Another arc that won't be drawn. Just for labels positioning
    $: outerArc = d3
        .arc()
        .innerRadius(radius * 0.9)
        .outerRadius(radius * 0.9);
</script>

<div class="wrapper" bind:clientWidth={width}>
    <svg {width} {height}>
        <g transform="translate({width/2} {height/2})">
            {#each data_ready as slice}
                <path d={arc(slice)} stroke="white" />
            {/each}
        </g>
    </svg>
</div>

<style>
    .wrapper {
        width: 100%;
    }
</style>
