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
	
	let positionX = 0;
	let positionY = 0;

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

			positionX = 0.5 * (faceBoundsMinX + faceBoundsMaxX);
			positionY = 0.5 * (faceBoundsMinY + faceBoundsMaxY);
			// positionX = 0;
			// positionX = 0;
			
			imageZoom = !debug;

			if (!imgEl || !imgEl.complete || !wrapperEl) return;
			
			renderedWidth = imgEl.clientWidth;
			renderedHeight = imgEl.clientHeight;

			const naturalWidth = imgEl.naturalWidth;
			const naturalHeight = imgEl.naturalHeight;
		
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

	function GetFace(i: number) {
		let person = image[1].people as PersonWithFacesResponseDto[];
		person = person.filter((x) => x.name);
		return person[i].faces[0];
	}

	function GetPosX(i: number) {
		const face = GetFace(i);
		if (!face) return 0;
		return ((face.boundingBoxX1 ?? 0) * scaleX + offsetX) / wrapperWidth * 100;
	}

	function GetPosY(i: number) {
		return 10;
		const face = GetFace(i);
		if (!face) return 0;
		return ((face.boundingBoxY1 ?? 0) * scaleY + offsetY) / wrapperHeight * 100;
	}

	function getCenterX(i: number) {
		const face = GetFace(i);
		if (!face) return 0;
		const midX = ((face.boundingBoxX2 ?? 0) + (face.boundingBoxX1 ?? 0)) / 2;
		return ((midX * scaleX + offsetX) / wrapperWidth) * 100;
	}

	function getCenterY(i: number) {
		const face = GetFace(i);
		if (!face) return 0;
		const midY = ((face.boundingBoxY2 ?? 0) + (face.boundingBoxY1 ?? 0)) / 2;
		return ((midY * scaleY + offsetY) / wrapperHeight) * 100;
	}

	function getWidth(i: number) {
		return 10;
		const face = GetFace(i);
		if (!face) return 0;
		return (((face.boundingBoxX2 ?? 0) - (face.boundingBoxX1 ?? 0)) * scaleX / wrapperWidth) * 100;
	}

	function getHeight(i: number) {
		return 10;
		const face = GetFace(i);
		if (!face) return 0;
		return (((face.boundingBoxY2 ?? 0) - (face.boundingBoxY1 ?? 0)) * scaleY / wrapperHeight) * 100;
	}

	function zoomEffect() {
		return 0.5 > Math.random();
	}
</script>

<div bind:this={wrapperEl} class="immichframe_image place-self-center overflow-hidden">
	{#if debug}
		{#each image[1].people?.map((x) => x.name) ?? [] as _, i}
			<div
				class="face z-[900] bg-red-600 absolute"
				style="top: {GetPosY(i)}%;
					left: {GetPosX(i)}%;
					width: {getWidth(i)}%;
					height: {getHeight(i)}%;"
			></div>
			<div
				class="centerface z-[999] w-1 h-1 bg-blue-600 absolute"
				style="top: {getCenterY(i)}%;
					left: {getCenterX(i)}%;"
			></div>
		{/each}
	{/if}

	<img
		bind:this={imgEl}
		style="--interval: {interval + 2}s; --posX: {positionX}%; --posY: {positionY}%;"
		class="{multi || imageZoom
			? 'w-screen h-dvh object-cover'
			: 'max-h-screen h-dvh max-w-full object-contain'} 
		{imageZoom
			? zoomEffect()
				? hasPerson
					? 'zoom-in-person'
					: 'zoom-in'
				: hasPerson
					? 'zoom-out-person'
					: 'zoom-out'
			: ''}
			"
		src={image[0]}
		alt="data"
	/>
</div>
<AssetInfo asset={image[1]} {showLocation} {showPhotoDate} {showImageDesc} {showPeopleDesc} />

<style>
	.zoom-in {
		animation: zoom-in var(--interval) ease-out normal forwards;
	}
	.zoom-in-person {
		animation: zoom-in-person var(--interval) ease-out normal forwards;
	}
	.zoom-out {
		animation: zoom-out var(--interval) ease-out normal forwards;
	}
	.zoom-out-person {
		animation: zoom-out-person var(--interval) ease-out normal forwards;
	}

	@keyframes zoom-in {
		from {
			transform: scale(1);
		}
		to {
			transform: scale(1.3);
		}
	}

	@keyframes zoom-in-person {
		from {
			transform: scale(1);
			transform-origin: center;
		}
		to {
			transform: scale(1.5);
			transform-origin: var(--posX) var(--posY);
		}
	}

	@keyframes zoom-out {
		from {
			transform: scale(1.3);
		}
		to {
			transform: scale(1);
		}
	}

	@keyframes zoom-out-person {
		from {
			transform: scale(1.5);
			transform-origin: var(--posX) var(--posY);
		}
		to {
			transform: scale(1);
			transform-origin: center;
		}
	}
</style>