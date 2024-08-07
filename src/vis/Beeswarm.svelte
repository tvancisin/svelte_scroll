<script>
    import * as d3 from "d3";

    export let beeswarm_data;

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

    $: xScale = d3
        .scaleTime()
        .range([0, innerWidth])
        .domain(d3.extent(beeswarm_data, (d) => d[1][0][0]));

    let yScale = d3
        .scaleTime()
        .range([10, innerHeight])
        .domain(d3.extent(beeswarm_data, (d) => d[1][0][0]));

    const simulation = d3.forceSimulation(beeswarm_data);

    $: {
        simulation
            .force("x", d3.forceX((d) => xScale(d[1][0][0])).strength(3))
            .force("y", d3.forceY(height / 2))
            .force("collide", d3.forceCollide(radius))
            .alpha(0.2)
            .alphaDecay(0.0005)
            .restart();
    }

    let nodes = [];
    simulation.on("tick", () => {
        nodes = simulation.nodes();
    });
</script>

<div class="wrapper" bind:clientWidth={width}>
    <svg {width} {height}>
        <g
            class="inner_chart"
            transform="translate({margin.left}, {margin.top})"
        >
            {#each nodes as node}
                <circle cx={node.x} cy={node.y} r={radius} fill="black" />
            {/each}
        </g>
    </svg>
</div>

<style>
  .wrapper {
    width: 100%;
  }
</style>
