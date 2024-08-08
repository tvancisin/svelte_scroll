<script>
    import * as d3 from "d3";

    export let beeswarm_data;
    export let just_year_parser;

    const xTicks = [1995, 2000, 2005, 2010, 2015, 2020];

    let width = 400,
        height = 400,
        radius = 7;

    const margin = {
        top: 10,
        right: 10,
        bottom: 10,
        left: 10,
    };

    $: innerWidth = width - margin.left - margin.right;
    $: innerHeight = height - margin.top - margin.bottom;

    $: xScale = d3
        .scaleTime()
        .range([0, innerWidth])
        .domain(d3.extent(beeswarm_data, (d) => d[1][0][0]));

    let yScale = d3
        .scaleTime()
        .range([10, innerHeight])
        .domain(d3.extent(beeswarm_data, (d) => d[1][0][0]));

    // const simulation = d3.forceSimulation(beeswarm_data);

    // $: {
    //     simulation
    //         .force("x", d3.forceX((d) => xScale(d[1][0][0])).strength(3))
    //         .force("y", d3.forceY(height / 2))
    //         .force("collide", d3.forceCollide(8))
    //         .alpha(0.2)
    //         .alphaDecay(0.0005)
    //         // .restart();
    // }

    // let nodes = [];
    // simulation.on("tick", () => {
    //     nodes = simulation.nodes();
    // });

    $: simulation = d3
        .forceSimulation(beeswarm_data)
        .force("x", d3.forceX((d) => xScale(d[1][0][0])).strength(0.95))
        .force("y", d3.forceY(innerHeight / 2).strength(0.075))
        .force("collide", d3.forceCollide(8))
        .alpha(0.2)
        .alphaDecay(0.0005)
        .stop();

    $: {
        for (
            let i = 0, n = 120;
            // The REPL thinks there is an infinite loop with this next line but it's generally a better way to go
            //n = Math.ceil(Math.log(simulation.alphaMin()) / Math.log(1 - simulation.alphaDecay()));
            i < n;
            ++i
        ) {
            simulation.tick();
        }
    }

    function formatMobile(tick) {
        return "'" + tick.toString().slice(-2);
    }
</script>

<div class="wrapper" bind:clientWidth={width} bind:clientHeight={height}>
    <svg {width} {height}>
        <g
            class="inner_chart"
            transform="translate({margin.left}, {0})"
        >
            <g class="axis x-axis">
                {#each xTicks as tick}
                    <g
                        class="tick tick-{tick}"
                        transform="translate({xScale(
                            just_year_parser(tick),
                        )},{height})"
                    >
                        <line
                            y1="-{height}"
                            y2="-{margin.bottom}"
                            x1="0"
                            x2="0"
                        />
                        <text y="-2"
                            >{width > 380 ? tick : formatMobile(tick)}</text
                        >
                    </g>
                {/each}
                {#each simulation.nodes() as node}
                    <circle cx={node.x} cy={node.y} r={radius} fill="black" />
                {/each}
            </g>
        </g></svg
    >
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
        text-anchor: middle;
    }
</style>
