<script>
	// CORE IMPORTS
	import { setContext, onMount } from "svelte";
	import { getCSV, getMotion } from "./utils.js";
	import { themes } from "./config.js";
	import ONSHeader from "./layout/ONSHeader.svelte";
	import ONSFooter from "./layout/ONSFooter.svelte";
	import Header from "./layout/Header.svelte";
	import Section from "./layout/Section.svelte";
	import Media from "./layout/Media.svelte";
	import Scroller from "./layout/Scroller.svelte";
	import Filler from "./layout/Filler.svelte";
	import Divider from "./layout/Divider.svelte";
	import Toggle from "./ui/Toggle.svelte";
	import Arrow from "./ui/Arrow.svelte";
	import Em from "./ui/Em.svelte";
	import * as d3 from "d3";
	import Beeswarm from "./vis/Beeswarm.svelte";
	import Area from "./vis/Area.svelte";
	import Donut from "./vis/Donut.svelte";
	import Bars from "./vis/Bars.svelte";
	import Globe from "./vis/Globe.svelte"

	// DEMO-SPECIFIC IMPORTS
	import bbox from "@turf/bbox";
	import {
		getData,
		setColors,
		getTopo,
		getGeo,
		getBreaks,
		getColor,
	} from "./utils.js";
	import { colors, units } from "./config.js";
	import {
		ScatterChart,
		LineChart,
		BarChart,
	} from "@onsvisual/svelte-charts";
	import {
		Map,
		MapSource,
		MapLayer,
		MapTooltip,
	} from "@onsvisual/svelte-maps";

	// CORE CONFIG (COLOUR THEMES)
	// Set theme globally (options are 'light', 'dark' or 'lightblue')
	let theme = "light";
	setContext("theme", theme);
	setColors(themes, theme);

	// CONFIG FOR SCROLLER COMPONENTS
	// Config
	const threshold = 0.65;
	// State
	let animation = getMotion(); // Set animation preference depending on browser preference
	let id = {}; // Object to hold visible section IDs of Scroller components
	let idPrev = {}; // Object to keep track of previous IDs, to compare for changes
	onMount(() => {
		idPrev = { ...id };
	});

	// DEMO-SPECIFIC CONFIG
	// Constants
	const datasets = ["region", "district"];
	const topojson = "./data/geo_lad2021.json";

	const mapstyle =
		"https://bothness.github.io/ons-basemaps/data/style-omt.json";
	const mapbounds = {
		uk: [
			[-9, 49],
			[2, 61],
		],
	};

	// Data
	let data = { district: {}, region: {} };
	let metadata = { district: {}, region: {} };
	let geojson;

	// Element bindings
	let map = null; // Bound to mapbox 'map' instance once initialised

	// State
	let hovered; // Hovered district (chart or map)
	let selected; // Selected district (chart or map)
	$: region =
		selected && metadata.district.lookup
			? metadata.district.lookup[selected].parent
			: null; // Gets region code for 'selected'
	$: chartHighlighted =
		metadata.district.array && region
			? metadata.district.array
					.filter((d) => d.parent == region)
					.map((d) => d.code)
			: []; // Array of district codes in 'region'
	let mapHighlighted = []; // Highlighted district (map only)
	let xKey = "area"; // xKey for scatter chart
	let yKey = null; // yKey for scatter chart
	let zKey = null; // zKey (color) for scatter chart
	let rKey = null; // rKey (radius) for scatter chart
	let mapKey = "density"; // Key for data to be displayed on map
	let explore = false; // Allows chart/map interactivity to be toggled on/off

	// FUNCTIONS (INCL. SCROLLER ACTIONS)

	// Functions for chart and map on:select and on:hover events
	function doSelect(e) {
		console.log(e);
		selected = e.detail.id;
		if (e.detail.feature) fitById(selected); // Fit map if select event comes from map
	}
	function doHover(e) {
		hovered = e.detail.id;
	}

	// Functions for map component
	function fitBounds(bounds) {
		if (map) {
			map.fitBounds(bounds, { animate: animation, padding: 30 });
		}
	}
	function fitById(id) {
		if (geojson && id) {
			let feature = geojson.features.find(
				(d) => d.properties.AREACD == id,
			);
			let bounds = bbox(feature.geometry);
			fitBounds(bounds);
		}
	}

	// Actions for Scroller components
	const actions = {
		map: {
			// Actions for <Scroller/> with id="map"
			map01: () => {
				// Action for <section/> with data-id="map01"
				fitBounds(mapbounds.uk);
				mapKey = "density";
				mapHighlighted = [];
				explore = false;
			},
			map02: () => {
				fitBounds(mapbounds.uk);
				mapKey = "age_med";
				mapHighlighted = [];
				explore = false;
			},
			map03: () => {
				let hl = [...data.district.indicators].sort(
					(a, b) => b.age_med - a.age_med,
				)[0];
				fitById(hl.code);
				mapKey = "age_med";
				mapHighlighted = [hl.code];
				explore = false;
			},
			map04: () => {
				fitBounds(mapbounds.uk);
				mapKey = "age_med";
				mapHighlighted = [];
				explore = true;
			},
		},
		chart: {
			chart01: () => {
				xKey = "area";
				yKey = null;
				zKey = null;
				rKey = null;
				explore = false;
			},
			chart02: () => {
				xKey = "area";
				yKey = null;
				zKey = null;
				rKey = "pop";
				explore = false;
			},
			chart03: () => {
				xKey = "area";
				yKey = "density";
				zKey = null;
				rKey = "pop";
				explore = false;
			},
			chart04: () => {
				xKey = "area";
				yKey = "density";
				zKey = "parent_name";
				rKey = "pop";
				explore = false;
			},
			chart05: () => {
				xKey = "area";
				yKey = "density";
				zKey = null;
				rKey = "pop";
				explore = true;
			},
		},
	};

	// Code to run Scroller actions when new caption IDs come into view
	function runActions(codes = []) {
		codes.forEach((code) => {
			if (id[code] != idPrev[code]) {
				if (actions[code][id[code]]) {
					actions[code][id[code]]();
				}
				idPrev[code] = id[code];
			}
		});
	}
	$: id && runActions(Object.keys(actions)); // Run above code when 'id' object changes

	// INITIALISATION CODE
	datasets.forEach((geo) => {
		getData(`./data/data_${geo}.csv`).then((arr) => {
			let meta = arr.map((d) => ({
				code: d.code,
				name: d.name,
				parent: d.parent ? d.parent : null,
			}));
			let lookup = {};
			meta.forEach((d) => {
				lookup[d.code] = d;
			});
			metadata[geo].array = meta;
			metadata[geo].lookup = lookup;

			let indicators = arr.map((d, i) => ({
				...meta[i],
				area: d.area,
				pop: d["2020"],
				density: d.density,
				age_med: d.age_med,
			}));

			if (geo == "district") {
				["density", "age_med"].forEach((key) => {
					let values = indicators
						.map((d) => d[key])
						.sort((a, b) => a - b);
					let breaks = getBreaks(values);
					indicators.forEach(
						(d, i) =>
							(indicators[i][key + "_color"] = getColor(
								d[key],
								breaks,
								colors.seq,
							)),
					);
				});
			}
			data[geo].indicators = indicators;

			let years = [
				2001, 2002, 2003, 2004, 2005, 2006, 2007, 2008, 2009, 2010,
				2011, 2012, 2013, 2014, 2015, 2016, 2017, 2018, 2019, 2020,
			];

			let timeseries = [];
			arr.forEach((d) => {
				years.forEach((year) => {
					timeseries.push({
						code: d.code,
						name: d.name,
						value: d[year],
						year,
					});
				});
			});
			data[geo].timeseries = timeseries;
		});
	});

	getTopo(topojson, "geog").then((geo) => {
		geo.features.sort((a, b) =>
			a.properties.AREANM.localeCompare(b.properties.AREANM),
		);
		geojson = geo;
	});
	// $: console.log(geojson);

	let mygeojson;
	const myjson_path = "./data/all_countries.json";
	getGeo(myjson_path).then((geo) => {
		mygeojson = geo;
	});
	
	

	//change date format to GMT
	let parser = d3.timeParse("%Y-%m-%d");
	let just_year_parser = d3.timeParse("%Y");
	let russia, china;
	let beeswarm_data;
	let areachart_data;
	let donut_data;
	let barchart_data;
	let path = [
		"./data/agts_rus_china.csv",
		"./data/paax_practical_third_labelled_signatories.csv",
	];

	getCSV(path).then((data) => {
		//prepare data for beeswarm
		let bees = data[0];

		bees.forEach(function (d) {
			d.AgtId = +d.AgtId;
			d.dat = d.date;
			d.date = parser(d.date);
		});

		let four_group = d3.groups(bees, (d) => d.global_actor);
		russia = four_group[0][1];
		china = four_group[1][1];

		let year_division = d3.groups(
			bees,
			(d) => d.AgtId,
			(d) => d.date,
		);
		year_division.sort(function (x, y) {
			return d3.ascending(x[1][0][0], y[1][0][0]);
		});

		beeswarm_data = year_division;

		//prepare data for area chart
		let area = data[1];

		let all_year_agt = d3.groups(
			area,
			(d) => +d.year,
			(d) => d.AgtId,
		);

		all_year_agt.sort(function (x, y) {
			return d3.ascending(x[0], y[0]);
		});

		let all_sorted = [];

		all_year_agt.forEach(function (d) {
			all_sorted.push({
				date: just_year_parser(d[0]),
				value: d[1].length,
			});
		});

		areachart_data = all_sorted;

		//prepare data for donut chart
		let donut = d3.groups(
			russia,
			(d) => d.stage_label,
			(d) => d.AgtId,
		);
		donut_data = donut;

		//prepare data for bar chart
		//generate objects with percentages
		const object_calc = function (data) {
			let totalagts = 0;
			data.forEach(function (d) {
				totalagts += d[1].length;
			});
			let all_percent_object = [];
			data.forEach(function (d) {
				all_percent_object.push({
					stage: d[0],
					percentage: (d[1].length / totalagts) * 100,
				});
			});
			return all_percent_object;
		};
		let ru_stage_agt = d3.groups(
			russia,
			(d) => d.stage_label,
			(d) => d.AgtId,
		);
		const all_stage_agt = d3.groups(
			area,
			(d) => d.stage_label,
			(d) => d.AgtId,
		);
		let all_percent_bar = object_calc(all_stage_agt);
		let ru_percent_bar = object_calc(ru_stage_agt);

		//prepare barchart data
		const comb_chart = all_percent_bar.map((obj1) => {
			const obj2 = ru_percent_bar.find(
				(obj2) => obj2.stage === obj1.stage,
			);
			return { ...obj1, chart_data: obj2 ? obj2.percentage : 0 };
		});

		// Renamed attribute names
		const fin_comb_chart = comb_chart.map((obj) => ({
			group: obj.stage,
			All: obj.percentage,
			Russia: obj.chart_data,
		}));
		//fix names of subgroups
		fin_comb_chart.forEach((obj) => {
			if (obj.group === "Framework-substantive, partial") {
				obj.group = "Partial";
			} else if (obj.group === "Framework-substantive, comprehensive") {
				obj.group = "Comprehensive";
			}
		});

		barchart_data = fin_comb_chart;
	});
	// $: console.log(donut_data);
</script>

<!-- <ONSHeader filled={true} center={false} /> -->

<Header bgcolor="black" bgfixed={true} theme="dark" center={true} short={true}>
	<h1>PeaceRep</h1>
	<p class="text-big" style="margin-top: 5px">
		Third-Party Actors in Peace Agreements
	</p>
	<p>
		<!-- <Toggle
			label="Animation {animation ? 'on' : 'off'}"
			mono={true}
			bind:checked={animation}
		/> -->
	</p>
	<div style="margin-top: 90px;">
		<Arrow color="white" {animation}>Scroll to begin</Arrow>
	</div>
</Header>

<Filler theme="dark" short={true} wide={true} center={true}>
	<p class="text-big">Timeline of Agreements</p>
</Filler>

<!-- <Section>
	<h2>This is a section title</h2>
	<p>
		This is a short paragraph of text to demonstrate the standard "medium" column width, font size and line spacing of the template.
	</p>
	<p>
		This is a second short paragraph of text to demonstrate the size of the paragraph spacing in the template.
	</p>
	<blockquote class="text-indent">
		"This is an example of a large embedded quotation."&mdash;A. Person
	</blockquote>
</Section> -->

<!-- <Divider/> -->

<!-- {#if data.region.indicators}
	<Media col="medium" caption="Source: ONS mid-year population estimates.">
		<div class="chart-sml">
			<BarChart
				data={[...data.region.indicators].sort((a, b) => a.pop - b.pop)}
				xKey="pop"
				yKey="name"
				snapTicks={false}
				xFormatTick={(d) => d / 1e6}
				xSuffix="m"
				height={350}
				padding={{ top: 0, bottom: 15, left: 140, right: 0 }}
				area={false}
				title="Population by region/nation, 2020"
			/>
		</div>
	</Media>
{/if} -->

<!-- <Section>
	<h2>Gridded charts or media</h2>
	<p>
		Below is a grid that can contain charts or any other kind of visual media. The grid can fit in a medium, wide or full-width column, and the media width itself can be narrow (min 200px), medium (min 300px), wide (min 500px) or full-width. The grid is responsive, and will re-flow on smaller screens.
	</p>
</Section> -->

<!-- {#if data.region.timeseries && data.region.indicators}
<Media
	col="wide"
	grid="narrow" gap={20}
	caption="Source: ONS mid-year population estimates."
>
	{#each [...data.region.indicators].sort((a, b) => b.pop - a.pop) as region}
	<div class="chart-sml">
		<LineChart
			data={data.region.timeseries}
			xKey="year" yKey="value" zKey="code"
			color="lightgrey"
			lineWidth={1} xTicks={2} snapTicks={false}
			yFormatTick={d => (d / 1e6)} ySuffix="m"
			height={200} padding={{top: 0, bottom: 20, left: 30, right: 15}}
			selected={region.code}
			area={false} title={region.name}/>
	</div>
	{/each}
</Media>
{/if} -->

<Scroller {threshold} bind:id={id["chart"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if beeswarm_data}
					<div class="chart">
						<Beeswarm {beeswarm_data} {just_year_parser}/>
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				<p>
					This chart shows the <strong
						>area in square kilometres</strong
					> of each local authority district in the UK. Each circle represents
					one district. The scale is logarithmic.
				</p>
			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<p>
					The radius of each circle shows the <strong
						>total population</strong
					> of the district.
				</p>
			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				<p>
					The vertical axis shows the <strong>density</strong> of the district
					in people per hectare.
				</p>
			</div>
		</section>
		<section data-id="chart04">
			<div class="col-medium">
				<p>
					The colour of each circle shows the <strong
						>part of the country</strong
					> that the district is within.
				</p>
			</div>
		</section>
		<section data-id="chart05">
			<div class="col-medium">
				<h3>Select a district</h3>
				<p>
					Use the selection box below or click on the chart to select
					a district. The chart will also highlight the other
					districts in the same part of the country.
				</p>
				{#if geojson}
					<p>
						<select bind:value={selected}>
							<option value={null}>Select one</option>
							{#each geojson.features as place}
								<option value={place.properties.AREACD}>
									{place.properties.AREANM}
								</option>
							{/each}
						</select>
					</p>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

<Filler theme="dark" short={true} wide={true} center={true}>
	<p class="text-big">
		Russia and other UN Security Council Permanent Members
	</p>
</Filler>

<Scroller {threshold} bind:id={id["chart"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if areachart_data}
					<div class="chart">
						<Area {areachart_data} {just_year_parser} />
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				<p>
					This chart shows the <strong
						>area in square kilometres</strong
					> of each local authority district in the UK. Each circle represents
					one district. The scale is logarithmic.
				</p>
			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<p>
					The radius of each circle shows the <strong
						>total population</strong
					> of the district.
				</p>
			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				<p>
					The vertical axis shows the <strong>density</strong> of the district
					in people per hectare.
				</p>
			</div>
		</section>
		<section data-id="chart04">
			<div class="col-medium">
				<p>
					The colour of each circle shows the <strong
						>part of the country</strong
					> that the district is within.
				</p>
			</div>
		</section>
		<section data-id="chart05">
			<div class="col-medium">
				<h3>Select a district</h3>
				<p>
					Use the selection box below or click on the chart to select
					a district. The chart will also highlight the other
					districts in the same part of the country.
				</p>
				{#if geojson}
					<p>
						<select bind:value={selected}>
							<option value={null}>Select one</option>
							{#each geojson.features as place}
								<option value={place.properties.AREACD}>
									{place.properties.AREANM}
								</option>
							{/each}
						</select>
					</p>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

<Filler theme="dark" short={true} wide={true} center={true}>
	<p class="text-big">Agreement Stages</p>
</Filler>

<Scroller {threshold} bind:id={id["chart"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if donut_data}
					<div class="chart">
						<Donut {donut_data} />
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				<p>
					This chart shows the <strong
						>area in square kilometres</strong
					> of each local authority district in the UK. Each circle represents
					one district. The scale is logarithmic.
				</p>
			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<p>
					The radius of each circle shows the <strong
						>total population</strong
					> of the district.
				</p>
			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				<p>
					The vertical axis shows the <strong>density</strong> of the district
					in people per hectare.
				</p>
			</div>
		</section>
		<section data-id="chart04">
			<div class="col-medium">
				<p>
					The colour of each circle shows the <strong
						>part of the country</strong
					> that the district is within.
				</p>
			</div>
		</section>
		<section data-id="chart05">
			<div class="col-medium">
				<h3>Select a district</h3>
				<p>
					Use the selection box below or click on the chart to select
					a district. The chart will also highlight the other
					districts in the same part of the country.
				</p>
				{#if geojson}
					<p>
						<select bind:value={selected}>
							<option value={null}>Select one</option>
							{#each geojson.features as place}
								<option value={place.properties.AREACD}>
									{place.properties.AREANM}
								</option>
							{/each}
						</select>
					</p>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

<Filler theme="dark" short={true} wide={true} center={true}>
	<p class="text-big">Comparison of Agreement Stages</p>
</Filler>

<Scroller {threshold} bind:id={id["chart"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if barchart_data}
					<div class="chart">
						<Bars {barchart_data} />
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				<p>
					This chart shows the <strong
						>area in square kilometres</strong
					> of each local authority district in the UK. Each circle represents
					one district. The scale is logarithmic.
				</p>
			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<p>
					The radius of each circle shows the <strong
						>total population</strong
					> of the district.
				</p>
			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				<p>
					The vertical axis shows the <strong>density</strong> of the district
					in people per hectare.
				</p>
			</div>
		</section>
		<section data-id="chart04">
			<div class="col-medium">
				<p>
					The colour of each circle shows the <strong
						>part of the country</strong
					> that the district is within.
				</p>
			</div>
		</section>
		<section data-id="chart05">
			<div class="col-medium">
				<h3>Select a district</h3>
				<p>
					Use the selection box below or click on the chart to select
					a district. The chart will also highlight the other
					districts in the same part of the country.
				</p>
				{#if geojson}
					<p>
						<select bind:value={selected}>
							<option value={null}>Select one</option>
							{#each geojson.features as place}
								<option value={place.properties.AREACD}>
									{place.properties.AREANM}
								</option>
							{/each}
						</select>
					</p>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

<!-- <Divider />

<Section>
	<h2>This is a full-width chart demo</h2>
	<p>
		Below is an example of a media grid where the column with is set to "full". This allows for full width images and charts.
	</p>
	<p>

	</p>
</Section>

<Media
	col="full"
	height={600}
	caption='This is an optional caption for the above chart or media. It can contain HTML code and <a href="#">hyperlinks</a>, and wrap onto multiple lines.'
>
	<div class="chart-full">
		{#if data.district.timeseries}
		<LineChart
			data={data.district.timeseries}
			padding={{left: 50, right: 150, top: 0, bottom: 0}}
			height="500px"
			xKey="year" yKey="value" zKey="code"
			color="lightgrey" lineWidth={1}
			yFormatTick={d => (d/1e6).toFixed(1)} ySuffix="m"
			select {selected} on:select={doSelect}
			hover {hovered} on:hover={doHover}
			highlighted={chartHighlighted}
			colorSelect="#206095" colorHighlight="#999"
			area={false} title="Mid-year population by district, 2001 to 2020"
			labels labelKey="name"/>
		{/if}
	</div>
</Media>

<Divider />-->

<Filler theme="dark" short={true} wide={true} center={true}>
	<p class="text-big">Geography</p>
</Filler>

{#if mygeojson}
<Scroller {threshold} bind:id={id['map']}>
	<div slot="background">
		<figure>
			<div class="col-full height-full">
				<Globe {mygeojson} />
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="map01">
			<div class="col-medium">
				<p>
					This map shows <strong>population density</strong> by district. Districts are coloured from <Em color={colors.seq[0]}>least dense</Em> to <Em color={colors.seq[4]}>most dense</Em>. You can hover to see the district name and density.
				</p>
			</div>
		</section>
		<section data-id="map02">
			<div class="col-medium">
				<p>
					The map now shows <strong>median age</strong>, from <Em color={colors.seq[0]}>youngest</Em> to <Em color={colors.seq[4]}>oldest</Em>.
				</p>
			</div>
		</section>
		<section data-id="map03">
			<div class="col-medium">
				{#each [[...data.district.indicators].sort((a, b) => b.age_med - a.age_med)[0]] as district}
				<p>
					The map is now zoomed on <Em color={district.age_med_color}>{district.name}</Em>, the district with the oldest median age, {district.age_med} years.
				</p>
				{/each}
			</div>
		</section>
		<section data-id="map04">
			<div class="col-medium">
				<h3>Select a district</h3>
				<p>Use the selection box below or click on the map to select and zoom to a district.</p>
				{#if geojson}
					<p>
						<select bind:value={selected} on:change={() => fitById(selected)}>
							<option value={null}>Select one</option>
							{#each geojson.features as place}
								<option value={place.properties.AREACD}>
									{place.properties.AREANM}
								</option>
							{/each}
						</select>
					</p>
				{/if}
			</div>
		</section>
	</div>
</Scroller>
{/if}

<Section>
	<h2>This is a dynamic map section</h2>
	<p class="mb">
		The map below will respond to the captions as you scroll down. The scroller is not set to splitscreen, so captions are placed over the map on any screen size.
	</p>
</Section>

<!-- {#if geojson && data.district.indicators}
<Scroller {threshold} bind:id={id['map']}>
	<div slot="background">
		<figure>
			<div class="col-full height-full">
				<Map style={mapstyle} bind:map interactive={false} location={{bounds: mapbounds.uk}}>
					<MapSource
					  id="lad"
					  type="geojson"
					  data={geojson}
					  promoteId="AREACD"
					  maxzoom={13}>
					  <MapLayer
					  	id="lad-fill"
							idKey="code"
							colorKey={mapKey + "_color"}
					  	data={data.district.indicators}
					  	type="fill"
							select {selected} on:select={doSelect} clickIgnore={!explore}
							hover {hovered} on:hover={doHover}
							highlight highlighted={mapHighlighted}
					  	paint={{
					  		'fill-color': ['case',
					  			['!=', ['feature-state', 'color'], null], ['feature-state', 'color'],
					  			'rgba(255, 255, 255, 0)'
					  		],
					  		'fill-opacity': 0.7
					  	}}>
								<MapTooltip content={
									hovered ? `${metadata.district.lookup[hovered].name}<br/><strong>${data.district.indicators.find(d => d.code == hovered)[mapKey].toLocaleString()} ${units[mapKey]}</strong>` : ''
								}/>
							</MapLayer>
						<MapLayer
					  	id="lad-line"
					  	type="line"
					  	paint={{
					  		'line-color': ['case',
					  			['==', ['feature-state', 'hovered'], true], 'orange',
					  			['==', ['feature-state', 'selected'], true], 'black',
					  			['==', ['feature-state', 'highlighted'], true], 'black',
					  			'rgba(255,255,255,0)'
					  		],
					  		'line-width': 2
					  	}}
				    />
				  </MapSource>
				</Map>
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="map01">
			<div class="col-medium">
				<p>
					This map shows <strong>population density</strong> by district. Districts are coloured from <Em color={colors.seq[0]}>least dense</Em> to <Em color={colors.seq[4]}>most dense</Em>. You can hover to see the district name and density.
				</p>
			</div>
		</section>
		<section data-id="map02">
			<div class="col-medium">
				<p>
					The map now shows <strong>median age</strong>, from <Em color={colors.seq[0]}>youngest</Em> to <Em color={colors.seq[4]}>oldest</Em>.
				</p>
			</div>
		</section>
		<section data-id="map03">
			<div class="col-medium">
				{#each [[...data.district.indicators].sort((a, b) => b.age_med - a.age_med)[0]] as district}
				<p>
					The map is now zoomed on <Em color={district.age_med_color}>{district.name}</Em>, the district with the oldest median age, {district.age_med} years.
				</p>
				{/each}
			</div>
		</section>
		<section data-id="map04">
			<div class="col-medium">
				<h3>Select a district</h3>
				<p>Use the selection box below or click on the map to select and zoom to a district.</p>
				{#if geojson}
					<p>
						<select bind:value={selected} on:change={() => fitById(selected)}>
							<option value={null}>Select one</option>
							{#each geojson.features as place}
								<option value={place.properties.AREACD}>
									{place.properties.AREANM}
								</option>
							{/each}
						</select>
					</p>
				{/if}
			</div>
		</section>
	</div>
</Scroller>
{/if} -->

<Divider />

<Section>
	<h2>How to use this template</h2>
	<p>
		You can find the source code and documentation on how to use this
		template in <a
			href="https://github.com/ONSvisual/svelte-scrolly/"
			target="_blank">this Github repo</a
		>.
	</p>
</Section>

<ONSFooter />

<style>
	/* Styles specific to elements within the demo */
	:global(svelte-scroller-foreground) {
		pointer-events: none !important;
	}
	:global(svelte-scroller-foreground section div) {
		pointer-events: all !important;
	}
	select {
		max-width: 350px;
	}
	.chart {
		margin-top: 45px;
		width: calc(100% - 5px);
	}
	.chart-full {
		margin: 0 20px;
	}
	.chart-sml {
		font-size: 0.85em;
	}
	/* The properties below make the media DIVs grey, for visual purposes in demo */
	.media {
		background-color: #f0f0f0;
		display: -webkit-box;
		display: -ms-flexbox;
		display: flex;
		-webkit-box-orient: vertical;
		-webkit-box-direction: normal;
		-ms-flex-flow: column;
		flex-flow: column;
		-webkit-box-pack: center;
		-ms-flex-pack: center;
		justify-content: center;
		text-align: center;
		color: #aaa;
	}
</style>
