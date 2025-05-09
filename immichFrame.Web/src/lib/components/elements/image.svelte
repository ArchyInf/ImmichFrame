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

	function GetFaceBounds() {
		let persons = image[1].people as PersonWithFacesResponseDto[];
		persons = persons.filter((x) => x.name);

		let faceBoundsX1 = 1000000;
		let faceBoundsY1 = 1000000;
		let faceBoundsX2 = 0;
		let faceBoundsY2 = 0;
		for (let i = 0; i < persons.length; i++) {
			let face = persons[i].faces[0];
			faceBoundsX1 = Math.min(face.boundingBoxX1 ?? 0, faceBoundsX1);
			faceBoundsY1 = Math.min(face.boundingBoxY1 ?? 0, faceBoundsY1);
			faceBoundsX2 = Math.max(face.boundingBoxX2 ?? 0, faceBoundsX2);
			faceBoundsY2 = Math.max(face.boundingBoxY2 ?? 0, faceBoundsY2);
		}
		return {X1: faceBoundsX1, Y1: faceBoundsY1, X2: faceBoundsX2, Y2: faceBoundsY2};
	}

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

			const faceBounds = GetFaceBounds();
			
			// 2. decrease scale until face rect could fit into screen
			const faceBoundsWidth = faceBounds.X2 - faceBounds.X1;
			const faceBoundsHeight = faceBounds.Y2 - faceBounds.Y1;
			scale *= Math.min(wrapperWidth / (scale * faceBoundsWidth), 1)
			scale *= Math.min(wrapperHeight / (scale * faceBoundsHeight), 1)
			
			// 3. move center until faces are on screen
			const visibleWidth = wrapperWidth / scale;
			const visibleHeight = wrapperHeight / scale;
			
			const visibleX1 = centerX - (visibleWidth / 2);
			const visibleX2 = centerX + (visibleWidth / 2);
			const visibleY1 = centerY - (visibleHeight / 2);
			const visibleY2 = centerY + (visibleHeight / 2);

			if (visibleX1 > faceBounds.X1) {
				centerX -= visibleX1 - faceBounds.X1;
			}
			else if (visibleX2 < faceBounds.X2) {
				centerX += faceBounds.X2 - visibleX2;
			}

			if (visibleY1 > faceBounds.Y1) {
				centerY -= visibleY1 - faceBounds.Y1;
			} 
			else if (visibleY2 < faceBounds.Y2) {
				centerY += faceBounds.Y2 - visibleY2;
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

<div bind:this={wrapperEl} class="immichframe_image" style="overflow: hidden;">
	{#if debug}
		<div
			class="face z-[900] bg-red-600 absolute"
			style="top: {(-centerY*scale + wrapperHeight/2) + GetFaceBounds().Y1*scale}px;
			left: {(-centerX*scale + wrapperWidth/2) + GetFaceBounds().X1*scale}px;
			width: {(GetFaceBounds().X2-GetFaceBounds().X1)*scale}px;
			height: {(GetFaceBounds().Y2-GetFaceBounds().Y1)*scale}px;"
		></div>
	{/if}
	
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