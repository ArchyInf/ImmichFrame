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

	let hasPerson = $derived(image[1].people?.filter((x) => x.name).length ?? 0 > 0);
	
	onMount(() => {
		function updateImageMetrics() {
			
			imageZoom = !debug;
			
			naturalWidth = imgEl.naturalWidth;
			naturalHeight = imgEl.naturalHeight;
			wrapperWidth = wrapperEl.clientWidth;
			wrapperHeight = wrapperEl.clientHeight;

			// 1. fill screen without letterbox
			let aspectImg = naturalWidth / naturalHeight;
			let aspectDiv = wrapperWidth / wrapperHeight;
			if(aspectImg > aspectDiv) {
				scale = wrapperHeight / naturalHeight;
			} else {
				scale = wrapperWidth / naturalWidth;
			}
			
			centerX = naturalWidth / 2;
			centerY = naturalHeight / 2;
			
			if (!hasPerson) {
				return;
			}
			
			let faceBoundsX1 = 1000000;
			let faceBoundsY1 = 1000000;
			let faceBoundsX2 = 0;
			let faceBoundsY2 = 0;

			let person = image[1].people as PersonWithFacesResponseDto[];
			for (let i = 0; i < person.length; i++) {
				person = person.filter((x) => x.name);
				let face = person[i].faces[0];
				faceBoundsX1 = Math.min(face.boundingBoxX1 ?? 0, faceBoundsX1);
				faceBoundsY1 = Math.min(face.boundingBoxY1 ?? 0, faceBoundsY1);
				faceBoundsX2 = Math.max(face.boundingBoxX2 ?? 0, faceBoundsX2);
				faceBoundsY2 = Math.max(face.boundingBoxY2 ?? 0, faceBoundsY2);
			}

			// 2. decrease scale until face rect could fit into screen
			const faceBoundsWidth = faceBoundsX2 - faceBoundsX1;
			const faceBoundsHeight = faceBoundsY2 - faceBoundsY1;
			scale *= Math.min(wrapperWidth / (scale * faceBoundsWidth), 1)
			scale *= Math.min(wrapperHeight / (scale * faceBoundsHeight), 1)
			
			// 3. move center until faces are on screen
			const visibleWidth = wrapperWidth / scale;
			const visibleHeight = wrapperHeight / scale;
			
			const visibleX1 = centerX - (visibleWidth / 2);
			const visibleX2 = centerX + (visibleWidth / 2);
			const visibleY1 = centerY - (visibleHeight / 2);
			const visibleY2 = centerY + (visibleHeight / 2);

			if (visibleX1 > faceBoundsX1) {
				centerX -= visibleX1 - faceBoundsX1;
			}
			else if (visibleX2 < faceBoundsX2) {
				centerX -= faceBoundsX2 - visibleX2;
			}

			if (visibleY1 > faceBoundsY1) {
				centerY -= visibleY1 - faceBoundsY1;
			} 
			else if (visibleY2 < faceBoundsY2) {
				centerY -= faceBoundsY2 - visibleY2;
			}
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