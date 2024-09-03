<script>
    import * as d3 from "d3";

    export let beeswarm_data;
    export let just_year_parser;
    export let step;
    export let selectedCountry;

    function fillBlack() {
        document.querySelectorAll("circle").forEach((circle) => {
            circle.setAttribute("fill", "black");
        });
    }

    $: {
        if (step === "two") {
            fillBlack();
            if (selectedCountry === "Russia") {
                document.querySelectorAll(".soviet").forEach((circle) => {
                    circle.setAttribute("fill", "#7b8ad6");
                });
            } else if (selectedCountry === "China") {
                document.querySelectorAll(".china_high").forEach((circle) => {
                    circle.setAttribute("fill", "#7b8ad6");
                });
            }
        } else if (step === "three") {
            fillBlack();
            if (selectedCountry === "Russia") {
                document.querySelectorAll(".syria").forEach((circle) => {
                    circle.setAttribute("fill", "#7b8ad6");
                });
            } else if (selectedCountry === "China") {
                document.querySelectorAll(".un_p5").forEach((circle) => {
                    circle.setAttribute("fill", "#7b8ad6");
                });
            }
        } else {
            fillBlack();
        }
    }

    const soviet = [
        "Armenia",
        "Azerbaijan",
        "Belarus",
        "Estonia",
        "Georgia",
        "Kazakhstan",
        "Kyrgyzstan",
        "Latvia",
        "Lithuania",
        "Moldova",
        "Russia",
        "Tajikistan",
        "Turkmenistan",
        "Ukraine",
        "Uzbekistan",
    ];

    const syria = ["Syria", "Libya", "Central African Republic"];

    let china_highlight = [
        "Text of Joint Statement (29/09/2005)",
        "Agreement on the Resolution of the Conflict in the Republic of South Sudan (ARCSS) (17/08/2015)",
        "Agreement on the Cessation of Hostilities, Protection of Civilians and Humanitarian Access, Republic of South Sudan (21/12/2017)",
        "Joint Trilateral Statement by the People’s Republic of China, the Kingdom of Saudi Arabia, and the Islamic Republic of Iran (10/03/2023)",
        "The Nationwide Ceasefire Agreement (NCA) between The Government of the Republic of the Union of Myanmar and the Ethnic Armed Organizations (EAO) (15/10/2015)",
    ];

    const xTicks = [1995, 2000, 2005, 2010, 2015, 2020];

    let width = 400,
        height = 400;

    const margin = {
        top: 10,
        right: 10,
        bottom: 10,
        left: 10,
    };

    $: innerWidth = width - margin.left - margin.right;
    $: innerHeight = height - margin.top - margin.bottom;

    let tooltipVisible = false;
    let tooltipContent = "";
    let tooltipX = 0;
    let tooltipY = 0;

    // Scale for radius based on width
    $: radiusScale = d3
        .scaleLinear()
        .domain([300, 800]) // Example domain: min and max width values
        .range([4, 7]); // Example range: min and max radius values

    // Compute the radius based on the current width
    $: radius = radiusScale(innerWidth);

    $: xScale = d3
        .scaleTime()
        .range([0, innerWidth])
        .domain(d3.extent(beeswarm_data, (d) => d[1][0][0]));

    let yScale = d3
        .scaleTime()
        .range([10, innerHeight])
        .domain(d3.extent(beeswarm_data, (d) => d[1][0][0]));

    $: simulation = d3
        .forceSimulation(beeswarm_data)
        .force("x", d3.forceX((d) => xScale(d[1][0][0])).strength(0.95))
        .force("y", d3.forceY(innerHeight / 2).strength(0.03))
        .force("collide", d3.forceCollide(radius + 2)) // Use dynamic radius here
        .alpha(0.3)
        .alphaDecay(0.001)
        .stop();

    $: {
        for (let i = 0, n = 200; i < n; ++i) {
            simulation.tick();
        }
    }

    function formatMobile(tick) {
        return "'" + tick.toString().slice(-2);
    }

    function handleMouseOver(event, d) {
        // Display tooltip with information
        tooltipContent = d[1][0][1][0].agt_dat;
        if (d.x >= width / 2) {
            tooltipX = d.x - 210;
        } else {
            tooltipX = d.x + 10;
        }
        tooltipY = d.y + 10;
        tooltipVisible = true;

        d3.select(event.target).attr("fill", "red"); // Change fill color on hover
    }

    function handleMouseOut(event, d) {
        // Hide the tooltip
        tooltipVisible = false;

        d3.select(event.target).attr("fill", "black"); // Reset fill color when mouse leaves
    }

    function assignClass(d) {
        let individual_info = d[1][0][1][0];
        let soviet_included = soviet.some((country) =>
            individual_info.Con.includes(country),
        );
        let syria_included = syria.some((country) =>
            individual_info.Con.includes(country),
        );

        let all_actors = [];
        d[1][0][1].forEach(function (x) {
            all_actors.push(x.actor_name);
        });

        let specifiedCountries = [
            "Russia",
            "France",
            "United Kingdom",
            "United States",
            "China",
        ];
        let containsUnitedNations = all_actors.includes("United Nations");
        let containsEurope = all_actors.includes(
            "Conference on Security and Cooperation in Europe",
        );
        let containsAllSpecifiedCountries = specifiedCountries.every(
            (country) => all_actors.includes(country),
        );


        let classes = [];
        //Add china classes
        if (!china_highlight.includes(d[1][0][1][0].agt_dat)) {
            if (
                containsUnitedNations ||
                containsAllSpecifiedCountries ||
                containsEurope ||
                d[1][0][1][0].AgtId == 2433
            ) {
                classes.push("china_high un_p5");
            } else {
                classes.push("china_high");
            }
        }

        // Add "soviet" to the array if the condition is met
        if (soviet_included) {
            classes.push("soviet");
        }
        // Add "syria" to the array if the condition is met
        if (syria_included) {
            classes.push("syria");
        }

        // Join the class names into a single string separated by spaces
        return classes.join(" ");
    }
</script>

<div class="wrapper" bind:clientWidth={width} bind:clientHeight={height}>
    <svg {width} {height}>
        <g class="inner_chart" transform="translate({margin.left}, {0})">
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
                    <circle
                        cx={node.x}
                        cy={node.y}
                        r={radius}
                        fill="black"
                        class={assignClass(node)}
                        on:mouseover={(event) => handleMouseOver(event, node)}
                        on:mouseout={(event) => handleMouseOut(event, node)}
                        on:focus={(event) => handleMouseOver(event, node)}
                        on:blur={(event) => handleMouseOut(event, node)}
                    />
                {/each}
            </g>
        </g>
    </svg>

    <!-- Tooltip div -->
    {#if tooltipVisible}
        <div class="tooltip" style="left: {tooltipX}px; top: {tooltipY}px;">
            {tooltipContent}
        </div>
    {/if}
</div>

<style>
    .wrapper {
        height: 90vh;
        position: relative;
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

    .tooltip {
        width: 200px;
        z-index: 5;
        position: absolute;
        background-color: white;
        border: 1px solid #ccc;
        padding: 5px;
        font-size: 0.6em;
        pointer-events: none; /* Prevent tooltip from blocking mouse events */
        box-shadow: 0px 0px 5px rgba(0, 0, 0, 0.2);
    }

    circle:focus {
        outline: none; /* Optional: Remove default focus outline */
    }
</style>
