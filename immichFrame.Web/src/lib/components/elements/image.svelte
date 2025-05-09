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
	
	let wrapperWidth = $state(100);
	let wrapperHeight = $state(100);
	let centerX = $state(0);
	let centerY = $state(0);
	let naturalWidth = $state(0);
	let naturalHeight = $state(0);
	let scale = $state(1.0);

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

			imageZoom = !debug;

			naturalWidth = imgEl.naturalWidth;
			naturalHeight = imgEl.naturalHeight;
		
			// size of render area
			wrapperWidth = wrapperEl.clientWidth;
			wrapperHeight = wrapperEl.clientHeight;
			
			let aspectImg = naturalWidth / naturalHeight;
			let aspectDiv = wrapperWidth / wrapperHeight;

			centerX = ((faceBoundsMinX + faceBoundsMaxX) / 2);
			centerY = ((faceBoundsMinY + faceBoundsMaxY) / 2);
			
			if(aspectImg > aspectDiv) {
				centerY = naturalHeight / 2;
				scale = wrapperHeight / naturalHeight;
			} else {
				centerX = naturalWidth / 2;
				scale = wrapperWidth / naturalWidth;
			}

			if (debug) console.log(`X ${faceBoundsMinX}`);
			if (debug) console.log(`Y ${faceBoundsMinY}`);
			if (debug) console.log(`W ${faceBoundsMaxX}`);
			if (debug) console.log(`H ${faceBoundsMaxY}`);
			if (debug) console.log(`Pos (${centerX},${centerY})`);
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
			width: {naturalWidth*scale}px; height: {naturalHeight*scale}px;
			transform: translate({-centerX*scale + wrapperWidth/2}px, {-centerY*scale + wrapperHeight/2}px);
		"
		src={image[0]}
		alt="data"
	/>
</div>
<AssetInfo asset={image[1]} {showLocation} {showPhotoDate} {showImageDesc} {showPeopleDesc} />

<style>
</style>