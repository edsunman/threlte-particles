<script lang="ts">
	import { T, useTask } from '@threlte/core';
	import { Grid, OrbitControls } from '@threlte/extras';
	import Particles from './emitter/Particles.svelte';
	import { useTexture, TransformControls } from '@threlte/extras';
	import {
		Pane,
		Slider,
		Checkbox,
		Button,
		Folder,
		Separator,
		Text,
		Point,
		Textarea,
		List,
		Element
	} from 'svelte-tweakpane-ui';
	import { presets } from './presets';
	import { MeshBasicMaterial, TextureLoader } from 'three';
	import Peformance from './Peformance.svelte';

	/*
	 *	Image Loading
	 */
	const imageUrls = $state(['', '/circle.png', '/smoke.png', '/snowflake.png', '']);
	const mapUrls =  $state(['', '']);
	let texturesLoaded = $state(false);
	let textures: any = $state();
	let mapTextures: any = [false, false];
	const texturesPromise = Promise.all([
		useTexture('/circle.png'),
		useTexture('/smoke.png'),
		useTexture('/snowflake.png')
	]);
	texturesPromise.then((values) => {
		textures = [false, ...values];
		texturesLoaded = true;
	});
	const imageOptions: { [key: string]: number } = {
		'no texture': 0,
		'circle texture': 1,
		'smoke texture': 2,
		'snowflake texture': 3,
		'custom texture': 4
	};

	let start: any = $state();
	let stop: any = $state();
	let box: any = $state();
	let selectedTextureIndex = $state(1);
	let selectedMapIndex = $state(0);
	let selectedGeometryIndex = $state(0);
	let emitterPosition = $state({ x: 0, y: 0, z: 0 });
	let eState = $state('');
	let reset = $state(0);

	let debug = $state(false);
	let count = $state(150);
	let life = $state(3);
	let oneShot = $state(false);
	let explosiveness = $state(0.5);
	let spread = $state(75);
	let emitterScale = $state({ x: 0, y: 0, z: 0 });
	let velocity = $state(5);
	let velocityRandom = $state(0);
	let dampen = $state(true);
	let color = $state(
		'rgba(255,0,255,0) 0%, rgba(255,0,255,1) 15%, rgba(255,0,255,1) 50%,rgba(255,0,255,0) 100%'
	);
	let colorRandom = $state(0.12);
	let lightnessRandom = $state(0.2);
	let clampAlpha = $state(false);
	let additiveBlend = $state(true);
	let gravity = $state({ x: 0, y: -2, z: 0 });
	let direction = $state({ x: 0, y: 1, z: 0 });
	let wind = $state({ x: 0, y: 0, z: 0 });
	let size = $state('size(3) 0%, size(20) 100%');
	let sizeRandom = $state(10);
	let textureRotation = $state(0);
	let rotationRandom = $state(0);
	let driftAmount = $state(0.3);
	let driftSpeed = $state(0.3);
	let r = $state(0);
	let spinEmitter = $state(false);

	const resetEmitter = () => {
		reset = Math.random();
	};

	$effect.pre(() => {
		resetEmitter();
		life;
		oneShot;
		count;
		explosiveness;
		spread;
		velocity;
		velocityRandom;
		dampen;
		color;
		colorRandom;
		lightnessRandom;
		clampAlpha;
		additiveBlend;
		gravity.x,gravity.y,gravity.z;
		direction.x,direction.y,direction.z;
		wind.x,wind.y,wind.z;
		size;
		sizeRandom;
		textureRotation;
		rotationRandom;
		driftAmount;
		driftSpeed;
		selectedTextureIndex;
		textures;
		selectedMapIndex;
		mapTextures;
		selectedGeometryIndex;
	});

	const applyPreset = (preset: any) => {
		emitterPosition = preset.emitterPosition;
		count = preset.count;
		life = preset.life;
		oneShot = preset.oneShot;
		explosiveness = preset.explosiveness;
		spread = preset.spread;
		emitterScale = preset.emitterScale;
		velocity = preset.velocity;
		velocityRandom = preset.velocityRandom;
		dampen = preset.dampen;
		color = preset.color;
		colorRandom = preset.colorRandom;
		lightnessRandom = preset.lightnessRandom;
		clampAlpha = preset.clampAlpha;
		additiveBlend = preset.additiveBlend;
		gravity = preset.gravity;
		direction = preset.direction;
		wind = preset.wind;
		size = preset.size;
		sizeRandom = preset.sizeRandom;
		textureRotation = preset.textureRotation;
		rotationRandom = preset.rotationRandom;
		driftAmount = preset.driftAmount;
		driftSpeed = preset.driftSpeed;
		selectedTextureIndex = preset.selectedTextureIndex;
		spinEmitter = Object.hasOwn(preset, 'spinEmitter') ? true : false;
		selectedMapIndex = 0;
		Object.hasOwn(preset, 'geometry') ? (selectedGeometryIndex = 1) : (selectedGeometryIndex = 0);
	};

	const generateComponent = () => {
		navigator.clipboard.writeText(`<Particles
	emitterPosition={{x:${emitterPosition.x},y:${emitterPosition.y},z:${emitterPosition.z}}}
	emitterScale={{x:${emitterScale.x},y:${emitterScale.y},z:${emitterScale.z}}}
	count={${count}}
	life={${life}}
	explosiveness={${explosiveness}}
	spread={${spread}}
	gravity={{x:${gravity.x},y:${gravity.y},z:${gravity.z}}}
	direction={{x:${direction.x},y:${direction.y},z:${direction.z}}}
	wind={{x:${wind.x},y:${wind.y},z:${wind.z}}}
	driftSpeed={${driftSpeed}}
	velocity={${velocity}}
	velocityRandom={${velocityRandom}}
	size={'${size}'}
	sizeRandom={${sizeRandom}}
	color={'${color}'}
	colorRandom={${colorRandom}}
	lightnessRandom={${lightnessRandom}}
	textureRotation={${textureRotation}}
	rotationRandom={${rotationRandom}}
	dampen={${dampen}}
	oneShot={${oneShot}}
	clampAlpha={${clampAlpha}}
	additiveBlend={${additiveBlend}}
	{alphaMap}
/>`);

		alert('Copied to clipboard');
	};

	useTask((delta) => {
		if (spinEmitter) {
			r += delta * 3;
		} else {
			r = 0;
		}
		if (box) {
			emitterPosition.x = box.position.x;
			emitterPosition.y = box.position.y;
			emitterPosition.z = box.position.z;
		}
	});

	async function getFile(alpha = true) {
		// @ts-ignore
		const [fileHandle] = await window.showOpenFilePicker({
			types: [
				{
					description: 'Images',
					accept: {
						'image/*': ['.png', '.gif', '.jpeg', '.jpg']
					}
				}
			],
			excludeAcceptAllOption: true,
			multiple: false
		});
		const file = await fileHandle.getFile();
		if (!file) return;
		if (alpha) {
			imageUrls[4] = URL.createObjectURL(file);
			let newTex = new TextureLoader().load(imageUrls[4]);
			textures[4] = newTex;
			resetEmitter();
		} else {
			mapUrls[1] = URL.createObjectURL(file);
			mapTextures[1] = new TextureLoader().load(mapUrls[1]);
			resetEmitter();
		}
	}
</script>

<Peformance />

<Pane title="Particles" position="fixed">
	<Checkbox bind:value={debug} label="debug" />
	<Folder title="Main" expanded={false}>
		<Text bind:value={eState} label="state" disabled />
		<Button
			on:click={() => start()}
			title="start"
			disabled={eState !== 'finished' ? true : false}
		/>
		<Button on:click={() => stop()} title="stop" disabled={eState !== 'running' ? true : false} />
		<Separator />
		<Point bind:value={emitterPosition} label="emitter position" min={-10} max={10} />
		<Point bind:value={emitterScale} label="emitter scale" min={0} max={10} />
		<Separator />
		<Checkbox bind:value={oneShot} label="one shot" />
		<Slider bind:value={count} min={0} max={1000} format={(v) => v.toFixed(0)} label="count" />
		<Slider bind:value={life} min={0} max={10} format={(v) => v.toFixed(1)} label="life" />
		<List
			bind:value={selectedGeometryIndex}
			label="custom geometry"
			options={{ none: 0, 'ring geometry': 1 }}
		/> 
	</Folder>
	<Folder title="Movement" expanded={false}>
		<Slider bind:value={velocity} min={0} max={20} format={(v) => v.toFixed(1)} label="velocity" />
		<Slider
			bind:value={velocityRandom}
			min={0}
			max={10}
			format={(v) => v.toFixed(1)}
			label="velocity random"
		/>
		<Slider
			bind:value={spread}
			min={0}
			max={360}
			format={(v) => `${v.toFixed(0)}°`}
			label="spread"
		/>
		<Slider
			bind:value={explosiveness}
			min={0}
			max={1}
			format={(v) => v.toFixed(2)}
			label="explosiveness"
		/>
		<Checkbox bind:value={dampen} label="dampen" />
		<Point bind:value={direction} label="direction" min={-1} max={1} />
		<Point bind:value={gravity} label="gravity" min={-5} max={5} />
		<Point bind:value={wind} label="wind" min={-5} max={5} />
		<Slider
			bind:value={driftAmount}
			min={0}
			max={5}
			format={(v) => v.toFixed(1)}
			label="drift amount"
		/>
		<Slider
			bind:value={driftSpeed}
			min={0}
			max={5}
			format={(v) => v.toFixed(1)}
			label="drift speed"
		/>
	</Folder>
		<Folder title="Apperance" expanded={false}>
		<Textarea bind:value={color} label="color" rows={4} />
		<Slider
			bind:value={colorRandom}
			min={0}
			max={1}
			format={(v) => v.toFixed(2)}
			label="color random"
		/>
		<Slider
			bind:value={lightnessRandom}
			min={0}
			max={1}
			format={(v) => v.toFixed(2)}
			label="lightness random"
		/>
		<Textarea bind:value={size} label="size" rows={4} />
		<Slider
			bind:value={sizeRandom}
			min={0}
			max={20}
			format={(v) => v.toFixed(1)}
			label="size random"
		/>
		<Slider
			bind:value={textureRotation}
			min={-10}
			max={10}
			format={(v) => v.toFixed(1)}
			label="texture rotation"
		/>
		<Slider
			bind:value={rotationRandom}
			min={0}
			max={10}
			format={(v) => v.toFixed(1)}
			label="rotation random"
		/>
		<Checkbox bind:value={clampAlpha} label="clamp alpha" />
		<Checkbox bind:value={additiveBlend} label="addative blend" />
	</Folder>
	<Folder title="Texture" expanded={false}>
		<List
			bind:value={selectedMapIndex}
			label="map"
			options={{ 'no texture': 0, 'custom texture': 1 }}
		/>
		{#if selectedMapIndex === 1}
			<Button on:click={() => getFile(false)} title="Load image" />
			<Element>
				<!-- svelte-ignore element_invalid_self_closing_tag -->
				<div
					style:background-image={'url(' + mapUrls[selectedMapIndex] + ')'}
					style="height:65px;width:65px;background-size: contain;background-repeat: no-repeat;margin-left:auto;background-color:black"
				/>
			</Element>
		{/if}
		<List bind:value={selectedTextureIndex} label="alpha map" options={imageOptions} />
		{#if selectedTextureIndex === 4}<Button on:click={() => getFile()} title="Load image" />{/if}
		{#if selectedTextureIndex !== 0}
			<Element>
				<!-- svelte-ignore element_invalid_self_closing_tag -->
				<div
					style:background-image={'url(' + imageUrls[selectedTextureIndex] + ')'}
					style="height:65px;width:65px;background-size: contain;background-repeat: no-repeat;margin-left:auto;background-color:black"
				/>
			</Element>
		{/if}
	</Folder>
	<Folder title="Presets">
		<Button on:click={() => applyPreset(presets['fountain'])} title="fountain" />
		<Button on:click={() => applyPreset(presets['snow'])} title="snow" />
		<Button on:click={() => applyPreset(presets['fire'])} title="fire" />
		<Button on:click={() => applyPreset(presets['portal'])} title="portal" />
		<Button on:click={() => applyPreset(presets['tornado'])} title="tornado" />
		<Button on:click={() => applyPreset(presets['fireflies'])} title="fireflies" />
		<Separator />
		<Button on:click={() => generateComponent()} title="Generate Component" />
	</Folder>
</Pane>

{#snippet ringGeometry()}
	<T.RingGeometry args={[1.9, 2]} />
{/snippet}

{#key reset}
	{#if texturesLoaded}
			<Particles
				rotation.y={r}
				{clampAlpha}
				{debug}
				{emitterPosition}
				{textureRotation}
				{emitterScale}
				{count}
				{life}
				{oneShot}
				{spread}
				{velocity}
				{velocityRandom}
				{dampen}
				{wind}
				{gravity}
				{direction}
				{sizeRandom}
				{explosiveness}
				{colorRandom}
				{lightnessRandom}
				{color}
				{additiveBlend}
				{size}
				{rotationRandom}
				alphaMap={textures[selectedTextureIndex]}
				map={mapTextures[selectedMapIndex]}
				{driftAmount}
				{driftSpeed}
				bind:start
				bind:stop
				emitterStateChanged={(e: string) => (eState = e)}
				customGeometry={selectedGeometryIndex === 0 ? null : ringGeometry}
			/>
	{/if}
{/key}

{#if debug}
	<T.Mesh
		let:ref
		bind:ref={box}
		scale={[1, 1, 1]}
		position={[emitterPosition.x, emitterPosition.y, emitterPosition.z]}
	>
		<TransformControls object={ref} />
	</T.Mesh>
{/if}

<T.PerspectiveCamera makeDefault position={[-15, 5, 15]} fov={20}>
	<OrbitControls autoRotate enableDamping autoRotateSpeed={1} target.y={1.5} />
</T.PerspectiveCamera>

<Grid
	gridSize={[10, 10]}
	cellColor={'#46536b'}
	position.y={-0.001}
	sectionColor="#ffffff"
	sectionThickness={0}
	fadeDistance={50}
/>
