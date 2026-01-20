<script lang="ts">
	import { Panel } from '$lib/components/common';
	import {
		CABLE_LANDINGS,
		CHOKEPOINTS,
		CONFLICT_ZONES,
		HOTSPOTS,
		MILITARY_BASES,
		NUCLEAR_SITES,
		THREAT_COLORS
	} from '$lib/config/map';
	import type { CustomMonitor } from '$lib/types';
	import { onMount } from 'svelte';

interface GlobeControls {
	autoRotate: boolean;
	autoRotateSpeed: number;
}

interface GlobeInstance {
	pointOfView(view: { lat: number; lng: number; altitude: number }, ms?: number): void;
	width(width: number): void;
	height(height: number): void;
	pointsData(data: unknown[]): GlobeInstance;
	labelsData(data: unknown[]): GlobeInstance;
	polygonsData(data: unknown[]): GlobeInstance;
	controls(): GlobeControls;
	_destructor(): void;
}

interface Props {
		monitors?: CustomMonitor[];
		loading?: boolean;
		error?: string | null;
	}

	let { monitors = [], loading = false, error = null }: Props = $props();

let container: HTMLElement;
let myGlobe: GlobeInstance | null = null;

// --- CONFIGURATION ---
// Altitude > 1.0 zooms out. 4.5 fits the globe in a 400px height.
const INITIAL_VIEW = { lat: 30, lng: 50, altitude: 0.9 };

// 1. Prepare Data Points
const pointsData = $derived([
	...HOTSPOTS.map((h) => ({
		lat: h.lat,
		lng: h.lon,
		size: h.level === 'critical' ? 1.5 : 0.8,
		color: THREAT_COLORS[h.level],
		name: h.name,
		desc: `Hotspot: ${h.desc}`,
		type: 'hotspot'
	})),
	...NUCLEAR_SITES.map((n) => ({
		lat: n.lat,
		lng: n.lon,
		size: 0.6,
		color: '#ffff00',
		name: n.name,
		desc: `Nuclear: ${n.desc}`,
		type: 'nuclear'
	})),
	...MILITARY_BASES.map((m) => ({
		lat: m.lat,
		lng: m.lon,
		size: 0.6,
		color: '#ff00ff',
		name: m.name,
		desc: `Military: ${m.desc}`,
		type: 'military'
	})),
	...CHOKEPOINTS.map((c) => ({
		lat: c.lat,
		lng: c.lon,
		size: 0.6,
		color: '#00aaff',
		name: c.name,
		desc: `Chokepoint: ${c.desc}`,
		type: 'chokepoint'
	})),
	...CABLE_LANDINGS.map((c) => ({
		lat: c.lat,
		lng: c.lon,
		size: 0.4,
		color: '#aa44ff',
		name: c.name,
		desc: `Cable: ${c.desc}`,
		type: 'cable'
	})),
	...monitors
		.filter((m) => m.enabled && m.location)
		.map((m) => ({
			lat: m.location!.lat,
			lng: m.location!.lon,
			size: 1.2,
			color: m.color || '#00ffff',
			name: m.name,
			desc: `Monitor: ${m.name}`,
			type: 'monitor'
		}))
]);

const zonesData = $derived(
		CONFLICT_ZONES.map((zone) => ({
			type: 'Feature',
			geometry: { type: 'Polygon', coordinates: [zone.coords] },
			properties: { name: zone.name, color: zone.color }
		}))
	);

	function resetView() {
		if (myGlobe) {
			myGlobe.pointOfView(INITIAL_VIEW, 1500); // Animate to home
		}
	}

	onMount(() => {
		let resizeObserver: ResizeObserver;

		(async () => {
			if (typeof window === 'undefined') return;

			const module = await import('globe.gl');
			const Globe = module.default;

			const globeInstance = new Globe(container)
				.globeImageUrl('//unpkg.com/three-globe/example/img/earth-blue-marble.jpg')
				.bumpImageUrl('//unpkg.com/three-globe/example/img/earth-topology.png')
				.backgroundColor('rgba(0,0,0,0)') // Transparent
				.atmosphereColor('lightskyblue')
				.atmosphereAltitude(0.15)
				
				// --- Points ---
				.pointsData(pointsData)
				.pointAltitude(0.02)
				.pointColor('color')
				.pointRadius('size')
				.pointLabel('desc')

				// --- Text Labels ---
				.labelsData(pointsData)
				.labelLat('lat')
				.labelLng('lng')
				.labelText('name')
				.labelSize((d: any) => d.type === 'hotspot' ? 1.5 : 1.0)
				.labelDotRadius(0.3)
				.labelColor(() => '#ffffff')
				.labelResolution(2)
				.labelAltitude(0.05) // Lift text higher to avoid clipping

				// --- Polygons ---
				.polygonsData(zonesData)
				.polygonGeoJsonGeometry((d: any) => d.geometry)
				.polygonCapColor((d: any) => d.properties.color)
				.polygonSideColor(() => 'rgba(0, 100, 0, 0.05)')
				.polygonStrokeColor(() => '#111')
				.polygonAltitude(0.01)
				.polygonLabel((d: any) => `<b>${d.properties.name}</b>`);

			const controls = globeInstance.controls();
			controls.autoRotate = true;
			// Auto rotate speed in degrees per second
			controls.autoRotateSpeed = 0.5;

			myGlobe = globeInstance;

			// Apply the zoomed-out view immediately
			myGlobe.pointOfView(INITIAL_VIEW);

			if (container && myGlobe) {
				myGlobe.width(container.clientWidth);
				myGlobe.height(container.clientHeight);
			}

			resizeObserver = new ResizeObserver((entries) => {
				const { width, height } = entries[0].contentRect;
				if (myGlobe) {
					myGlobe.width(width);
					myGlobe.height(height);
				}
			});
			resizeObserver.observe(container);
		})();

		return () => {
			if (resizeObserver) resizeObserver.disconnect();
			if (myGlobe) myGlobe._destructor();
		};
	});

	$effect(() => {
		if (myGlobe && pointsData) {
			myGlobe.pointsData(pointsData);
			myGlobe.labelsData(pointsData);
		}
		if (myGlobe && zonesData) {
			myGlobe.polygonsData(zonesData);
		}
	});
</script>

<Panel id="map" title="Global Situation (3D)" {loading} {error}>
	<div class="globe-wrapper">
		<div class="globe-container" bind:this={container}></div>
		
		<div class="globe-controls">
			<button class="control-btn" onclick={resetView}>⟲ Reset View</button>
		</div>

		<div class="globe-legend">
			<div class="legend-item"><span class="dot critical"></span> Critical</div>
			<div class="legend-item"><span class="dot conflict"></span> Conflict Zone</div>
			<div class="legend-item"><span class="dot nuclear"></span> Nuclear</div>
			<div class="legend-item"><span class="dot military"></span> Military</div>
			<div class="legend-item"><span class="dot chokepoint"></span> Chokepoint</div>
			<div class="legend-item"><span class="dot cable"></span> Submarine Cable</div>
			<div class="legend-item"><span class="dot monitor"></span> My Monitor</div>
		</div>
	</div>
</Panel>

<style>
	.globe-wrapper {
		position: relative;
		width: 100%;
		height: 400px;
		/* background: #000; */
		background: var(--bg); /* Was #000 */
		border-radius: 4px;
		overflow: hidden;
	}

	.globe-container {
		width: 100%;
		height: 100%;
	}

	.globe-legend {
		position: absolute;
		bottom: 10px;
		right: 10px;
		background: rgba(0, 0, 0, 0.7);
		padding: 8px;
		border-radius: 4px;
		font-size: 0.7rem;
		color: #ccc;
		pointer-events: none;
		/* display: grid; */
		grid-template-columns: 1fr 1fr; /* 2 Columns to save vertical space */
		gap: 0.5rem 1rem;
	}

	.globe-controls {
		position: absolute;
		bottom: 10px;
		left: 10px;
		z-index: 10;
	}

	.control-btn {
		background: rgba(30, 30, 30, 0.8);
		color: #fff;
		border: 1px solid #444;
		border-radius: 4px;
		padding: 6px 12px;
		font-size: 0.75rem;
		cursor: pointer;
		transition: background 0.2s;
	}

	.control-btn:hover {
		background: rgba(60, 60, 60, 0.9);
	}

	.legend-item {
		display: flex;
		align-items: center;
		gap: 6px;
		margin-bottom: 2px;
	}

	.dot {
		width: 8px;
		height: 8px;
		border-radius: 50%;
		display: inline-block;
	}

	/* Color Definitions matching the JS Config */
	.dot.critical { background: #ff0000; }
	.dot.conflict { background: #ff4444; opacity: 0.6; }
	.dot.nuclear { background: #ffff00; }
	.dot.military { background: #ff00ff; }
	.dot.chokepoint { background: #00aaff; }
	.dot.cable { background: #aa44ff; }
	.dot.monitor { background: #00ffff; }

</style>
