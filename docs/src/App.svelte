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
	let x = 50;
	let y = 50;
	let fleche_src = "/img/fleche1-13.svg";
	let infos = "";
	let region_url = "/img/cercle1-03.svg";



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
				index= 1;
			},
			chart02: () => {
				index = 2;
			},
			chart03: () => {
				index = 3;
			},
			chart04: () => {
				index =3;
			}
		},
		loi: {
			loi01: () => {
			},
			loi02: () => {
			},
			loi03: () => {
			},
			loi04: () => {
			},
			loi05: () => {
			}
		},
		cercle: {
			cercle01: () => {
				region_url = "/img/cercle1-03.svg";
			},
			cercle02: () => {
				region_url = "/img/cercle2-06.svg";

			},
			cercle03: () => {
				region_url = "/img/cercle3-04.svg";

			},
			cercle04: () => {
				region_url = "/img/cercle4-05.svg";

			}
		},
		fleche: {
			fleche01: () => {
				fleche_src = "/img/fleche1-13.svg";
				infos = "0%";
			},
			fleche02: () => {
				fleche_src = "/img/fleche2-14.svg";
				infos = "38%";
			}
		},
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
import {fade} from 'svelte/transition'


let percentage = 50;

$: response = "";
let answer ="";


function handleClick() {
		if(percentage == 4){
			response = "Bien joué ! Dans l’enseignement supérieur belge, la proportion de cours qui abordent les enjeux climatiques est légèrement inférieure à";
		} else if ( percentage < 10 ){
			response= ': Pas si loin ! En réalité, dans l’enseignement supérieur belge, la proportion de cours qui abordent les enjeux climatiques est légèrement inférieure à ';
		} else if (percentage >= 10){
			response = 'Oups, raté… En réalité, dans l’enseignement supérieur belge, la proportion de cours qui abordent les enjeux climatiques est légèrement inférieure à ';
		}
		answer = "4%";
	}

let src ="";
let answer3 = "";

function UNifClick() {
			src = "/img/université-10.svg"
			answer3 = "Tout à fait ! Les universités arrivent largement en tête. Si l’on s’intéresse à l’ensemble des cours qui traitent des enjeux climatiques en Belgique, on s’aperçoit que 69% sont dispensés par des universités … contre 31% seulement par des hautes écoles "
	}
function HEClick(){
			src ="/img/hauteecole-11.svg"
			answer3 = "Et non… Ce sont en fait les universités qui arrivent largement en tête. Si l’on s’intéresse à l’ensemble des cours qui traitent des enjeux climatiques en Belgique, on s’aperçoit que 69% sont dispensés par des universités … contre 31% seulement par des hautes écoles"
}

	$: number = 1;
	let image = ['/img/bonhomme_tous-12.svg','/img/bonhomme_tous2-12.svg','/img/bonhomme_tous3-12.svg','/img/bonhomme_tous4-12.svg'];
	let index = 1;

</script>
<div>
	<img id="header" src="/img/hacktualite-19.svg" alt="scroll" style="width:30%; margin-bottom:0.25cm; margin-top:0.5cm" class="center"/>

</div>

<Header bgcolor="#F9F8F4" bgfixed={true} theme="dark" center={false} short={true}>
	<img src="/img/titre-07.svg" alt=""/>

	<p class="center" style="margin-top: 5px">
		Malgré l'urgence climatique à laquelle nous sommes confrontés, les universités et hautes écoles belges semblent ne pas accorder suffisamment d'importance à la formation sur le climat. En effet, peu de cours spécifiques sont proposés aux étudiants pour leur permettre de comprendre les enjeux environnementaux actuels et d’entrer dans une nouvelle génération de professionnels avertis. Dans cet article, nous vous amenons à constater le manque cruel de présence de cours sur le climat dans l’enseignement supérieur belge. 	</p>
	<img id="planete" src="/img/planete-06.svg" alt="scroll"/>
	
</Header>

<!-- // quizz 1 -->
	<div>
	<h2 class="center" style=''>Selon vous, quel est le pourcentage de cours qui traitent des enjeux climatiques et environnementaux dans les universités et hautes écoles belges ?  </h2>
	</div>
	<div>
		<p class="center">{percentage} %</p>
	<input type=range bind:value={percentage} min=0 max=100 class="center">
	</div>
	<button on:click={handleClick} id="close-image" class='testbutton'>
		confirmer
	</button>
	<p class='center'> {response}</p>
	<p class="big center"> {answer}</p>

<!-- // texte de loi -->
<Scroller {threshold} bind:id={id['loi']} splitscreen={true}>
	<div slot="background">
		<figure>
			<img id="planete" src="/img/text_documenet-10.svg" alt="scroll" class='text'/>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="loi01">
			<div class="col-medium">
				<p class="center">En 2020, le Parlement européen a voté le Pacte vert pour l’Europe. Objectif : une Union européenne neutre en carbone d’ici 2050. </p>
			</div>
		</section>
		<section data-id="loi02">
			<div class="col-medium">
				<p class="center">Le pacte insiste notamment sur l’importance de l’enseignement pour effectuer cette transition.</p>
			</div>
		</section>

		<section data-id="loi03">
			<div class="col-medium">
					<img id="planete" src="/img/loi-09.svg" alt="scroll"/>
			</div>
		</section>

		<section data-id="loi04">
			<div class="col-medium">
				<p class="center">
					En effet, si l’on souhaite comprendre les causes du changement climatique et mettre en place des solutions pour le combattre, il est nécessaire de développer des compétences nouvelles dans tous les domaines. Cela passe notamment par les hautes écoles et les universités.
				</p>
			</div>
		</section>

		<section data-id="loi05">
			<div class="col-medium">
				<p class="center">
					Pourtant, en Belgique, on est loin de répondre à cet objectif…
				</p>
			</div>
		</section>

	</div>
</Scroller>
	
<!-- bonshommes  -->
<Scroller {threshold} bind:id={id['chart']} splitscreen={true}>
	<div slot="background">

		<h2 class="center">Population active belge</h2>
		{#each [image[index]] as src (index)}
	<img transition:fade class="bonhomme" {src} alt="" />	
	{/each}
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				<p class='center'>
					Au sein de la population active de notre pays, seulement 1 personne sur 20 a reçu un enseignement qui aborde les questions climatiques
				</p>
			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<p class='center'>
					En supposant une transformation rapide de la majorité des formations d’ici dix ans, il faudrait attendre 2045 pour passer à 1 personne sur 3. 
				</p>
			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				<p class="center">
					Ce n’est qu’en 2060 que l’on pourrait espérer voir la moitié de la population active belge formée aux questions climatiques. C’est un changement qui est donc possible, mais qui demande du travail.
				</p>
			</div>
		</section>		
	<section data-id="chart04">
		<div class="col-medium">
			<p class="center">
				Les universités et les hautes écoles, qui forment près de la moitié de la population en Belgique, doivent par conséquent améliorer leur offre de cours qui abordent les enjeux climatiques si elles souhaitent être en accord avec le Pacte vert européen. 
			</p>
			<p class="center">
				Le manque de données disponibles ne permet malheureusement pas d’analyser l’évolution de cette offre au fil du temps. Nous pouvons cependant analyser la situation actuelle. 
			</p>
		</div>
	</section>	
</div>

</Scroller>

<Scroller {threshold} bind:id={id['cercle']} splitscreen={true}>
	<div slot="background">
			<img id="planete" src={region_url} alt="scroll" class='center cercle'/>
	</div>


	<div slot="foreground">
		<section data-id="cercle01">
			<div class="col-medium">
				<p class='center moyen'>
				En région bruxelloise, à peine 3,48% des cours dispensés traitent de la question climatique.
				</p>
			</div>
		</section>
		<section data-id="cercle02">
			<div class="col-medium">
				<p class='center moyen'>
					La Wallonie suit de très près avec 3,58% de cours liés au climat.
				</p>
			</div>
		</section>
		<section data-id="cercle03">
			<div class="col-medium">
				<p class="center moyen">
					La Flandre se démarque légèrement en dépassant le seuil des 4% de cours traitant des enjeux climatiques.
				</p>
			</div>
		</section>		
	<section data-id="cercle04">
		<div class="col-medium">
			<p class="center">
				La différence entre les régions est donc minime, mais il apparaît clairement qu’elles doivent chacune faire un effort afin d’augmenter la proportion de cours qui abordent les enjeux climatiques dans l’enseignement supérieur.
			</p>
		</div>
	</section>	
</div>
</Scroller>

<div>
	<h2 class="center"style="font-size:12px;"> Là où une différence plus importante existe, c’est entre les hautes écoles et les universités.
		D’après vous, laquelle de ces deux institutions possède l’offre la plus importante de cours traitant des enjeux climatiques ? 
		</h2>
	</div>
	<div>
	<button on:click={UNifClick} id="close-image" class='testbutton' style="width:50%;">
		université
	</button>
	<button on:click={HEClick} id="close-image" class='testbutton' style="width:50%; margin-bottom:1cm;">
		Haute-école
	</button>

	<img src={src} alt='' class='cercle-image'>
	<p class='center'> {answer3}</p>
</div>

<div> 
	<h2 class='center'style="margin-bottom:1cm; margin-top:0.5cm;"> Repartition par facultés</h2>
	<img id="planete" src="/img/cercle-19.svg" alt="scroll" class='center'style="width:75%; margin-bottom:1cm; margin-top:2cm;"/>
	<p class="center">La répartition des cours liés au climat est également très inégale en fonction des facultés.</p>
	<p class="center">Les enjeux climatiques et environnementaux sont surtout abordés dans les facultés de sciences et de sciences appliquées. (éclairage de ces cercles-là) A contrario, ils sont très peu présents dans les sciences médicales ou humaines. (éclairage de ces cercles-là)</p>
	<p class="center">En conséquence, on estime que la moitié des diplômés du supérieur ne reçoivent aucune formation sur les enjeux climatiques et environnementaux. </p>
	<p class="center">Le Pacte vert européen est pourtant clair : il est indispensable de développer des compétences liées aux enjeux climatiques dans tous les domaines d’expertise.  Ce cloisonnement entre les différentes facultés contribue donc au problème en laissant de côté des disciplines qui peuvent apporter des contributions décisives.</p>
</div>
<Scroller {threshold} bind:id={id['fleche']} splitscreen={true}>
	<div slot="background">
		<figure>
			<img id="planete" src="/img/fleche2-14.svg" alt="scroll" class='center'style="width:75%; margin-bottom:1cm; margin-top:2cm;"/>
		</figure>
		<p class="big center" style="margin-top:5cm;"> 87%</p>
	</div>

	<div slot="foreground">
		<section data-id="fleche01">
			<div class="col-medium">
				<p class='center'>L’Europe n’est pas la seule à tirer la sonnette d’alarme. Les étudiants, eux aussi, demandent à être mieux formés sur les enjeux climatiques.</p>
				<p class='center'>L’asbl The Shifters Belgium, qui promeut l’éducation sur le climat dans l’enseignement supérieur belge, a mené en 2022 une enquête auprès de 600 étudiants. 87% de ceux-ci pensent que « les universités belges doivent dispenser des modules d’enseignement sur le changement climatique ». </p>
			</div>
	</div>
</Scroller>
<h2 class='center' style="font-size = 12px;">Afin de répondre à cette demande et d’améliorer l’offre de cours traitant des enjeux climatiques dans l’enseignement supérieur belge, l’asbl émet différentes propositions :
</h2>

<div> 
	<img id="planete" src="/img/solution1-15.svg" alt="scroll" class='cercle-image center'/>
	<p class='center'> Actuellement, seulement 12% des professeurs d’université et 7% des professeurs des hautes écoles ont une expérience d’enseignement sur les enjeux climatiques et environnementaux. </p>
</div>

<div> 
	<img id="planete" src="/img/solution2-16.svg" alt="scroll" class='cercle-image center'/>
	<p class='center'> Ces derniers travailleraient en collaboration avec les professeurs et les établissements d’enseignement supérieur dans le but de transformer les programmes de cours.</p>
</div>
<div> 
	<img id="planete" src="/img/solution3-17.svg" alt="scroll" class='cercle-image center'/>
	<p class="center"> Ceux-ci permettraient d’évaluer les progrès de la transformation des programmes ainsi que d’assurer la publicité de ces formations. </p>
</div>

<p class="center"> <b>Au vu de l’urgence climatique actuelle, il est impératif que les institutions de l'enseignement supérieur qui n’ont pas ou très peu de propositions de cours sur le climat prennent leurs responsabilités et proposent des cours de qualité sur le sujet.
	Il est inquiétant de constater que les universités et hautes écoles belges sont en retard sur la question environnementale et climatique. Les étudiants doivent être formés pour devenir des citoyens engagés et conscients de l'impact de leurs choix sur l'environnement et le climat. Il est donc crucial que des modifications soient faites au niveau de l’éducation belge pour le climat afin de répondre aux besoins des étudiants et de la demande sociétale. Les futures générations en dépendent.
</b></p>
<div>
	<img id="planete" src="/img/header-18.svg" alt="scroll"/>

</div>
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
		font-size: 18px;
		text-transform: uppercase;
		margin-bottom: 0.5cm;
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
		font-size: 12px;
		font-family: "LunchType_Medium_Expanded";
		width: 80%;
		text-align: left;
		margin-top: 1cm;
		margin-bottom:1cm;

	}

	.moyen{
		color : #048D14;
		font-size: 15px;
		font-family: "Avara-black";
		width: 80%;
		text-align: left;
		text-transform: uppercase;

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
	font-size: 100px;
	font-family: "Avara-black";

}
.bonhomme {
	width: 80%;
	margin-top: 1.5cm;
	margin-bottom: 1.5cm;
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
.cercle{
	width: 75%;
	margin-top: 2.5cm;
	margin-bottom: auto;
	display :block;
	margin-left: auto;
	margin-right: auto;
	

}
.cercle-image{
	width: 75%;
	display :block;
	margin-left: auto;
	margin-right: auto;
}
</style>
