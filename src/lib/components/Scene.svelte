<script lang="ts">
	import { T, useTask } from '@threlte/core';
	import { ContactShadows, Float, Grid, OrbitControls } from '@threlte/extras';
	import Particles from './emitter/Particles.svelte';
	import {
		BoxGeometry,
		DefaultLoadingManager,
		MeshBasicMaterial,
		ShaderMaterial,
		SphereGeometry,
		Vector3
	} from 'three';
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
		List
	} from 'svelte-tweakpane-ui';
	import { presets } from './presets';

	const circleTexture = useTexture('/circle.png');
	const smokeTexture = useTexture('/smoke.png');
	const snowflakeTexture = useTexture('/snowflake.png');

	const textures = Promise.all([
		circleTexture.promise,
		smokeTexture.promise,
		snowflakeTexture.promise
	]);

	let start: any;
	let stop: any;
	let box: any;
	let selectedTextureIndex = 0;
	let emitterPosition = { x: 0, y: 0, z: 0 };
	let state = '';
	let reset = 0;

	let debug = false;
	let count = 150;
	let life = 3;
	let oneShot = false;
	let explosiveness = 0;
	let spread = 75;
	let emitterScale = { x: 0, y: 0, z: 0 };
	let velocity = 4;
	let velocityRandom = 0;
	let dampen = true;
	let color =
		'rgba(255,0,255,0) 0%, rgba(255,0,255,1) 15%, rgba(255,0,255,1) 50%,rgba(255,0,255,0) 100%';
	let colorRandom = 0.12;
	let lightnessRandom = 0.2;
	let clampAlpha = false;
	let additiveBlend = true;
	let gravity = { x: 0, y: -1, z: 0 };
	let direction = { x: 0, y: 1, z: 0 };
	let wind = { x: 0, y: 0, z: 0 };
	let size = 'size(5) 0%, size(20) 100%';
	let sizeRandom = 10;
	let textureRotation = 0;
	let rotationRandom = 0;
	let driftAmount = 0;
	let driftSpeed = 0;
	let r = 0;
	let spinEmitter = false;

	const startParticles = () => {
		start();
	};

	const stopParticles = () => {
		stop();
	};

	const resetEmitter = () => {
		reset = Math.random();
	};

	$: {
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
		gravity;
		direction;
		wind;
		size;
		sizeRandom;
		textureRotation;
		rotationRandom;
		driftAmount;
		driftSpeed;
		selectedTextureIndex;
	}

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
	};

	const generateComponent = () => {
		navigator.clipboard.writeText(`<Particles
	emitterPosition={{x:${emitterPosition.x},y:${emitterPosition.y},z:${emitterPosition.z}}}
	clampAlpha={${clampAlpha}}
	{debug}
	{emitterPosition}
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

	const stateChanged = (event: CustomEvent) => {
		state = event.detail.state;
	};
</script>

<Pane title="Particles" position="fixed">
	<Checkbox bind:value={debug} label="debug" />
	<Folder title="Main">
		<Text bind:value={state} label="state" disabled />
		<Button
			on:click={startParticles}
			title="start"
			disabled={state !== 'finished' ? true : false}
		/>
		<Button on:click={stopParticles} title="stop" disabled={state !== 'running' ? true : false} />
		<Separator />
		<Point value={emitterPosition} label="emitter position" min={-10} max={10} />
		<Point bind:value={emitterScale} label="emitter scale" min={0} max={10} />
		<Separator />
		<Checkbox bind:value={oneShot} label="one shot" />
		<Slider bind:value={count} min={0} max={500} format={(v) => v.toFixed(0)} label="count" />
		<Slider bind:value={life} min={0} max={10} format={(v) => v.toFixed(1)} label="life" />
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
		<List
			bind:value={selectedTextureIndex}
			label="alpha map"
			options={{ 'circle texture': 0, 'smoke texture': 1, 'snowflake texture': 2 }}
		/>
	</Folder>
	<Folder title="Presets">
		<Button on:click={() => applyPreset(presets['fountain'])} title="fountain" />
		<Button on:click={() => applyPreset(presets['snow'])} title="snow" />
		<Button on:click={() => applyPreset(presets['fire'])} title="fire" />
		<Button on:click={() => applyPreset(presets['tornado'])} title="tornado" />
		<Button on:click={() => applyPreset(presets['fireflies'])} title="fireflies" />
		<Separator />
		<Button on:click={() => generateComponent()} title="Generate Component" />
	</Folder>
</Pane>

{#key reset}
	{#await textures then t}
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
			alphaMap={t[selectedTextureIndex]}
			{driftAmount}
			{driftSpeed}
			bind:start
			bind:stop
			on:stateChanged={stateChanged}
		/>
	{/await}
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

<T.DirectionalLight intensity={0.8} position.x={5} position.y={10} />
<T.AmbientLight intensity={0.2} />

<Grid
	gridSize={[10, 10]}
	cellColor={'#46536b'}
	position.y={-0.001}
	sectionColor="#ffffff"
	sectionThickness={0}
	fadeDistance={50}
/>
<!--
<ContactShadows scale={10} blur={2} far={2.5} opacity={0.5} />

<Float floatIntensity={1} floatingRange={[0, 1]}>
	<T.Mesh position.y={0.2} position.z={-5.75}>
		<T.BoxGeometry />
		<T.MeshStandardMaterial color="#0059BA" />
	</T.Mesh>
</Float>

<Float floatIntensity={1} floatingRange={[0, 1]}>
	<T.Mesh position={[5.2, 0.5, 0.75]} rotation.x={5} rotation.y={71}>
		<T.TorusKnotGeometry args={[0.5, 0.15, 100, 12, 2, 3]} />
		<T.MeshStandardMaterial color="#F85122" />
	</T.Mesh>
</Float>

<Float floatIntensity={1} floatingRange={[0, 1]}>
	<T.Mesh position={[-5.4, 1.5, 2.75]} rotation={[-5, 128, 10]}>
		<T.IcosahedronGeometry />
		<T.MeshStandardMaterial color="#F8EBCE" />
	</T.Mesh>
</Float>-->
