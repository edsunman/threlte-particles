<script lang="ts">
	import { T, useTask, useThrelte } from '@threlte/core';
	import {
		BufferGeometry,
		Float32BufferAttribute,
		Vector3,
		type Texture,
		ShaderMaterial,
		NormalBlending,
		AdditiveBlending
	} from 'three';
	import { createEventDispatcher } from 'svelte';
	import {
		ramdomPointInsideCube,
		randomDirectionSpread,
		createGradientObject,
		randomNumber
	} from './util';
	import fragmentShader from './particles-fragment.glsl?raw';
	import vertexShader from './particles-vertex.glsl?raw';

	/** Position of the emitter */
	export let emitterPosition = { x: 0, y: 0, z: 0 };
	/** Scale of the emitter. */
	export let emitterScale = { x: 0, y: 0, z: 0 };
	/** The number of particles. */
	export let count = 10;
	/** The life of each particle in seconds. */
	export let life = 2;
	/** Value between 0 and 1. Emit particles one after another or all at once. */
	export let explosiveness = 0;
	export let spread = 0;
	// TODO: direction should be reactive?
	export let direction = { x: 0, y: 0, z: 0 };
	export let gravity = { x: 0, y: 0, z: 0 };
	export let wind = { x: 0, y: 0, z: 0 };
	/** The drift speed. */
	export let driftSpeed = 0;
	export let driftAmount = 0;
	export let velocity = 1;
	export let velocityRandom = 0;
	export let size = '';
	export let sizeRandom = 0;
	export let color: string = '';
	export let colorRandom = 0;
	export let lightnessRandom = 0;
	export let rotation = 0;
	export let rotationRandom = 0;
	export let dampen = true;
	export let oneShot = false;
	export let debug = false;
	export let boundingSphereRadius = 5;
	export let clampAlpha = false;
	export let additiveBlend = false;
	export let alphaMap: Texture | undefined = undefined;
	export let map: Texture | undefined = undefined;

	let position = new Vector3(emitterPosition.x, emitterPosition.y, emitterPosition.z);
	let directionVector = new Vector3(direction.x, direction.y, direction.z);
	let emitterLife = 0;
	let state = '';
	let newPosition;
	let paused = false;
	let pausedTime: number;
	let useAlphaMap = alphaMap ? 1 : 0;
	let useMap = map ? 1 : 0;
	let material: ShaderMaterial;

	const positionAttributeArray: number[] = [];
	const lifeAttributeArray: number[] = [];
	const parsedColorGradient = createGradientObject(color, 16);
	const parsedSizeGradient = createGradientObject(size, 4);
	const dispatch = createEventDispatcher();
	const geometry = new BufferGeometry();

	export const start = () => {
		if (state !== 'finished') {
			console.warn('start() was called but the emitter is ' + state + ', not finished.');
			return;
		}
		paused = false;
		emitterLife = 0;
	};

	export const stop = () => {
		if (oneShot) {
			console.warn('stop() has no effect when oneShot is set to true.');
			return;
		}
		if (state !== 'running') {
			console.warn('stop() was called but the emitter is ' + state + ', not running.');
			return;
		}
		paused = true;
		pausedTime = emitterLife;
	};

	const particles: any = [];

	// runs once when when component mounts
	const setupParticles = () => {
		for (let i = 0; i < count; i++) {
			const pDirection = new Vector3().copy(directionVector.normalize());
			if (spread > 0) pDirection.copy(randomDirectionSpread(pDirection, spread / 2));
			const pVelocity =
				velocityRandom > 0
					? randomNumber(velocity - velocityRandom / 2, velocity + velocityRandom / 2)
					: velocity;
			const pSize = sizeRandom > 0 ? randomNumber(-sizeRandom / 2, sizeRandom / 2) : 0;
			const pColor = colorRandom > 0 ? randomNumber(-colorRandom / 2, colorRandom / 2) : 0;
			const pLightness =
				lightnessRandom > 0 ? randomNumber(-lightnessRandom / 2, lightnessRandom / 2) : 0;
			const pRotation =
				rotationRandom > 0
					? randomNumber(rotation - rotationRandom / 2, rotation + rotationRandom / 2)
					: rotation;
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
		//if (emitterScale.x > 0 || emitterScale.y > 0 || emitterScale.z) {
		newPosition = ramdomPointInsideCube(position, emitterScale);
		positionAttributeArray[index * 3] = newPosition.x;
		positionAttributeArray[index * 3 + 1] = newPosition.y;
		positionAttributeArray[index * 3 + 2] = newPosition.z;
		//} else {
		//	positionAttributeArray[index * 3] = position.x;
		//	positionAttributeArray[index * 3 + 1] = position.y;
		//	positionAttributeArray[index * 3 + 2] = position.z;
		//	}
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
		if (delta > 0.5) return;
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
</script>

<T.Points {geometry} {...$$restProps}>
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
			}
		}}
	/>
</T.Points>

<T.Mesh
	let:ref
	scale={[emitterScale.x, emitterScale.y, emitterScale.z]}
	position={[position.x, position.y, position.z]}
>
	<T.BoxGeometry />
	<T.MeshBasicMaterial wireframe visible={debug} />
	<slot {ref} />
</T.Mesh>
