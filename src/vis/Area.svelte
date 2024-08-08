<script>
	import * as d3 from "d3";

	export let areachart_data;
	export let just_year_parser;
	console.log(areachart_data);
	

	let width = 400,
		height = 800;

	const yTicks = [30, 60, 90];
	const xTicks = [1995, 2000, 2005, 2010, 2015, 2020];

	const margin = {
		top: 10,
		right: 10,
		bottom: 10,
		left: 10,
	};

	$: innerWidth = width - margin.left - margin.right;
	$: innerHeight = height - margin.top - margin.bottom;

	$: xScale = d3
		.scaleUtc()
		.domain(d3.extent(areachart_data, (d) => d.date))
		.range([0, innerWidth]);

	$: yScale = d3.scaleLinear().domain([0, 100]).range([innerHeight, 0]);

	$: minX = areachart_data[0].date;
	$: maxX = areachart_data[areachart_data.length - 1].date;
	$: path = `M${areachart_data.map((p) => `${xScale(p.date)},${yScale(p.value)}`).join("L")}`;
	$: area = `${path}L${xScale(maxX)},${yScale(0)}L${xScale(minX)},${yScale(0)}Z`;

	function formatMobile(tick) {
		return "'" + tick.toString().slice(-2);
	}
</script>

<div class="wrapper" bind:clientWidth={width} bind:clientHeight={height}>
	<svg {width} {height}>
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

		<g class="axis x-axis">
			{#each xTicks as tick}
				<g
					class="tick tick-{tick}"
					transform="translate({xScale(just_year_parser(tick))},{height})"
				>
					<line y1="-{height}" y2="-{margin.bottom}" x1="0" x2="0" />
					<text y="-2">{width > 380 ? tick : formatMobile(tick)}</text
					>
				</g>
			{/each}
		</g>

		<!-- data -->
		<path class="path-area" d={area} />
		<path class="path-line" d={path} />
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
		text-anchor: middle;
	}

	.path-line {
		fill: none;
		stroke: rgb(0, 100, 100);
		stroke-linejoin: round;
		stroke-linecap: round;
		stroke-width: 2;
	}

	.path-area {
		fill: rgba(0, 100, 100, 0.2);
	}
</style>
