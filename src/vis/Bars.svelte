<script>
    import * as d3 from "d3";

    export let barchart_data;
    export let selectedCountry;

    let width = 400,
        height = 800,
        radius = 8;

    const margin = {
        top: 10,
        right: 10,
        bottom: 10,
        left: 10,
    };

    const yTicks = [10, 20, 30, 40];

    $: innerWidth = width - margin.left - margin.right;
    $: innerHeight = height - margin.top - margin.bottom;

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
    $: if (barchart_data) {
        subgroups = ["All"]; // Reset to ["All"]
        let keys = Object.keys(barchart_data[0]); // Get keys from the first object in barchart_data
        subgroups.push(keys[2]); // Add the specific key you need
    }

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

    $: yScale = d3.scaleLinear().domain([0, 45]).range([innerHeight, 0]);

    $: formatted_barchart_data = barchart_data
        .map((d) => {
            return ["All", selectedCountry].map((key) => ({
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

    function formatMobile(tick) {
        return "'" + tick.toString().slice(-2);
    }
</script>

<div class="wrapper" bind:clientWidth={width} bind:clientHeight={height}>
    <svg {width} {height}>
        <g>
            <g class="axis x-axis">
                {#each groups as tick}
                    <g
                        class="tick tick-{tick}"
                        transform="translate({xScale(tick)},{height})"
                    >
                        <text y="-2"
                            >{width > 380 ? tick : formatMobile(tick)}</text
                        >
                    </g>
                {/each}
            </g>
            <g class="axis y-axis" transform="translate(0, {margin.top})">
                {#each yTicks as tick}
                    <g
                        class="tick tick-{tick}"
                        transform="translate(0, {yScale(tick) - margin.bottom})"
                    >
                        <line x2="100%" />
                        <text y="-4"
                            >{tick} {tick === 8 ? " million sq km" : ""}</text
                        >
                    </g>
                {/each}
            </g>

            {#each formatted_barchart_data as bar}
                <rect
                    x={bar.x}
                    y={bar.y}
                    width={bar.width}
                    height={bar.height}
                    fill={bar.key === "All" ? "black" : "red"}
                />
                <text
                    x={bar.x + bar.width / 2}
                    y={bar.y - 3}
                    text-anchor="middle"
                    font-size="14px">{Math.round(bar.value)}</text
                >
            {/each}
        </g>
    </svg>
</div>

<style>
    .wrapper {
        height: 90vh;
    }
    .tick {
        font-size: 0.725em;
        font-weight: 200;
    }

    .tick line {
        stroke: #888;
        stroke-dasharray: 2;
    }

    .tick text {
        fill: #888;
        text-anchor: start;
    }

    .tick.tick-0 line {
        stroke-dasharray: 0;
    }

    .x-axis .tick text {
        text-anchor: start;
    }
</style>
