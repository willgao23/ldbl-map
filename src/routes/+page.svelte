<script lang="ts">
	import CA from '../lib/data/canadaprov.json';
	import legStatusJson from '../lib/data/legislationStatus.json';
	import type { FeatureCollection, Feature, Geometry, GeoJsonProperties } from 'geojson';
	import { geoPath, geoMercator } from 'd3-geo';
	import { select, selectAll, pointer } from 'd3-selection';
	import 'd3-transition';
	import { zoom, zoomIdentity } from 'd3-zoom';
	import type { ZoomBehavior, ZoomTransform } from 'd3-zoom';
	import { scaleOrdinal } from 'd3-scale';

	const options = [
		'Aquamation',
		'Ash Dispersion',
		'Ash Conversion',
		'Contained Ash Burial',
		'Natural Ground Burial',
		'Conservation Burial / Land Legacy',
		'Home Burial',
		'Mushroom Burial Suit',
		'Body Composting',
		'Donating Body to Science',
		'Conventional Burial',
		'Cremation'
	];
	let selectedOption = options[0];
	let chosenProvince: string | null = null;
	let infoTitle: string | null = null;
	const legStatus: Record<string, Record<string, string>> = legStatusJson;
	const keys = ['Legal', 'Partially Legal', 'Illegal'];
	const colorScale = scaleOrdinal(keys, ['#6eb1ac', '#abd9d5', '#edf7f7']);
	let width: number;
	$: height = width * 0.4;
	let margin = { top: 100, left: 0, right: 0, bottom: 100 };
	let projection, path;
	let zoomFunction: ZoomBehavior<Element, unknown>;
	let initialTransform: ZoomTransform;
	let gElement: SVGElement;
	let svgElement: SVGElement;
	let legendElement: SVGElement;

	$: zoomFunction = zoom()
		.scaleExtent([1, 10])
		.translateExtent([
			[0, -margin.bottom],
			[width, height + margin.top]
		])
		.on('zoom', (event) => {
			const { transform } = event;
			select(gElement)
				.attr('transform', transform)
				.attr('stroke-width', 1 / transform.k);
		});

	$: initialTransform = zoomIdentity
		.translate(width / 2, height / 2)
		.scale(2)
		.translate(-width / 2, -(2 * height) / 3);

	$: select(svgElement)
		.transition()
		.duration(2000)
		.call(zoomFunction.transform as any, initialTransform);

	const provinceFeatures: Feature<Geometry, GeoJsonProperties>[] = CA.features.map((f: any) => ({
		type: 'Feature',
		properties: f.properties,
		geometry: f.geometry
	}));

	const provinceCollection: FeatureCollection = {
		type: 'FeatureCollection',
		features: provinceFeatures
	};

	$: projection = geoMercator().fitExtent(
		[
			[margin.left, margin.top],
			[width - margin.bottom, height - margin.right]
		],
		provinceCollection
	);

	$: path = geoPath().projection(projection);

	$: if (gElement && path && svgElement) {
		select(gElement)
			.selectAll('path')
			.data(provinceFeatures)
			.join('path')
			.attr('d', path as any)
			.attr('fill', (d) => colorScale(legStatus[selectedOption][d.properties?.postal]))
			.attr('stroke', 'black')
			.attr('stroke-width', 0.5)
			.on('mouseover', function (event, d) {
				select(this).transition().duration(100).attr('stroke-width', 1.5);
			})
			.on('mouseout', function (event, d) {
				select(this).transition().duration(100).attr('stroke-width', 0.5);
			})
			.on('click', function (event, d) {
				console.log(d);
				const [[x0, y0], [x1, y1]] = path.bounds(d);
				let scaleFactor;
				const postalAbbr = d.properties?.postal;
				const increaseScaleGroup = new Set(['NU', 'PE', 'NB', 'NS']);
				if (postalAbbr == 'NT') {
					scaleFactor = 1.3;
				} else if (increaseScaleGroup.has(postalAbbr)) {
					scaleFactor = 1.8;
				} else {
					scaleFactor = 0.9;
				}
				event.stopPropagation();
				select(svgElement)
					.transition()
					.duration(2000)
					.call(
						zoomFunction.transform as any,
						zoomIdentity
							.translate(width / 2, height / 2)
							.scale(Math.min(10, scaleFactor / Math.max((x1 - x0) / width, (y1 - y0) / height)))
							.translate(-(x0 + x1) / 2, -(y0 + y1) / 2),
						pointer(event, select(svgElement).node())
					)
					.on('end', () => {
						chosenProvince = postalAbbr;
						infoTitle = `${selectedOption} in ${d.properties?.name}`;
					});
			});
	}

	$: if (legendElement && svgElement && width) {
		const size = 40;
		const textPad = 10;
		const backgroundPad = 20;
		const svgNode = select(svgElement).node();
		let legendX = 0;
		if (svgNode) {
			const rect = svgNode.getBoundingClientRect();
			legendX = rect.width - rect.width / 3;
		}

		select(legendElement).selectAll('*').remove();

		const legendHeight = keys.length * (size + 5) + backgroundPad;
		const legendWidth = size + textPad + 120 + backgroundPad;

		select(legendElement)
			.append('rect')
			.attr('class', 'legendBackground')
			.attr('x', legendX - backgroundPad / 2)
			.attr('y', 100 - backgroundPad / 2)
			.attr('width', legendWidth)
			.attr('height', legendHeight)
			.attr('rx', 5)
			.attr('ry', 5)
			.style('fill', '#f8f8f8')
			.style('fill-opacity', 0.8)
			.style('stroke', '#ccc')
			.style('stroke-width', 1);

		select(legendElement)
			.selectAll('.legendDots')
			.data(keys)
			.join('rect')
			.attr('x', legendX)
			.attr('y', function (d, i) {
				return 100 + i * (size + 5);
			})
			.attr('width', size)
			.attr('height', size)
			.attr('stroke', 'black')
			.attr('stroke-width', 1)
			.style('fill', function (d) {
				return colorScale(d);
			});

		select(legendElement)
			.selectAll('.mylabels')
			.data(keys)
			.join('text')
			.attr('x', legendX + size + textPad)
			.attr('y', function (d, i) {
				return 100 + i * (size + 5) + size / 2;
			})
			.style('font-size', '20px')
			.style('fill', 'black')
			.text(function (d) {
				return d;
			})
			.attr('text-anchor', 'left')
			.style('alignment-baseline', 'middle');
	}

	function handleClick(event: MouseEvent) {
		chosenProvince = null;
		select(svgElement)
			.transition()
			.duration(2000)
			.call((zoomFunction as any).transform, initialTransform);
	}
</script>

<div class="map-container" bind:clientWidth={width}>
	{#if width}
		{#if chosenProvince}
			<div class="infoDiv" style="width: {width / 4}px;">
				<div class="infoTitle">{infoTitle}</div>
				<div class="infoText">
					Lorem ipsum dolor sit amet, consectetur adipiscing elit. Mauris tincidunt, orci ut lacinia
					tincidunt, lacus ante pulvinar felis, id consectetur neque dolor ac massa. Integer
					vulputate, elit mattis dictum elementum, orci mauris lobortis quam, in vestibulum ex lorem
					ac risus. Mauris volutpat, arcu ac consequat porttitor, lectus neque faucibus urna, at
					viverra nunc lacus at erat. Quisque eu nunc eu tortor elementum consectetur. Phasellus
					vehicula at sem quis dapibus.
				</div>
				<button class="returnBtn" onclick={handleClick}>Return to full map</button>
			</div>
		{/if}
		<div class="dropdown-container" style="left: {width - width / 3 - 75}px;">
			<select bind:value={selectedOption} id="policy">
				{#each options as opt}
					<option value={opt}>{opt}</option>
				{/each}
			</select>
		</div>

		<svg
			bind:this={svgElement}
			id="mapSvg"
			width={width + margin.left + margin.right}
			height={height + margin.top + margin.bottom}
		>
			<g bind:this={gElement} />
			<g bind:this={legendElement} class="legend" />
		</svg>
	{/if}
</div>

<style>
	.infoDiv {
		position: absolute;
		top: 60%;
		left: 50%;
		transform: translate(-50%, -50%);
		background: rgba(255, 255, 255, 0.95);
		padding: 20px;
		border-radius: 10px;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
		z-index: 20;
		text-align: center;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
	}

	.infoTitle {
		font-size: 24px;
		font-weight: bold;
	}

	.infoText {
		font-size: 16px;
		margin: 16px;
	}

	.returnBtn {
		background-color: #abd9d5;
		font-family: 'Times New Roman', Times, serif;
		border-radius: 10px;
		border: none;
		color: black;
		padding: 10px;
		text-align: center;
		text-decoration: none;
		display: inline-block;
		font-size: 16px;
		cursor: pointer;
		width: 200px;
	}

	.returnBtn:hover {
		background-color: #6eb1ac;
	}

	.map-container {
		position: relative;
		display: inline-block;
		width: 100%;
	}

	.dropdown-container {
		position: absolute;
		top: 260px;
		background: rgba(255, 255, 255, 0.8);
		padding: 6px 10px;
		border-radius: 6px;
		box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
		z-index: 10;
	}

	select {
		font-size: 20px;
		font-family: 'Times New Roman', Times, serif;
		padding: 4px 8px;
	}
</style>
