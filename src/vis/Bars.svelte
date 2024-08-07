<script>
    import * as d3 from "d3";

    export let barchart_data;
    console.log(barchart_data);

    let width = 400,
        height = 800,
        radius = 8;

    const margin = {
        top: 0,
        right: 0,
        bottom: 20,
        left: 0,
    };

    $: innerWidth = width - margin.left - margin.right;
    let innerHeight = height - margin.top - margin.bottom;

    // List of groups
    let groups = [
        "Pre-negotiation",
        "Ceasefire",
        "Partial",
        "Comprehensive",
        "Implementation",
        "Renewal",
        "Other",
    ];

    let subgroups = ["All"];
    let keys = Object.keys(barchart_data[0]);
    subgroups.push(keys[2]);

    $: xScale = d3
        .scaleBand()
        .domain(groups)
        .range([0, innerWidth])
        .padding([0.2]);

    $: xScaleSubgroup = d3
        .scaleBand()
        .domain(subgroups)
        .range([0, xScale.bandwidth()])
        .padding([0.05]);

    let yScale = d3.scaleLinear().domain([0, 45]).range([innerHeight, 0]);

    $: formatted_barchart_data = barchart_data
        .map((d) => {
            return ["All", "Russia"].map((key) => ({
                key,
                value: d[key],
                group: d.group,
                x: xScale(d.group) + xScaleSubgroup(key),
                y: yScale(d[key]),
                width: xScaleSubgroup.bandwidth(),
                height: innerHeight - yScale(d[key]),
            }));
        })
        .flat();

</script>

<div class="wrapper" bind:clientWidth={width}>
    <svg {width} {height}>
        <g>
            {#each formatted_barchart_data as bar}
                <rect
                    x={bar.x}
                    y={bar.y}
                    width={bar.width}
                    height={bar.height}
                />
            {/each}
        </g>
    </svg>
</div>

<style>
    .wrapper {
        width: 100%;
    }
</style>
