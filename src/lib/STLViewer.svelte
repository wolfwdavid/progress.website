<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { STLLoader } from 'three/addons/loaders/STLLoader.js';
	import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

	let { url, onclose }: { url: string; onclose?: () => void } = $props();

	let container: HTMLDivElement;

	onMount(() => {
		const width = container.clientWidth;
		const height = container.clientHeight;

		// Scene
		const scene = new THREE.Scene();
		scene.background = new THREE.Color(0x0a081c);

		// Camera
		const camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 2000);
		camera.position.set(0, 50, 150);

		// Renderer
		const renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(width, height);
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
		renderer.toneMapping = THREE.ACESFilmicToneMapping;
		renderer.toneMappingExposure = 1.2;
		container.appendChild(renderer.domElement);

		// Controls
		const controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.08;
		controls.autoRotate = true;
		controls.autoRotateSpeed = 2;

		// Lighting
		const ambientLight = new THREE.AmbientLight(0x404060, 0.8);
		scene.add(ambientLight);

		const keyLight = new THREE.DirectionalLight(0xffffff, 1.5);
		keyLight.position.set(80, 120, 80);
		scene.add(keyLight);

		const fillLight = new THREE.DirectionalLight(0x8888ff, 0.6);
		fillLight.position.set(-60, 40, -60);
		scene.add(fillLight);

		const rimLight = new THREE.DirectionalLight(0xbb44ff, 0.8);
		rimLight.position.set(0, -50, -80);
		scene.add(rimLight);

		// Grid helper
		const grid = new THREE.GridHelper(200, 20, 0x333355, 0x1a1a33);
		scene.add(grid);

		// Load STL
		const loader = new STLLoader();
		loader.load(url, (geometry) => {
			geometry.computeVertexNormals();

			const material = new THREE.MeshPhysicalMaterial({
				color: 0x8855cc,
				metalness: 0.3,
				roughness: 0.4,
				clearcoat: 0.3,
				clearcoatRoughness: 0.25,
				envMapIntensity: 0.5
			});

			const mesh = new THREE.Mesh(geometry, material);

			// Center and scale
			geometry.computeBoundingBox();
			const box = geometry.boundingBox!;
			const center = new THREE.Vector3();
			box.getCenter(center);
			mesh.position.sub(center);

			const size = new THREE.Vector3();
			box.getSize(size);
			const maxDim = Math.max(size.x, size.y, size.z);
			const scale = 80 / maxDim;
			mesh.scale.setScalar(scale);

			scene.add(mesh);

			// Fit camera
			const dist = maxDim * scale * 1.6;
			camera.position.set(dist * 0.6, dist * 0.5, dist * 0.8);
			controls.target.set(0, 0, 0);
			controls.update();
		});

		// Animation loop
		let animId: number;
		function animate() {
			animId = requestAnimationFrame(animate);
			controls.update();
			renderer.render(scene, camera);
		}
		animate();

		// Resize
		function onResize() {
			const w = container.clientWidth;
			const h = container.clientHeight;
			camera.aspect = w / h;
			camera.updateProjectionMatrix();
			renderer.setSize(w, h);
		}
		window.addEventListener('resize', onResize);

		return () => {
			cancelAnimationFrame(animId);
			window.removeEventListener('resize', onResize);
			renderer.dispose();
			controls.dispose();
		};
	});
</script>

<div class="stl-backdrop" onclick={onclose} role="presentation"></div>
<div class="stl-modal">
	<button class="stl-close" onclick={onclose}>&times;</button>
	<div class="stl-header">
		<h2 class="stl-title">EATING TOOL</h2>
		<p class="stl-subtitle">Interactive 3D Model &mdash; Drag to rotate, scroll to zoom</p>
	</div>
	<div class="stl-container" bind:this={container}></div>
</div>

<style>
	.stl-backdrop {
		position: fixed;
		inset: 0;
		background: rgba(0, 0, 0, 0.7);
		z-index: 70;
		animation: fadeIn 0.3s ease;
	}

	.stl-modal {
		position: fixed;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		width: min(800px, 92vw);
		height: min(600px, 80vh);
		background: rgba(10, 8, 28, 0.95);
		border: 1px solid rgba(187, 68, 255, 0.25);
		border-top: 2px solid #bb44ff;
		backdrop-filter: blur(24px);
		z-index: 80;
		display: flex;
		flex-direction: column;
		animation: modalIn 0.4s ease;
		pointer-events: all;
	}

	@keyframes fadeIn {
		from { opacity: 0; }
		to { opacity: 1; }
	}

	@keyframes modalIn {
		from { opacity: 0; transform: translate(-50%, -50%) scale(0.92); }
		to { opacity: 1; transform: translate(-50%, -50%) scale(1); }
	}

	.stl-close {
		position: absolute;
		top: 10px;
		right: 14px;
		background: none;
		border: none;
		color: rgba(255, 255, 255, 0.4);
		font-size: 1.6rem;
		cursor: pointer;
		z-index: 5;
		transition: color 0.2s;
	}

	.stl-close:hover {
		color: #bb44ff;
	}

	.stl-header {
		padding: 16px 24px 8px;
		text-align: center;
	}

	.stl-title {
		font-size: 1.1rem;
		font-weight: 300;
		letter-spacing: 0.35em;
		color: #bb44ff;
		margin: 0;
		text-shadow: 0 0 20px rgba(187, 68, 255, 0.35);
	}

	.stl-subtitle {
		font-size: 0.68rem;
		letter-spacing: 0.1em;
		color: rgba(255, 255, 255, 0.35);
		margin: 6px 0 0;
	}

	.stl-container {
		flex: 1;
		margin: 8px 12px 12px;
		border-radius: 2px;
		overflow: hidden;
	}

	@media (max-width: 768px) {
		.stl-modal {
			width: 96vw;
			height: 70vh;
		}
	}
</style>
