<script lang="ts">
	import { T, useTask, useThrelte } from '@threlte/core';
	import {
		BufferGeometry,
		Float32BufferAttribute,
		Vector3,
		ShaderMaterial,
		NormalBlending,
		AdditiveBlending,
		type Texture,
		Mesh
	} from 'three';
	import { MeshSurfaceSampler } from 'three/examples/jsm/math/MeshSurfaceSampler.js';
	import { createEventDispatcher } from 'svelte';
	import {
		ramdomPointInsideCube,
		randomDirectionSpread,
		createGradientObject,
		randomNumber
	} from './util';
	import fragmentShader from './particles-fragment.glsl?raw';
	import vertexShader from './particles-vertex.glsl?raw';

	/** Position of the emitter. You can update while the emitter is running. */
	export let emitterPosition = { x: 0, y: 0, z: 0 };
	/** Scale of the emitter. You can update while the emitter is running. */
	export let emitterScale = { x: 0, y: 0, z: 0 };
	/** The number of particles. */
	export let count = 5;
	/** The life of each particle in seconds. */
	export let life = 2;
	/** Value between 0 and 1. Emit particles one after another or all at once. */
	export let explosiveness = 0;
	/** Value between 0 and 360 degrees. Gives the particles a random direction within this range.  */
	export let spread = 0;
	// TODO: direction should be reactive?
	/** Normalised direction vector. */
	export let direction = { x: 0, y: 1, z: 0 };
	/** Gravity direction vector. Higher numbers for stronger force. Gets stronger over life of particle. */
	export let gravity = { x: 0, y: 0, z: 0 };
	/** Gravity direction vector. Higher numbers for stronger force. Constant force over life of particle. */
	export let wind = { x: 0, y: 0, z: 0 };
	/** Use in combination with driftSpeed to give the particles random movement. */
	export let driftAmount = 0;
	/** Use in combination with driftAmount to give the particles random movement. */
	export let driftSpeed = 0;
	/** Initial particle force. */
	export let velocity = 3;
	/** Randomise the velocity by this amount in both diretions. */
	export let velocityRandom = 0;
	/** Size of particle over it's life. */
	export let size: string | number = 3;
	/** Randomise the size by this amount in both diretions. */
	export let sizeRandom = 0;
	/** Color of particle over it's life. */
	export let color: string = '';
	/** Value between 0 and 1 to randomise the hue. */
	export let colorRandom = 0;
	/** Value between 0 and 1 to randomise the lightness. */
	export let lightnessRandom = 0;
	/** Speed to rotate the texture. */
	export let textureRotation = 0;
	/** Randomise the rotation by this amount in both diretions. */
	export let rotationRandom = 0;
	/** Slow the particle to a stop over it's life. */
	export let dampen = false;
	/** Run the emitter once then stop. */
	export let oneShot = false;
	/** Clamp alpha map values for a hard edge stylised look */
	export let clampAlpha = false;
	/** Blend transparent particles for a glow effect */
	export let additiveBlend = false;
	/** Texture for alpha values. White is opaque black is transparent. */
	export let alphaMap: Texture | undefined = undefined;
	/** Texture */
	export let map: Texture | undefined = undefined;
	/** Show emitter outline. */
	export let debug = false;
	export let boundingSphereRadius = 5;

	let position = new Vector3(emitterPosition.x, emitterPosition.y, emitterPosition.z);
	let directionVector = new Vector3(direction.x, direction.y, direction.z);
	let emitterLife = 0;
	let state = '';
	let newPosition: { x: number; y: number; z: number };
	let paused = false;
	let pausedTime: number;
	let useAlphaMap = alphaMap ? 1 : 0;
	let useMap = map ? 1 : 0;
	let material: ShaderMaterial;
	let emitterMesh: Mesh;
	let useCustomGeometry = false;
	let sampler: any;

	const { renderer } = useThrelte();
	const pixelRatio = renderer.getPixelRatio();
	const positionAttributeArray: number[] = [];
	const lifeAttributeArray: number[] = [];
	const parsedColorGradient = createGradientObject(color, 16);
	const parsedSizeGradient = createGradientObject(size, 4);
	const dispatch = createEventDispatcher();
	const geometry = new BufferGeometry();
	const particles: any = [];
	const samplerVector = new Vector3();

	export const start = () => {
		if (state !== 'finished') {
			console.warn('particles: start() was called but the emitter is ' + state + ', not finished.');
			return;
		}
		paused = false;
		emitterLife = 0;
	};

	export const stop = () => {
		if (oneShot) {
			console.warn('particles: stop() has no effect when oneShot is set to true.');
			return;
		}
		if (state !== 'running') {
			console.warn('particles: stop() was called but the emitter is ' + state + ', not running.');
			return;
		}
		paused = true;
		pausedTime = emitterLife;
	};

	const setupParticles = () => {
		if (map) map.flipY = false;
		if (alphaMap) alphaMap.flipY = false;
		for (let i = 0; i < count; i++) {
			const pDirection = new Vector3().copy(directionVector.normalize());
			if (spread > 0) pDirection.copy(randomDirectionSpread(pDirection, spread / 2));
			const pVelocity =
				velocityRandom > 0
					? randomNumber(velocity - velocityRandom / 2, velocity + velocityRandom / 2)
					: velocity;
			let pSize = sizeRandom > 0 ? randomNumber(-sizeRandom / 2, sizeRandom / 2) : 0;
			pSize = pSize < 0 ? 0 : pSize;
			const pColor = colorRandom > 0 ? randomNumber(-colorRandom / 2, colorRandom / 2) : 0;
			const pLightness =
				lightnessRandom > 0 ? randomNumber(-lightnessRandom / 2, lightnessRandom / 2) : 0;
			const pRotation =
				rotationRandom > 0
					? randomNumber(textureRotation - rotationRandom / 2, textureRotation + rotationRandom / 2)
					: textureRotation;
			pDirection.multiplyScalar(pVelocity);
			const pRandomSeed = Math.random();
			particles.push({
				position: { x: 0, y: 0, z: 0 },
				sizeRandom: pSize,
				colorRandom: pColor,
				lightnessRandom: pLightness,
				life: -(life / count) * i * (1 - explosiveness),
				maxLife: life,
				rotation: pRotation,
				velocity: pDirection,
				randomSeed: pRandomSeed
			});
		}

		const sizes = [];
		const colors: any = [];
		const lightness: any = [];
		const rotations: any = [];
		const velocities: any = [];
		const randomSeeds: any = [];
		for (let particle of particles) {
			positionAttributeArray.push(particle.position.x, particle.position.y, particle.position.z);
			sizes.push(particle.sizeRandom);
			colors.push(particle.colorRandom);
			lightness.push(particle.lightnessRandom);
			rotations.push(particle.rotation);
			lifeAttributeArray.push(particle.life);
			velocities.push(particle.velocity.x, particle.velocity.y, particle.velocity.z);
			randomSeeds.push(particle.randomSeed);
		}
		geometry.setAttribute('position', new Float32BufferAttribute(positionAttributeArray, 3));
		geometry.setAttribute('sizeRandom', new Float32BufferAttribute(sizes, 1));
		geometry.setAttribute('colorRandom', new Float32BufferAttribute(colors, 1));
		geometry.setAttribute('lightnessRandom', new Float32BufferAttribute(lightness, 1));
		geometry.setAttribute('rotation', new Float32BufferAttribute(rotations, 1));
		geometry.setAttribute('life', new Float32BufferAttribute(lifeAttributeArray, 1));
		geometry.setAttribute('velocity', new Float32BufferAttribute(velocities, 3));
		geometry.setAttribute('randomSeed', new Float32BufferAttribute(randomSeeds, 1));

		geometry.attributes.sizeRandom.needsUpdate = true;
		geometry.attributes.velocity.needsUpdate = true;
		geometry.attributes.colorRandom.needsUpdate = true;
		geometry.attributes.lightnessRandom.needsUpdate = true;
		geometry.attributes.rotation.needsUpdate = true;
		geometry.attributes.life.needsUpdate = true;
		geometry.attributes.randomSeed.needsUpdate = true;
	};

	setupParticles();

	const distributePreBirthParticles = () => {
		particles.forEach((particle: any, i: number) => {
			particle.life = -(life / count) * i * (1 - explosiveness);
		});
	};

	const positionNewParticle = (index: number) => {
		if (useCustomGeometry) {
			sampler.sample(samplerVector);
			emitterMesh.updateMatrix();
			samplerVector.applyMatrix4(emitterMesh.matrix);
			positionAttributeArray[index * 3] = samplerVector.x;
			positionAttributeArray[index * 3 + 1] = samplerVector.y;
			positionAttributeArray[index * 3 + 2] = samplerVector.z;
		} else if (emitterScale.x > 0 || emitterScale.y > 0 || emitterScale.z) {
			newPosition = ramdomPointInsideCube(position, emitterScale);
			positionAttributeArray[index * 3] = newPosition.x;
			positionAttributeArray[index * 3 + 1] = newPosition.y;
			positionAttributeArray[index * 3 + 2] = newPosition.z;
		} else {
			positionAttributeArray[index * 3] = position.x;
			positionAttributeArray[index * 3 + 1] = position.y;
			positionAttributeArray[index * 3 + 2] = position.z;
		}
	};

	const stateChanged = () => {
		dispatch('stateChanged', {
			state
		});
	};

	const positionUpdated = (p: any) => {
		position.x = p.x;
		position.y = p.y;
		position.z = p.z;
		if (!geometry.boundingSphere) {
			geometry.computeBoundingSphere();
		}
		if (!geometry.boundingSphere) return;
		geometry.boundingSphere.radius = boundingSphereRadius;
		geometry.boundingSphere.center = position;
	};

	$: positionUpdated(emitterPosition);

	useTask((delta) => {
		emitterLife += delta;
		let newState = 'running';
		if (emitterLife < life) {
			// emmitting new particles
			newState = 'starting';
		}
		if (oneShot) {
			if (emitterLife >= life) {
				// emmitting no more particles
				newState = 'stopping';
			}
			if (emitterLife >= life + life * (1 - explosiveness) + 0.1) {
				// all particles have died
				newState = 'finished';
			}
		} else if (paused) {
			if (emitterLife >= pausedTime) {
				// emmitting no more particles
				newState = 'stopping';
			}
			if (emitterLife >= pausedTime + life + 0.1) {
				// all particles have died
				newState = 'finished';
			}
		}
		if (state !== newState) {
			state = newState;
			stateChanged();
			if (state === 'finished') distributePreBirthParticles();
		}

		if (state === 'starting') {
			// move unborn particles to emitter position
			particles.forEach((particle: any, index: number) => {
				if (particle.life <= 0) {
					positionNewParticle(index);
				}
			});
		}
		if (state !== 'finished') {
			// update particles
			lifeAttributeArray.length = 0;
			particles.forEach((particle: any, index: number) => {
				particle.life += delta;
				if (particle.life > particle.maxLife) {
					// particle died
					if (state === 'running' && !paused) {
						particle.life = 0;
						positionNewParticle(index);
					}
				}
				lifeAttributeArray.push(particle.life);
			});
			const lifeAttribute = (geometry.getAttribute('life') as any).set(lifeAttributeArray);
			lifeAttribute.needsUpdate = true;
			geometry.setAttribute('position', new Float32BufferAttribute(positionAttributeArray, 3));
		}
	});

	const geometryLoaded = (d: Mesh) => {
		if (!d || d.geometry.name === 'defaultBox' || !('position' in d.geometry.attributes)) return;
		useCustomGeometry = true;
		d.geometry = d.geometry.toNonIndexed();
		sampler = new MeshSurfaceSampler(d).build();
	};

	$: geometryLoaded(emitterMesh);
</script>

<T.Points {geometry} {...$$restProps} name="particles">
	<T.ShaderMaterial
		blending={additiveBlend ? AdditiveBlending : NormalBlending}
		bind:ref={material}
		{vertexShader}
		{fragmentShader}
		depthTest
		depthWrite={false}
		transparent
		vertexColors
		uniforms={{
			alphaMap: {
				value: alphaMap
			},
			useAlphaMap: {
				value: useAlphaMap
			},
			map: {
				value: map
			},
			useMap: {
				value: useMap
			},
			maxLifetime: {
				value: life
			},
			dampen: {
				value: dampen ? 1 : 0
			},
			driftSpeed: {
				value: driftSpeed
			},
			driftAmount: {
				value: driftAmount
			},
			colorStops: {
				value: parsedColorGradient.stops
			},
			colors: {
				value: parsedColorGradient.values
			},
			sizeStops: {
				value: parsedSizeGradient.stops
			},
			sizes: {
				value: parsedSizeGradient.values
			},
			wind: {
				value: [wind.x, wind.y, wind.z]
			},
			gravity: {
				value: [gravity.x, gravity.y, gravity.z]
			},
			emitterPosition: {
				value: position
			},
			useClamp: {
				value: clampAlpha ? 1 : 0
			},
			screenPixelRatio: {
				value: pixelRatio ? pixelRatio : 1
			}
		}}
	/>
</T.Points>

<T.Mesh
	bind:ref={emitterMesh}
	scale={[emitterScale.x, emitterScale.y, emitterScale.z]}
	position={[position.x, position.y, position.z]}
	name="emitterDebugMesh"
>
	<slot>
		<T.BoxGeometry name="defaultBox" />
	</slot>
	<T.MeshBasicMaterial wireframe visible={debug} />
</T.Mesh>
