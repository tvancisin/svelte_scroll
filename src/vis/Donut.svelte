<script>
    import * as d3 from "d3";

    export let donut_data;
    export let selectedCountry;

    // Define the desired order
    let desiredOrder = [
        "Pre-negotiation",
        "Ceasefire",
        "Framework-substantive, partial",
        "Framework-substantive, comprehensive",
        "Implementation",
    ];

    $: if (selectedCountry == "China") {
        // Sort the array based on the desired order
        donut_data.sort((a, b) => {
            return desiredOrder.indexOf(a[0]) - desiredOrder.indexOf(b[0]);
        });
    }

    let width = 400,
        height = 800;

    const russia_color = d3
        .scaleOrdinal()
        .domain([
            "Pre-negotiation",
            "Ceasefire",
            "Framework-substantive, partial",
            "Framework-substantive, comprehensive",
            "Renewal",
            "Implementation",
            "Other",
        ])
        .range([
            "#016099",
            "#df1f36",
            "#fd8189",
            "#fdd900",
            "#7b8ad6",
            "#3aae2a",
            "#c0de88",
        ]);

    const china_color = d3
        .scaleOrdinal()
        .domain([
            "Pre-negotiation",
            "Ceasefire",
            "Framework-substantive, partial",
            "Framework-substantive, comprehensive",
            "Implementation",
        ])
        .range(["#016099", "#df1f36", "#fd8189", "#fdd900", "#3aae2a"]);

    const margin = {
        top: 10,
        right: 10,
        bottom: 10,
        left: 10,
    };

    $: innerWidth = width - margin.left - margin.right;
    $: innerHeight = height - margin.top - margin.bottom;

    $: radius = Math.min(innerWidth, innerHeight) / 2;

    // Compute the position of each group on the pie:
    const pie = d3
        .pie()
        .sort(null) // Do not sort group by size
        .value((d) => d[1][1].length);

    $: data_ready = pie(Object.entries(donut_data));

    // The arc generator
    $: arc = d3
        .arc()
        .innerRadius(radius * 0.5) // This is the size of the donut hole
        .outerRadius(radius * 0.8)
        .cornerRadius(4);

    // Another arc that won't be drawn. Just for labels positioning
    $: outerArc = d3
        .arc()
        .innerRadius(radius * 0.9)
        .outerRadius(radius * 0.9);

    $: calcPoints = function (d) {
        const posA = arc.centroid(d); // line insertion in the slice
        const posB = outerArc.centroid(d); // line break: we use the other arc generator that has been built only for that
        const posC = outerArc.centroid(d); // Label position = almost the same as posB
        const midangle = d.startAngle + (d.endAngle - d.startAngle) / 2; // we need the angle to see if the X position will be at the extreme right or extreme left
        posC[0] = radius * 0.88 * (midangle < Math.PI ? 1 : -1); // multiply by 1 or -1 to put it on the right or on the left
        return [posA, posB, posC];
    };

    $: calcText = function (d) {
        const pos = outerArc.centroid(d);
        const midangle = d.startAngle + (d.endAngle - d.startAngle) / 2;
        pos[0] = radius * 0.88 * (midangle < Math.PI ? 1 : -1);
        pos[1] -= 10;
        return `translate(${pos})`;
    };
</script>

<div class="wrapper" bind:clientWidth={width} bind:clientHeight={height}>
    <svg {width} {height}>
        <g transform="translate({width / 2} {height / 2})">
            {#each data_ready as slice}
                <path
                    d={arc(slice)}
                    stroke="white"
                    fill={selectedCountry === "Russia"
                        ? russia_color(slice.data[1])
                        : selectedCountry === "China"
                          ? china_color(slice.data[1])
                          : undefined}
                />
            {/each}
            {#each data_ready as line}
                <polyline
                    fill="none"
                    stroke="black"
                    stroke-width="1"
                    points={calcPoints(line)}
                />
            {/each}
            {#each data_ready as d}
                <text
                    fill="black"
                    font-size="14px"
                    transform={calcText(d)}
                    text-anchor="middle"
                >
                    {d.data[1][0] === "Framework-substantive, partial"
                        ? "Partial"
                        : d.data[1][0] ===
                            "Framework-substantive, comprehensive"
                          ? "Comprehensive"
                          : d.data[1][0]}
                </text>
            {/each}
        </g>
    </svg>
</div>

<style>
    .wrapper {
        height: 90vh;
    }
</style>
