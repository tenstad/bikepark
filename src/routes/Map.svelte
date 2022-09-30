<script>
	import data from '../data.json';

	import MapToolbar from './MapToolbar.svelte';
	import MarkerPopup from './MarkerPopup.svelte';
	import * as markerIcons from './markers.js';

	let map;
	let container;

	let resizeMap = () => {
		if (map) {
			map.invalidateSize();
		}
	};

	import { onMount } from 'svelte';


	onMount(async () => {
		if (true) {
			// if (browser) {
			const L = await import('leaflet');

			console.log(data);

			const markerLocations = [];
			for (var i in data.features) {
				let feat = data.features[i];
				if (feat.geometry != null && feat.geometry.coordinates != null ) {
					if (feat.geometry.coordinates[0] != null && feat.geometry.coordinates[1] != null) {
						markerLocations.push([feat.geometry.coordinates[0] / 10000, feat.geometry.coordinates[1] / 100000]);
					}
				}
			}
			console.log(markerLocations);

			const initialView = [39.8283, -98.5795];
			function createMap() {
				let m = L.map(container, { preferCanvas: true }).setView(initialView, 5);
				L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png', {
					attribution: `&copy;<a href="https://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a>,
				&copy;<a href="https://carto.com/attributions" target="_blank">CARTO</a>`,
					subdomains: 'abcd',
					maxZoom: 14
				}).addTo(m);

				return m;
			}

			let eye = true;
			let lines = true;

			let toolbar = L.control({ position: 'topright' });
			let toolbarComponent;
			toolbar.onAdd = (map) => {
				let div = L.DomUtil.create('div');
				toolbarComponent = new MapToolbar({
					target: div,
					props: {}
				});

				toolbarComponent.$on('click-eye', ({ detail }) => (eye = detail));
				toolbarComponent.$on('click-lines', ({ detail }) => (lines = detail));
				toolbarComponent.$on('click-reset', () => {
					map.setView(initialView, 5, { animate: true });
				});

				return div;
			};

			toolbar.onRemove = () => {
				if (toolbarComponent) {
					toolbarComponent.$destroy();
					toolbarComponent = null;
				}
			};

			// Create a popup with a Svelte component inside it and handle removal when the popup is torn down.
			// `createFn` will be called whenever the popup is being created, and should create and return the component.
			function bindPopup(marker, createFn) {
				let popupComponent;
				marker.bindPopup(() => {
					let container = L.DomUtil.create('div');
					popupComponent = createFn(container);
					return container;
				});

				marker.on('popupclose', () => {
					if (popupComponent) {
						let old = popupComponent;
						popupComponent = null;
						// Wait to destroy until after the fadeout completes.
						setTimeout(() => {
							old.$destroy();
						}, 500);
					}
				});
			}

			let markers = new Map();

			function markerIcon(count) {
				let html = `<div class="map-marker"><div>${markerIcons.library}</div><div class="marker-text">${count}</div></div>`;
				return L.divIcon({
					html,
					className: 'map-marker'
				});
			}

			function createMarker(loc) {
				let count = Math.ceil(Math.random() * 25);
				let icon = markerIcon(count);
				let marker = L.marker(loc, { icon });
				bindPopup(marker, (m) => {
					let c = new MarkerPopup({
						target: m,
						props: {
							count
						}
					});

					c.$on('change', ({ detail }) => {
						count = detail;
						marker.setIcon(markerIcon(count));
					});

					return c;
				});

				return marker;
			}

			function createLines() {
				return L.polyline(markerLocations, { color: '#E4E', opacity: 0.5 });
			}

			let markerLayers;
			let lineLayers;

			// We could do these in the toolbar's click handler but this is an example
			// of modifying the map with reactive syntax.
			$: if (markerLayers && map) {
				if (eye) {
					markerLayers.addTo(map);
				} else {
					markerLayers.remove();
				}
			}

			$: if (lineLayers && map) {
				if (lines) {
					lineLayers.addTo(map);
				} else {
					lineLayers.remove();
				}
			}

			map = createMap();
			toolbar.addTo(map);

			markerLayers = L.layerGroup();
			for (let location of markerLocations) {
				let m = createMarker(location);
				markerLayers.addLayer(m);
			}

			lineLayers = createLines();

			markerLayers.addTo(map);
			//lineLayers.addTo(map);

			return {
				destroy: () => {
					toolbar.remove();
					map.remove();
					map = null;
				}
			};
		}
	});
</script>

<svelte:window on:resize={resizeMap} />

<link
	rel="stylesheet"
	href="https://unpkg.com/leaflet@1.6.0/dist/leaflet.css"
	integrity="sha512-xwE/Az9zrjBIphAcBb3F6JVqxf46+CDLwfLMHloNu6KEQCAWi6HcDUbeOfBIptF7tcCzusKFjFw2yuvEpDL9wQ=="
	crossorigin=""
/>
<div class="map" style="height:100vh;width:100vw" bind:this={container} />

<style>
	.map :global(.marker-text) {
		width: 100%;
		text-align: center;
		font-weight: 600;
		background-color: #444;
		color: #eee;
		border-radius: 0.5rem;
	}

	.map :global(.map-marker) {
		width: 30px;
		transform: translateX(-50%) translateY(-25%);
	}
</style>
