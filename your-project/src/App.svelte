<script>
	// CORE IMPORTS
	import { setContext, onMount } from "svelte";
	import { getMotion } from "./utils.js";
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
	import slider from "./ui/slider.svelte";


	// DEMO-SPECIFIC IMPORTS
	import bbox from "@turf/bbox";
	import { getData, setColors, getTopo, getBreaks, getColor } from "./utils.js";
	import { colors, units } from "./config.js";
	import { ScatterChart, LineChart, BarChart } from "@onsvisual/svelte-charts";
	import { Map, MapSource, MapLayer, MapTooltip } from "@onsvisual/svelte-maps";
  import { each, text } from "svelte/internal";

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
		idPrev = {...id};
	});

	// DEMO-SPECIFIC CONFIG
	// Constants
	const datasets = ["region", "district"];
	const topojson = "./data/geo_lad2021.json";
	const mapstyle = "https://bothness.github.io/ons-basemaps/data/style-omt.json";
	const mapbounds = {
		uk: [
			[-9, 49 ],
			[ 2, 61 ]
		]
	};

	// Data
	let data = {district: {}, region: {}};
	let metadata = {district: {}, region: {}};
	let geojson;

	// Element bindings
	let map = null; // Bound to mapbox 'map' instance once initialised

	// State
	let hovered; // Hovered district (chart or map)
	let selected; // Selected district (chart or map)
	$: region = selected && metadata.district.lookup ? metadata.district.lookup[selected].parent : null; // Gets region code for 'selected'
	$: chartHighlighted = metadata.district.array && region ? metadata.district.array.filter(d => d.parent == region).map(d => d.code) : []; // Array of district codes in 'region'
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
			map.fitBounds(bounds, {animate: animation, padding: 30});
		}
	}
	function fitById(id) {
		if (geojson && id) {
			let feature = geojson.features.find(d => d.properties.AREACD == id);
			let bounds = bbox(feature.geometry);
			fitBounds(bounds);
		}
	}

	// Actions for Scroller components
	const actions = {
		map: { // Actions for <Scroller/> with id="map"
			map01: () => { // Action for <section/> with data-id="map01"
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
				let hl = [...data.district.indicators].sort((a, b) => b.age_med - a.age_med)[0];
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
			}
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
			}
		}
	};

	// Code to run Scroller actions when new caption IDs come into view
	function runActions(codes = []) {
		codes.forEach(code => {
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
	datasets.forEach(geo => {
		getData(`./data/data_${geo}.csv`)
		.then(arr => {
			let meta = arr.map(d => ({
				code: d.code,
				name: d.name,
				parent: d.parent ? d.parent : null
			}));
			let lookup = {};
			meta.forEach(d => {
				lookup[d.code] = d;
			});
			metadata[geo].array = meta;
			metadata[geo].lookup = lookup;

			let indicators = arr.map((d, i) => ({
				...meta[i],
				area: d.area,
				pop: d['2020'],
				density: d.density,
				age_med: d.age_med
			}));

			if (geo == "district") {
				['density', 'age_med'].forEach(key => {
					let values = indicators.map(d => d[key]).sort((a, b) => a - b);
					let breaks = getBreaks(values);
					indicators.forEach((d, i) => indicators[i][key + '_color'] = getColor(d[key], breaks, colors.seq));
				});
			}
			data[geo].indicators = indicators;


			let timeseries = [];
			arr.forEach(d => {
				years.forEach(year => {
					timeseries.push({
						code: d.code,
						name: d.name,
						value: d[year],
						year
					});
				});
			});
			data[geo].timeseries = timeseries;
		});
	});

	getTopo(topojson, 'geog')
	.then(geo => {
		geo.features.sort((a, b) => a.properties.AREANM.localeCompare(b.properties.AREANM));
		geojson = geo;
	});

	window.onscroll = function () {
    scrollRotate();
};

function scrollRotate() {
    let image = document.getElementById("planete");
    image.style.transform = "rotate(" + window.pageYOffset/6 + "deg)";
}

let percentage = 50;

$: response = "";
let answer ="";


function handleClick() {
		if(percentage == 5){
			response = "Bravo";
		} else if ( percentage < 49 ){
			response= 'pas loin';
		} else if (percentage >= 50){
			response = 'oups ';
		}
		answer = "5%";
	}

	$: number = 1;
	$: image = 'bonhomme_tous-12.svg';

	function backgoundClick(number){
		if (number== 1){
			image = 'bonhomme_tous-12.svg';
		} else if (number == 2){
			image = 'bonhomme_tous2-12.svg';
		} else if ( number == 3){
			image = 'bonhomme_tous3-12.svg';
		} else if (number == 4) {
			image = 'bonhomme_tous4-12.svg';
		}
	}


</script>


<Header bgcolor="#F9F8F4" bgfixed={true} theme="dark" center={false} short={true}>
	<img src="/img/titre-07.svg" alt=""/>

	<p class="text-big" style="margin-top: 5px">
		This is a short text description of the article that might take up a couple of lines
	</p>
	<img id="planete" src="/img/planete-06.svg" alt="scroll"/>
	
</Header>


	<div>
	<h2 class="center">Selon vous, quel est le pourcentage de cours qui traitent des enjeux climatiques et environnementaux dans les universités et hautes écoles belges ?  </h2>
	</div>
	<div>
		<p class="center">{percentage} %</p>
	<input type=range bind:value={percentage} min=0 max=100 class="center">
	</div>
	<button on:click={handleClick} id="close-image" class='testbutton'>
		confirmer
	</button>
	<p> {response}</p>
	<p class="big center"> {answer}</p>


<Scroller {threshold} bind:id={id['chart']} splitscreen={true}>
	<div slot="background">
		<figure>
			<img id="planete" src="/img/text_documenet-10.svg" alt="scroll" class='text'/>
		

		</figure>
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				<p>
					texte de loi
				</p>
			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<img id="planete" src="/img/loi-09.svg" alt="scroll"/>

			</div>
	</div>
</Scroller>
	

<Scroller {threshold} bind:id={id['chart']} splitscreen={true}>
	<div slot="background">
		<figure>
			<img id="planete" src="/img/{image}" alt="scroll" class='bonhomme center'/>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				<p class='center'>
				image 1
				</p>
				<button  on:click={backgoundClick(2)} class='center'>click</button>

			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<p class='center'>
					image 2
				</p>
				<button  on:click={backgoundClick(3)} class='center'>click</button>

			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				<p class="center">
					image 3
				</p>
				<button  on:click={backgoundClick(4)} class='center'>click</button>

			</div>
		</section>
		<section data-id="chart04">
			<div class="col-medium">
				<p class='center'>
					image 4
				</p>
				<button  on:click={backgoundClick(4)} class='center'>click</button>

			</div>		
	</div>
</Scroller>

<div>
<img id="planete" src="/img/region_comparaison-05.svg" alt="scroll" class='center'/>
</div>



<ONSFooter />

<style>

	@font-face{
		font-family: "LunchType_Medium_Expanded";
		src : url(/fonts/Lunchtype24-Medium-Expanded.ttf);
	}
	@font-face{
		font-family: "Avara-black";
		src : url(/fonts/Avara-Black.ttf);
	}
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
	h1 {
		color:#048D14;
		font-family: "Avara-Black", sans-serif;
	}

	h2 {
		color:black;
		font-family: "Avara-Black", sans-serif;
		font-size: 20px;
		text-transform: uppercase;
		margin-bottom: 1cm;
	}
	.testbutton {
  font-family: "Avara-black";
  color: #048D14 !important;
  font-size: 18px;
  text-shadow: 0px 0px 0px #048D14;
  padding: 10px 10px;
  border-radius: 10px;
  border: 4px solid #048D14;
  background: #F9F8F4;
  text-transform: uppercase;
  display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;
}
.testbutton:hover {
  color: #F9F8F4 !important;
  background: #048D14;
}

	p{
		color : black;
		font-size: 18px;
		font-family: "LunchType_Medium_Expanded";
		display: block;
  		margin-left: auto;
		margin-right: auto;
		width: 75%;
	}
	.chart {
		margin-top: 45px;
		width: calc(100% - 5px);
	}

.center {
  display: block;
  margin-left: auto;
  margin-right: auto;
  width: 75%;
}
input[type=range]:focus {
  outline: none;
}
input[type=range]::-webkit-slider-runnable-track {
  width: 100%;
  height: 23px;
  cursor: pointer;
  background: #FFFFFF;
  border-radius: 24px;
  border: 2px solid #000000;
}
input[type=range]::-webkit-slider-thumb {
  box-shadow: 0px 0px 1px #000000;
  border: 1px solid #000000;
  height: 23px;
  width: 23px;
  border-radius: 12px;
  background: #000000;
  cursor: pointer;
  -webkit-appearance: none;
  margin-top: 1.5px;
}
input[type=range]:focus::-webkit-slider-runnable-track {
  background: #FFFFFF;
}
input[type=range]::-moz-range-track {
  width: 100%;
  height: 29px;
  cursor: pointer;
  animate: 0.2s;
  box-shadow: 0px 0px 0px #000000;
  background: #FFFFFF;
  border-radius: 24px;
  border: 2px solid #000000;
}
input[type=range]::-moz-range-thumb {
  box-shadow: 0px 0px 1px #000000;
  border: 1px solid #000000;
  height: 23px;
  width: 23px;
  border-radius: 12px;
  background: #000000;
  cursor: pointer;
}
input[type=range]::-ms-track {
  width: 100%;
  height: 29px;
  cursor: pointer;
  animate: 0.2s;
  background: transparent;
  border-color: transparent;
  color: transparent;
}
input[type=range]::-ms-fill-lower {
  background: #FFFFFF;
  border: 2px solid #000000;
  border-radius: 48px;
  box-shadow: 0px 0px 0px #000000;
}
input[type=range]::-ms-fill-upper {
  background: #FFFFFF;
  border: 2px solid #000000;
  border-radius: 48px;
  box-shadow: 0px 0px 0px #000000;
}
input[type=range]::-ms-thumb {
  margin-top: 1px;
  box-shadow: 0px 0px 1px #000000;
  border: 1px solid #000000;
  height: 4vw;
  width: 4vw;
  border-radius: 12px;
  background: #000000;
  cursor: pointer;
}
input[type=range]:focus::-ms-fill-lower {
  background: #FFFFFF;
}
input[type=range]:focus::-ms-fill-upper {
  background: #FFFFFF;
}
.big {
	color:#048D14;
	font-size: 250px;
	font-family: "Avara-black";

}
.bonhomme {
	width: 100%;
	margin-top: 1.5cm;
	margin-bottom: auto;
	display :block;
	margin-left: auto;
	margin-right: auto;

}
.text {
	margin-top: 2cm;
	margin-bottom: 2cm;
	display: block;
  	margin-left: 1cm;
  	margin-right: auto;
	width :200%;
	opacity: 0.5;
	
}
</style>
