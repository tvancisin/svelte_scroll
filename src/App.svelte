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
	import Dropdown from "./ui/Dropdown.svelte";
	import * as d3 from "d3";
	import Beeswarm from "./vis/Beeswarm.svelte";
	import Area from "./vis/Area.svelte";
	import Donut from "./vis/Donut.svelte";
	import Bars from "./vis/Bars.svelte";
	import Globe from "./vis/Globe.svelte";

	// DEMO-SPECIFIC IMPORTS
	import bbox from "@turf/bbox";
	import { setColors, getGeo } from "./utils.js";

	let selectedCountry = "Russia";

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

	let step = null;

	// Actions for Scroller components
	const actions = {
		map: {
			// Actions for <Scroller/> with id="map"
			map01: () => {
				// fitBounds(mapbounds.uk);
				// mapKey = "density";
				// mapHighlighted = [];
				// explore = false;
			},
			map02: () => {
				if (selectedCountry == "Russia") {
					map.flyTo({
						center: [70, 40],
						essential: true, // this animation is considered essential with respect to prefers-reduced-motion
					});
				}
				if (selectedCountry == "China") {
					map.flyTo({
						center: [100, 20],
						essential: true, // this animation is considered essential with respect to prefers-reduced-motion
					});
				}
				// fitBounds(mapbounds.uk);
				// mapKey = "age_med";
				// mapHighlighted = [];
				// explore = false;
			},
		},
		chart: {
			chart01: () => {
				step = "one";
			},
			chart02: () => {
				step = "two";
			},
			chart03: () => {
				step = "three";
			},
		},
		area: {
			area01: () => {
				console.log("area");
			},
		},
		donut: {
			donut01: () => {
				console.log("donut");
			},
		},
		bar: {
			bar01: () => {
				console.log("bar1");
			},
			bar02: () => {
				console.log("bar2");
			},
			bar03: () => {
				console.log("bar3");
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

	//LOAD GEOJSON
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
	let russia_beeswarm_data;
	let china_beeswarm_data;

	let donut_data;
	let russia_donut;
	let china_donut;

	let barchart_data;
	let russia_barchart_data;
	let china_barchart_data;

	let areachart_data;
	let linechart_data = [];

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

		let year_division_russia = d3.groups(
			russia,
			(d) => d.AgtId,
			(d) => d.date,
		);
		year_division_russia.sort(function (x, y) {
			return d3.ascending(x[1][0][0], y[1][0][0]);
		});

		let year_division_china = d3.groups(
			china,
			(d) => d.AgtId,
			(d) => d.date,
		);
		year_division_china.sort(function (x, y) {
			return d3.ascending(x[1][0][0], y[1][0][0]);
		});

		russia_beeswarm_data = year_division_russia;
		china_beeswarm_data = year_division_china;

		beeswarm_data = russia_beeswarm_data; // final beeswarm data

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

		areachart_data = all_sorted; // final areachart_data

		//prepare data for linechart
		const act_group = d3.groups(
			bees,
			(d) => d.global_actor,
			(d) => +d.year,
			(d) => d.AgtId,
		);
		act_group.forEach(function (d) {
			d[1].sort(function (x, y) {
				return d3.ascending(x[0], y[0]);
			});
		});

		act_group.forEach(function (d) {
			d[1].forEach(function (m) {
				linechart_data.push({
					// final linechart data
					division: d[0],
					date: just_year_parser(m[0]),
					unemployment: m[1].length,
				});
			});
		});

		//prepare data for donut chart
		russia_donut = d3.groups(
			russia,
			(d) => d.stage_label,
			(d) => d.AgtId,
		);
		china_donut = d3.groups(
			china,
			(d) => d.stage_label,
			(d) => d.AgtId,
		);

		donut_data = russia_donut;

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
		let cn_stage_agt = d3.groups(
			china,
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
		let cn_percent_bar = object_calc(cn_stage_agt);

		//prepare barchart data
		const ru_comb_chart = all_percent_bar.map((obj1) => {
			const obj2 = ru_percent_bar.find(
				(obj2) => obj2.stage === obj1.stage,
			);
			return { ...obj1, chart_data: obj2 ? obj2.percentage : 0 };
		});

		const cn_comb_chart = all_percent_bar.map((obj1) => {
			const obj2 = cn_percent_bar.find(
				(obj2) => obj2.stage === obj1.stage,
			);
			return { ...obj1, chart_data: obj2 ? obj2.percentage : 0 };
		});

		// Renamed attribute names
		const ru_fin_comb_chart = ru_comb_chart.map((obj) => ({
			group: obj.stage,
			All: obj.percentage,
			Russia: obj.chart_data,
		}));
		const cn_fin_comb_chart = cn_comb_chart.map((obj) => ({
			group: obj.stage,
			All: obj.percentage,
			China: obj.chart_data,
		}));

		//fix names of subgroups
		ru_fin_comb_chart.forEach((obj) => {
			if (obj.group === "Framework-substantive, partial") {
				obj.group = "Partial";
			} else if (obj.group === "Framework-substantive, comprehensive") {
				obj.group = "Comprehensive";
			}
		});

		cn_fin_comb_chart.forEach((obj) => {
			if (obj.group === "Framework-substantive, partial") {
				obj.group = "Partial";
			} else if (obj.group === "Framework-substantive, comprehensive") {
				obj.group = "Comprehensive";
			}
		});

		russia_barchart_data = ru_fin_comb_chart;
		china_barchart_data = cn_fin_comb_chart;

		barchart_data = russia_barchart_data;
	});

	function handleCountry(country) {
		if (country.detail == "Russia") {
			beeswarm_data = russia_beeswarm_data;
			donut_data = russia_donut;
			barchart_data = russia_barchart_data;
		} else {
			beeswarm_data = china_beeswarm_data;
			donut_data = china_donut;
			barchart_data = china_barchart_data;
		}
	}
</script>

<ONSHeader center={false} />

<Dropdown on:close={handleCountry} bind:selectedCountry />

<Header
	bgcolor="white"
	bgfixed={true}
	theme="light"
	center={true}
	short={false}
>
	<h1>PeaceRep</h1>
	<p class="text-big" style="margin-top: 5px">
		Third-Party Actors in Peace Agreements
	</p>
	<div style="margin-top: 90px;">
		<Arrow color="black" {animation}>Scroll to begin</Arrow>
	</div>
</Header>

<Filler theme="light" short={true} wide={true} center={true}>
	<p class="text-big">Timeline of Agreements</p>
</Filler>

<Scroller {threshold} bind:id={id["chart"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if beeswarm_data}
					<div class="chart">
						<Beeswarm
							{beeswarm_data}
							{just_year_parser}
							{selectedCountry}
							{step}
						/>
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				{#if selectedCountry === "Russia"}
					<p>
						Russia is the second most prolific international
						third-party signatory of peace agreements between
						1990-2022. It follows the United Nations, and comes
						ahead of the United States, the African Union, and the
						European Union.
					</p>
					<div id="legend">
						<span class="dot"></span>
						<span id="legend_text"
							>Individual peace agreements signed by Russia (hover
							over for more detail)</span
						>
					</div>
				{:else if selectedCountry === "China"}
					<p>
						China is not the most prolific third-party signatory of
						peace agreements since 1990, ranking 15th of all actors,
						who have acted as third-party signatories. In terms of
						frequency, this puts it alongside actors such as Egypt,
						Kenya, and Nigeria.
					</p>
					<div id="legend">
						<span class="dot"></span>
						<span id="legend_text"
							>Individual peace agreements signed by China (hover
							over for more detail)</span
						>
					</div>
				{/if}
			</div>
		</section>

		<section data-id="chart02">
			<div class="col-medium">
				{#if selectedCountry === "Russia"}
					<p>
						Russia has most often acted as a third-party signatory
						in the 1990s. Majority of these agreements relate to the
						dissolution of the Soviet Union. Many of these are
						protracted conflicts, where Russia continues acting as a
						third - party signatory of peace agreements.
					</p>
					<div id="legend">
						<span class="dot" style="background-color: red;"></span>
						<span id="legend_text"
							>Peace agreements addressing conflicts in the former
							Soviet Union territories.</span
						>
					</div>
				{:else if selectedCountry === "China"}
					<p>
						As one of the UN Security Council (UNSC) permanent
						members, China has participated in all major
						international conferences (e.g., for Cambodia, Bosnia
						and Herzegovina, Afghanistan, Libya), with the key
						exception of negotiations relating to Israel and
						Palestine. Nearly all agreements signed by China as a
						third-party have been the result of large international
						conferences or UNSC resolutions.
					</p>
					<div id="legend">
						<span class="dot" style="background-color: red;"></span>
						<span id="legend_text"
							>Peace agreements resulting from large international
							conferences or UNSC resolutions.</span
						>
					</div>
				{/if}
			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				{#if selectedCountry === "Russia"}
					<p>
						Over the last decade, Russia increasingly acts as a
						signatory on agreements related to conflicts in Syria
						and, reflecting its increased engagements in Africa:
						Libya, and the Central African Republic. These are
						internationalised conflicts, where Russia is also
						militarily engaged in supporting conflict parties.
					</p>
					<div id="legend">
						<span class="dot" style="background-color: red;"></span>
						<span id="legend_text"
							>Peace agreements addressing conflicts in Syria,
							Libya, and the Central African Republic.</span
						>
					</div>
				{:else if selectedCountry === "China"}
					<p>
						Most agreements China has signed include the UN or other
						permanent members of the UN Security Council.
					</p>
					<div id="legend">
						<span class="dot" style="background-color: red;"></span>
						<span id="legend_text"
							>Peace agreements signed by China and the UN or all
							other permanent members of the UNSC.</span
						>
					</div>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

<Filler theme="light" short={true} wide={true} center={true}>
	<p class="text-big">
		Russia and other UN Security Council Permanent Members
	</p>
</Filler>

<Scroller {threshold} bind:id={id["area"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if areachart_data && linechart_data}
					<div class="chart">
						<Area
							{areachart_data}
							{just_year_parser}
							{linechart_data}
							{selectedCountry}
						/>
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="area01">
			<div class="col-medium">
				{#if selectedCountry === "Russia"}
					<p>
						Russia is the most prolific signatory of peace
						agreements of all UN Security Council Permanent Members.
						In a number of years (1995, 2016-2018) it has signed
						more agreements than the United Nations.
					</p>
					<div id="legend">
						<span
							class="rect"
							style="background-color: rgba(0, 100, 100, 0.3);"
						></span>
						<span id="legend_text">Overall agreements.</span>
					</div>
					<div id="legend">
						<span class="line" style="background-color: red;"
						></span>
						<span id="legend_text">Russian agreements.</span>
					</div>
					<div id="legend">
						<span
							class="line"
							style="background-color: rgb(0, 158, 158);"
						></span>
						<span id="legend_text">Chinese agreements.</span>
					</div>
				{:else if selectedCountry === "China"}
					<p>
						China is the least prolific third-party signatory of
						peace agreements of all UN Security Council permanent
						members. China signed 39 agreements as a third-party
						since 1990, in comparison to Russian 134 signatures and
						the US 132 signatures.
					</p>
					<div id="legend">
						<span
							class="rect"
							style="background-color: rgba(0, 100, 100, 0.3);"
						></span>
						<span id="legend_text">Overall agreements.</span>
					</div>
					<div id="legend">
						<span class="line" style="background-color: red;"
						></span>
						<span id="legend_text">Chinese agreements.</span>
					</div>
					<div id="legend">
						<span
							class="line"
							style="background-color: rgb(0, 158, 158);"
						></span>
						<span id="legend_text">Russian agreements.</span>
					</div>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

<Filler theme="light" short={true} wide={true} center={true}>
	<p class="text-big">Agreement Stages</p>
</Filler>

<Scroller {threshold} bind:id={id["donut"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if donut_data}
					<div class="chart">
						<Donut {donut_data} {selectedCountry} />
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="donut01">
			<div class="col-medium">
				{#if selectedCountry === "Russia"}
					<p>
						Russia primarily signs pre-negotiation and ceasefire
						agreements. These represent over half of all agreements
						signed. For more details on the categories see
						<a
							href="https://www.peaceagreements.org/files/Definitions_v7.pdf"
							target="_blank">here</a
						>.
					</p>
				{:else if selectedCountry === "China"}
					<p>
						Chinese involvement as a third-party signatory seems to
						come at the point when there is a broad international
						consensus regarding a peace process. Implementation
						agreements are the biggest category and represent 31% of
						all agreements signed by China. For more details on the
						categories see
						<a
							href="https://www.peaceagreements.org/files/Definitions_v7.pdf"
							target="_blank">here</a
						>.
					</p>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

<Filler theme="light" short={true} wide={true} center={true}>
	<p class="text-big">Comparison of Agreement Stages</p>
</Filler>

<Scroller {threshold} bind:id={id["bar"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if barchart_data}
					<div class="chart">
						<Bars {barchart_data} {selectedCountry} />
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="bar01">
			<div class="col-medium">
				{#if selectedCountry === "Russia"}
					<p>
						Compared with all agreements, Russia signs more
						pre-negotiation agreements and less comprehensive and
						implementation agreements.
					</p>
					<div id="legend">
						<span class="rect" style="background-color: black;"
						></span>
						<span id="legend_text"
							>Overall agreements (% of all).</span
						>
					</div>
					<div id="legend">
						<span class="rect" style="background-color: red;"
						></span>
						<span id="legend_text"
							>Russian signature (% of all signed by Russia).</span
						>
					</div>
				{:else if selectedCountry === "China"}
					<p>
						Compared with all agreements, China signs more
						comprehensive and implementation agreements, and less
						ceasefires and partial ones.
					</p>
					<div id="legend">
						<span class="rect" style="background-color: black;"
						></span>
						<span id="legend_text"
							>Overall agreements (% of all).</span
						>
					</div>
					<div id="legend">
						<span class="rect" style="background-color: red;"
						></span>
						<span id="legend_text"
							>Chinese signature (% of all signed by China).</span
						>
					</div>
				{/if}
			</div>
		</section>
		<section data-id="bar02">
			<div class="col-medium">
				{#if selectedCountry === "Russia"}
					<p>
						Pre-negotiation agreements represent 29% of all
						agreements with third-party signatories, but 35% of all
						agreements signed by Russia.
					</p>
					<div id="legend">
						<span class="rect" style="background-color: black;"
						></span>
						<span id="legend_text"
							>Overall agreements (% of all).</span
						>
					</div>
					<div id="legend">
						<span class="rect" style="background-color: red;"
						></span>
						<span id="legend_text"
							>Russian signature (% of all signed by Russia).</span
						>
					</div>
				{:else if selectedCountry === "China"}
					<p>
						Comprehensive agreements present only 6% of all
						agreements signed by third-parties, but amount to 10% of
						all agreements signed by China. Similarly, 31% of all
						agreements signed by China are implementation
						agreements, but the overall proportion of such
						agreements is 20%.
					</p>
					<div id="legend">
						<span class="rect" style="background-color: black;"
						></span>
						<span id="legend_text"
							>Overall agreements (% of all).</span
						>
					</div>
					<div id="legend">
						<span class="rect" style="background-color: red;"
						></span>
						<span id="legend_text"
							>Chinese signature (% of all signed by China).</span
						>
					</div>
				{/if}
			</div>
		</section>
		<section data-id="bar03">
			<div class="col-medium">
				{#if selectedCountry === "Russia"}
					<p>
						Comprehensive agreements represent 6% of all agreements
						signed, but only 4% of all agreements signed by Russia.
						Implementation agreements represent 20% of all
						agreements signed, but only 17% of all agreements signed
						by Russia.
					</p>
					<div id="legend">
						<span class="rect" style="background-color: black;"
						></span>
						<span id="legend_text"
							>Overall agreements (% of all).</span
						>
					</div>
					<div id="legend">
						<span class="rect" style="background-color: red;"
						></span>
						<span id="legend_text"
							>Russian signature (% of all signed by Russia).</span
						>
					</div>
				{:else if selectedCountry === "China"}
					<p>
						In contrast, only 8% of all agreements signed by China
						are ceasefires, with the overall proportion of such
						agreements at 19%.
					</p>
					<div id="legend">
						<span class="rect" style="background-color: black;"
						></span>
						<span id="legend_text"
							>Overall agreements (% of all).</span
						>
					</div>
					<div id="legend">
						<span class="rect" style="background-color: red;"
						></span>
						<span id="legend_text"
							>Chinese signature (% of all signed by China).</span
						>
					</div>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

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

<!-- <Section>
	<h2>Gridded charts or media</h2>
	<p>
		Below is a grid that can contain charts or any other kind of visual media. The grid can fit in a medium, wide or full-width column, and the media width itself can be narrow (min 200px), medium (min 300px), wide (min 500px) or full-width. The grid is responsive, and will re-flow on smaller screens.
	</p>
</Section> -->

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

<Filler theme="light" short={true} wide={true} center={true}>
	<p class="text-big">Geography</p>
</Filler>

{#if mygeojson}
	<Scroller {threshold} bind:id={id["map"]}>
		<div slot="background">
			<figure>
				<div class="col-full height-full">
					<Globe {mygeojson} bind:map />
				</div>
			</figure>
		</div>

		<div slot="foreground">
			<section data-id="map01">
				<div class="col-medium">
					<p>
						Russia has most often acted as a third-party signatory
						in the 1990s. Majority of these agreements relate to the
						dissolution of the Soviet Union.
					</p>
				</div>
			</section>
			<section data-id="map02">
				<div class="col-medium">
					<p>
						Russia has most often acted as a third-party signatory
						in the 1990s. Majority of these agreements relate to the
						dissolution of the Soviet Union.
					</p>
				</div>
			</section>
		</div>
	</Scroller>
{/if}

<Section>
	<h2>Research</h2>
	<p class="mb">
		Read our research on Russia and its approaches to conflict, peace
		processes and mediation.
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

<!-- <Section>
	<h2>How to use this template</h2>
	<p>
		You can find the source code and documentation on how to use this
		template in <a
			href="https://github.com/ONSvisual/svelte-scrolly/"
			target="_blank">this Github repo</a
		>.
	</p>
</Section>

<ONSFooter /> -->

<style>
	/* Styles specific to elements within the demo */
	:global(svelte-scroller-foreground) {
		pointer-events: none !important;
	}
	:global(svelte-scroller-foreground section div) {
		pointer-events: all !important;
	}

	.chart {
		margin-top: 45px;
		width: calc(100% - 5px);
	}

	.dot {
		height: 15px;
		width: 15px;
		background-color: black;
		border-radius: 50%;
		display: inline-block; /* Ensures the dot stays inline with the text */
		margin-right: 8px; /* Adjust the space between the dot and the text */
		vertical-align: middle; /* Vertically aligns the dot with the middle of the text */
	}

	#legend {
		display: flex; /* Align items horizontally */
		align-items: center; /* Vertically center the dot and text */
		padding-left: 0px;
	}

	.dot {
		height: 15px;
		width: 15px;
		background-color: black;
		border-radius: 50%; /* Ensures the element is a perfect circle */
		display: inline-block;
		margin-right: 8px; /* Space between the dot and the text */
		flex-shrink: 0; /* Prevents the dot from shrinking in a flex container */
	}

	.rect {
		height: 18px;
		width: 18px;
		display: inline-block;
		margin-right: 8px; /* Space between the dot and the text */
		flex-shrink: 0; /* Prevents the dot from shrinking in a flex container */
	}

	.line {
		height: 3px;
		width: 18px;
		display: inline-block;
		margin-right: 8px; /* Space between the dot and the text */
		flex-shrink: 0; /* Prevents the dot from shrinking in a flex container */
	}

	#legend_text {
		font-size: 14px;
		line-height: 1.5;
		margin: 0;
	}
	/* select {
		max-width: 350px;
	} */
	/* .chart-full {
		margin: 0 20px;
	}
	.chart-sml {
		font-size: 0.85em;
	} */
	/* The properties below make the media DIVs grey, for visual purposes in demo */
	/* .media {
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
	} */
</style>
