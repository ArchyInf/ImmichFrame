<script lang="ts">
	import { type AssetResponseDto, type PersonWithFacesResponseDto } from '$lib/immichFrameApi';
	import { decodeBase64 } from '$lib/utils';
	import { thumbHashToDataURL } from 'thumbhash';
	import AssetInfo from './asset-info.svelte';
	import { onMount } from 'svelte';

	interface Props {
		image: [url: string, asset: AssetResponseDto];
		showLocation: boolean;
		showPhotoDate: boolean;
		showImageDesc: boolean;
		showPeopleDesc: boolean;
		imageFill: boolean;
		imageZoom: boolean;
		interval: number;
		multi?: boolean;
	}

	let {
		image,
		showLocation,
		showPhotoDate,
		showImageDesc,
		showPeopleDesc,
		imageFill,
		imageZoom,
		interval,
		multi = false
	}: Props = $props();

	let debug = true;
	let imgEl: HTMLImageElement;
	let wrapperEl: HTMLDivElement;
	
	let renderedWidth = 0;
	let renderedHeight = 0;
	let offsetX = 0;
	let offsetY = 0;
	let scaleX = 1;
	let scaleY = 1;
	let wrapperWidth = 100;
	let wrapperHeight = 100;
	
	let positionX = $state(0);
	let positionY = $state(0);
	let naturalWidth = $state(0);
	let naturalHeight = $state(0);

	let hasPerson = $derived(image[1].people?.filter((x) => x.name).length ?? 0 > 0);

	onMount(() => {
		function updateImageMetrics() {
			let faceBoundsMinX = 1000000;
			let faceBoundsMinY = 1000000;
			let faceBoundsMaxX = 0;
			let faceBoundsMaxY = 0;

			let person = image[1].people as PersonWithFacesResponseDto[];
			for (let i = 0; i < person.length; i++) {
				person = person.filter((x) => x.name);
				let face = person[i].faces[0];
				faceBoundsMinX = Math.min(face.boundingBoxX1 ?? 0, faceBoundsMinX);
				faceBoundsMinY = Math.min(face.boundingBoxY1 ?? 0, faceBoundsMinY);
				faceBoundsMaxX = Math.max(face.boundingBoxX2 ?? 0, faceBoundsMaxX);
				faceBoundsMaxY = Math.max(face.boundingBoxY2 ?? 0, faceBoundsMaxY);
			}

			positionX = -((faceBoundsMinX + faceBoundsMaxX) / 2);
			positionY = -((faceBoundsMinY + faceBoundsMaxY) / 2);
			
			imageZoom = !debug;

			if (!imgEl || !imgEl.complete || !wrapperEl) return;
			
			renderedWidth = imgEl.clientWidth;
			renderedHeight = imgEl.clientHeight;

			naturalWidth = imgEl.naturalWidth;
			naturalHeight = imgEl.naturalHeight;
		
			// size of render area
			wrapperWidth = wrapperEl.clientWidth;
			wrapperHeight = wrapperEl.clientHeight;

			scaleX = renderedWidth / naturalWidth;
			scaleY = renderedHeight / naturalHeight;

			offsetX = (wrapperWidth - renderedWidth) / 2;
			offsetY = (wrapperHeight - renderedHeight) / 2;

			if (debug) console.log(`X ${faceBoundsMinX}`);
			if (debug) console.log(`Y ${faceBoundsMinY}`);
			if (debug) console.log(`W ${faceBoundsMaxX}`);
			if (debug) console.log(`H ${faceBoundsMaxY}`);
			if (debug) console.log(`Pos (${positionX},${positionY})`);
			if (debug) console.log(`natural (${naturalWidth},${naturalHeight})`);
			// if (debug) console.log(`X ${imgEl.getBoundingClientRect().x}`);
			// if (debug) console.log(`Y ${imgEl.getBoundingClientRect().y}`);
			// if (debug) console.log(`W ${imgEl.getBoundingClientRect().width}`);
			// if (debug) console.log(`H ${imgEl.getBoundingClientRect().height}`);
		}

		updateImageMetrics();
		imgEl.addEventListener('load', updateImageMetrics);
		window.addEventListener('resize', updateImageMetrics);

		return () => {
			imgEl.removeEventListener('load', updateImageMetrics);
			window.removeEventListener('resize', updateImageMetrics);
		};
	});
</script>

<div bind:this={wrapperEl} style="overflow: hidden;">
	<img
		bind:this={imgEl}
		style="
			max-width: none;
			width: {naturalWidth/2}px; height: {naturalHeight/2}px;
			transform: translate({positionX/2 + wrapperWidth/2}px, {positionY/2 + wrapperHeight/2}px);
		"
		src={image[0]}
		alt="data"
	/>
</div>
<AssetInfo asset={image[1]} {showLocation} {showPhotoDate} {showImageDesc} {showPeopleDesc} />

<style>
</style>