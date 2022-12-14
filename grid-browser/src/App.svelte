<script>
	import DateFilter from "./DateFilter.svelte";
	import Loader from "./Loader.svelte";
	import { beforeUpdate } from "svelte";
	import Modal from "./Modal.svelte";
	import Block from "./Block.svelte";
	import Collection from "./Collection.svelte";

	let isModalOpen = false;
	let currentCanvas;

	let allItems = [];
	let activeItems = [];
	function loadFromManifest(data) {
		let title;
		let source;
		let date;
		let decade;
		data.metadata.forEach((item) => {
			if (item.label.sv[0] == "Titel") {
				title = item.value.none[0];
			} else if (item.label.sv[0] == "Källhänvisning") {
				source = item.value.none[0];
			} else if (item.label.sv[0] == "Datering") {
				date = item.value.none[0];

				// we allways try to get the decade as to avoid computing it on the fly during filtering
				if (date.length == 4 && !isNaN(date)) { // parseInt is too forgiving on its own
					decade = Math.floor(parseInt(date)/10)*10;
				// parse year ranges and of the form 1700-1709 and if the two years is in the same decade add oy
				} else if (date.length == 9 && date[4] == "-" && !isNaN(date.substring(0,4)) && !isNaN(date.substring(5,9))) {
					if (Math.floor(parseInt(date.substring(0,4))/10)*10 === Math.floor(parseInt(date.substring(5,9))/10)*10) {
						decade = Math.floor(parseInt(date.substring(0,4))/10)*10;
					}
				} else {
					decade = null;
				}
			}
		});

		data.items.forEach((item) => {
			if (item.type == "Canvas") {
				let image = item.id.replace(
					"/canvas",
					"/square/350,/0/default.jpg"
				);
				let link = item.id
					.replace("/canvas", "")
					.replace(
						"https://lbiiif.riksarkivet.se/arkis!",
						"https://sok.riksarkivet.se/bildvisning/"
					);
					allItems.push({
					title: title,
					source: source,
					date: date,
					image: image,
					link: link,
					decade: decade,
				});
				allItems = allItems; // for the Svelte compiler https://svelte.dev/tutorial/updating-arrays-and-objects
				activeItems = allItems; // we set all items as active to trigger beforeUpdate which will preform filtering
			}
		});
	}

	let foundURIs = [];
	let title;
	function traverse(url) {
		fetch(url)
			.then((response) => {
				return response.json();
			})
			.then((data) => {
				if (!foundURIs.length) {
					title = data.label.sv[0];
					document.title = title;
				}

				if (data.type == "Manifest") {
					loadFromManifest(data);
				}

				foundURIs.push(url);

				for (let item of data.items) {
					if (!foundURIs.includes(item.id)) {
						if (item.type == "Manifest") {
							traverse(item.id);
						} else if (item.type == "Collection") {
							traverse(item.id);
						}
					}
				}
			})
	}
	let manifest = new URLSearchParams(window.location.search).get("manifest");
	
	if (manifest) {
		manifest.split(",").forEach(m => traverse(m));
		
	} else {
		title = 'Riksarkivet';
	}

	let collection = new URLSearchParams(window.location.search).get("collection");
	collection = collection ? collection : "https://lbiiif.riksarkivet.se/collection/riksarkivet";

	let inputManifest;
	function loadFromManifestInput() {
		inputManifest.split(",").forEach(m => traverse(m));
		window.history.pushState(
			{},
			"",
			`?manifest=${inputManifest}`
		);
	}

	let date = new URLSearchParams(window.location.search).get("date");

	let activeDecade = 0;
	let decades;
	let previousDecade;
	let nextDecade;
	beforeUpdate(() => {
		if (date) {
			// we use the median existing decade as a starting point
			decades = [...new Set(allItems.map(item => item.decade))].sort();
			const medianDecade = decades[Math.floor(decades.length / 2)];
			if (!userHasInteracted) activeDecade = medianDecade;

			filterDate();
		} else {
			activeItems = allItems;
		}
	});

	function filterDate() {
		activeItems = allItems.filter(item => item.decade === activeDecade);
		previousDecade = decades[decades.indexOf(activeDecade) - 1];
		nextDecade = decades[decades.indexOf(activeDecade) + 1];
	}

	let userHasInteracted = false;
	function dateFilterChanged(event) {
		const navigation = event.detail.action;
		userHasInteracted = true;
		if (navigation == 'previous') {
			activeDecade = previousDecade;
		} else {
			activeDecade = nextDecade;
		}
		filterDate();
	}

	function openModal(newCanvas) {
		currentCanvas = newCanvas;
		isModalOpen = true;
	}

	function closeModal() {
		isModalOpen = false;
	}
</script>

<nav>
	<h1>{title}</h1>
	{#if date}
		<DateFilter decade={activeDecade} canNavigateBack={previousDecade} canNavigateForward={nextDecade} on:navigation="{dateFilterChanged}"/>
	{/if}
</nav>
<main class="dark">
	{#if activeItems.length > 0 }
		<div class="container-background dark">
			<div class="container">
				{#each activeItems as item}
					<button title={item.title} on:click={openModal(item.image)}>
						<img src={item.image} alt="{item.title}" />
					</button>
				{/each}
			</div>
		</div>
	{:else if manifest}
		<Loader />
	{:else}
	
	<div class="browser-container">
		{#if collection == "https://lbiiif.riksarkivet.se/collection/riksarkivet"}
			<div class="responsive-container">
				<h2 style="margin-top: 0;padding-top: 17px;">Urval</h2>
				<div class="columns">
					<Block title="Handritade Kartverk" thumbnail="https://lbiiif.riksarkivet.se/arkis!K0037920_00001/square/350,/0/default.jpg" manifest="https://lbiiif.riksarkivet.se/collection/arkiv/AwBKLPUAqqUGtNLGHo3lq0"/>
					<Block title="Marinens ritningar" thumbnail="https://lbiiif.riksarkivet.se/arkis!K0035322_00001/square/350,/0/default.jpg" manifest="https://lbiiif.riksarkivet.se/collection/arkiv/lAELykjdz1EZN9cxABCer7" timeline="true"/>
					<Block title="001 Jacob Gillberg: svenska och finska uniformer" thumbnail="https://lbiiif.riksarkivet.se/arkis!K0038337_00001/square/350,/0/default.jpg" manifest="https://lbiiif.riksarkivet.se/collection/arkiv/8XCsKmH8XKATnPaXVPaWf2"/>
					<Block title="K Fotografier" thumbnail="https://lbiiif.riksarkivet.se/arkis!Z0000195_00001/square/350,/0/default.jpg" manifest="https://lbiiif.riksarkivet.se/collection/arkiv/vdLhUFT8rH6cxG02H087k3"/>
				</div>
			</div>
		{/if}


		<div class="responsive-container">
			<h2 style="margin-top: 0;padding-top: 17px;">Utforska</h2>
			<Collection collection={collection}/>
		</div>

		<div class="responsive-container">
			<details>
				<summary>Eget IIIF Manifest</summary>
				<form on:submit|preventDefault="{loadFromManifestInput}">
					<label for="iiif-input">IIIF Manifest</label><br>
					<input id="iiif-input" type="url" placeholder="https://lbiiif.riksarkivet.se/collection/arkiv/8XCsKmH8XKATnPaXVPaWf2" bind:value="{inputManifest}" /><br>
					<button type="submit">Ladda</button>
				</form>
			</details>
		</div>
	</div>
	
	{/if}
	{#if isModalOpen}
		<Modal currentCanvas={currentCanvas} isModalOpen={isModalOpen} on:modal-close="{closeModal}" />
	{/if}
</main>

<style>
nav {
	padding: 0 10px;
	border-bottom: 6px solid var(--ra-blue-main);
}

details {
	margin: 0;
	border-radius: 4px;
	padding: 2em 0;
	clear: both;
}

summary {
	font-weight: bold;
	margin: -.5em -.5em 0;
	padding: .5em;
}

.browser-container {
	padding: 0em 1em;
}

.container {
	display: grid;
	grid-template-columns: repeat(8, 1fr);
	grid-gap: 1em;
}

.container-background {
	padding: 1em;
	padding-bottom: 0;
}

.container button {
	overflow: hidden;
	aspect-ratio: 1;
	background: none;
	border: none;
	cursor: pointer;
}

.container button > img {
	width: 100%;
}

/* grid template break points */
@media (max-width: 1200px) {
	.container {
		grid-template-columns: repeat(6, 1fr);
	}
}

@media (max-width: 900px) {
	.container {
		grid-template-columns: repeat(4, 1fr);
	}
}

@media (max-width: 600px) {
	.container {
		grid-template-columns: repeat(2, 1fr);
	}
}

@media (max-width: 400px) {
	.container {
		grid-template-columns: repeat(1, 1fr);
	}
}
</style>
