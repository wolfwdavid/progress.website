<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { STLLoader } from 'three/addons/loaders/STLLoader.js';
	import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

	let { url, onclose }: { url: string; onclose?: () => void } = $props();

	let container: HTMLDivElement;
	let dimensions = $state<{ mm: { w: number; h: number; d: number }; in: { w: number; h: number; d: number } } | null>(null);
	let unitSystem = $state<'SI' | 'Imperial'>('SI');
	let rulerActive = $state(false);
	let rulerDistance = $state<{ mm: number; in: number } | null>(null);
	let rulerStep = $state<0 | 1 | 2>(0); // 0 = no points, 1 = point A placed, 2 = both placed

	onMount(() => {
		const width = container.clientWidth;
		const height = container.clientHeight;

		const scene = new THREE.Scene();
		scene.background = new THREE.Color(0x0a081c);

		const camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 2000);
		camera.position.set(0, 50, 150);

		const renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(width, height);
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
		renderer.toneMapping = THREE.ACESFilmicToneMapping;
		renderer.toneMappingExposure = 1.2;
		container.appendChild(renderer.domElement);

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

		const grid = new THREE.GridHelper(200, 20, 0x333355, 0x1a1a33);
		scene.add(grid);

		// Ruler state
		const raycaster = new THREE.Raycaster();
		const mouse = new THREE.Vector2();
		let modelMesh: THREE.Mesh | null = null;
		let modelScale = 1;

		// Ruler visual objects
		const rulerGroup = new THREE.Group();
		scene.add(rulerGroup);

		const pointMarkerGeo = new THREE.SphereGeometry(1.2, 16, 16);
		const pointMarkerMatA = new THREE.MeshBasicMaterial({ color: 0x00ffaa });
		const pointMarkerMatB = new THREE.MeshBasicMaterial({ color: 0xff6644 });
		const markerA = new THREE.Mesh(pointMarkerGeo, pointMarkerMatA);
		const markerB = new THREE.Mesh(pointMarkerGeo, pointMarkerMatB);
		markerA.visible = false;
		markerB.visible = false;
		rulerGroup.add(markerA, markerB);

		const lineMat = new THREE.LineBasicMaterial({ color: 0xffff00, linewidth: 2 });
		const lineGeo = new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(), new THREE.Vector3()]);
		const rulerLine = new THREE.Line(lineGeo, lineMat);
		rulerLine.visible = false;
		rulerGroup.add(rulerLine);

		let pointA = new THREE.Vector3();
		let pointB = new THREE.Vector3();
		// Store real-world (pre-scale) points for distance calc
		let realA = new THREE.Vector3();
		let realB = new THREE.Vector3();

		function onRulerClick(e: MouseEvent) {
			if (!rulerActive || !modelMesh) return;

			const rect = renderer.domElement.getBoundingClientRect();
			mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
			mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;

			raycaster.setFromCamera(mouse, camera);
			const intersects = raycaster.intersectObject(modelMesh);
			if (intersects.length === 0) return;

			const hitPoint = intersects[0].point.clone();
			// Convert back to real-world mm by dividing by model scale
			const realPoint = hitPoint.clone().divideScalar(modelScale);

			if (rulerStep === 0 || rulerStep === 2) {
				// Place point A (reset if both were placed)
				pointA.copy(hitPoint);
				realA.copy(realPoint);
				markerA.position.copy(hitPoint);
				markerA.visible = true;
				markerB.visible = false;
				rulerLine.visible = false;
				rulerDistance = null;
				rulerStep = 1;
			} else if (rulerStep === 1) {
				// Place point B
				pointB.copy(hitPoint);
				realB.copy(realPoint);
				markerB.position.copy(hitPoint);
				markerB.visible = true;

				// Update line
				const positions = rulerLine.geometry.attributes.position as THREE.BufferAttribute;
				positions.setXYZ(0, pointA.x, pointA.y, pointA.z);
				positions.setXYZ(1, pointB.x, pointB.y, pointB.z);
				positions.needsUpdate = true;
				rulerLine.visible = true;

				// Calculate real-world distance in mm
				const distMm = realA.distanceTo(realB);
				rulerDistance = {
					mm: parseFloat(distMm.toFixed(1)),
					in: parseFloat((distMm / 25.4).toFixed(2))
				};
				rulerStep = 2;
			}
		}

		renderer.domElement.addEventListener('click', onRulerClick);

		// Load STL
		const loader = new STLLoader();
		loader.load(url, (geometry) => {
			geometry.rotateX(-Math.PI / 2);
			geometry.computeVertexNormals();
			geometry.computeBoundingBox();

			const box = geometry.boundingBox!;
			const size = new THREE.Vector3();
			box.getSize(size);

			const mmW = parseFloat(size.x.toFixed(1));
			const mmH = parseFloat(size.y.toFixed(1));
			const mmD = parseFloat(size.z.toFixed(1));
			dimensions = {
				mm: { w: mmW, h: mmH, d: mmD },
				in: {
					w: parseFloat((mmW / 25.4).toFixed(2)),
					h: parseFloat((mmH / 25.4).toFixed(2)),
					d: parseFloat((mmD / 25.4).toFixed(2))
				}
			};

			const material = new THREE.MeshPhysicalMaterial({
				color: 0x8855cc,
				metalness: 0.3,
				roughness: 0.4,
				clearcoat: 0.3,
				clearcoatRoughness: 0.25,
				envMapIntensity: 0.5
			});

			const mesh = new THREE.Mesh(geometry, material);

			const center = new THREE.Vector3();
			box.getCenter(center);
			mesh.position.sub(center);
			mesh.position.y += size.y / 2;

			const maxDim = Math.max(size.x, size.y, size.z);
			const scale = 80 / maxDim;
			modelScale = scale;
			mesh.scale.setScalar(scale);
			mesh.position.multiplyScalar(scale);

			modelMesh = mesh;
			scene.add(mesh);

			const dist = maxDim * scale * 1.6;
			camera.position.set(dist * 0.6, dist * 0.5, dist * 0.8);
			controls.target.set(0, size.y * scale * 0.35, 0);
			controls.update();
		});

		let animId: number;
		function animate() {
			animId = requestAnimationFrame(animate);
			controls.update();
			renderer.render(scene, camera);
		}
		animate();

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
			renderer.domElement.removeEventListener('click', onRulerClick);
			renderer.dispose();
			controls.dispose();
		};
	});

	function toggleRuler() {
		rulerActive = !rulerActive;
		if (!rulerActive) {
			rulerStep = 0;
			rulerDistance = null;
		}
	}

	function clearRuler() {
		rulerStep = 0;
		rulerDistance = null;
	}
</script>

<div class="stl-backdrop" onclick={onclose} role="presentation"></div>
<div class="stl-modal">
	<button class="stl-close" onclick={onclose}>&times;</button>
	<div class="stl-header">
		<h2 class="stl-title">EATING TOOL</h2>
		<p class="stl-subtitle">Interactive 3D Model &mdash; Drag to rotate, scroll to zoom</p>
	</div>
	<div class="stl-container" bind:this={container}>
		{#if rulerActive}
			<div class="ruler-hint">
				{#if rulerStep === 0}
					Click a point on the model to start measuring
				{:else if rulerStep === 1}
					Click a second point to complete measurement
				{:else}
					Measurement complete &mdash; click to start a new one
				{/if}
			</div>
		{/if}
	</div>

	<div class="stl-toolbar">
		<div class="toolbar-left">
			<div class="dim-toggle">
				<button class="dim-btn" class:active={unitSystem === 'SI'} onclick={() => (unitSystem = 'SI')}>SI (mm)</button>
				<button class="dim-btn" class:active={unitSystem === 'Imperial'} onclick={() => (unitSystem = 'Imperial')}>Imperial (in)</button>
			</div>
			<button class="ruler-btn" class:active={rulerActive} onclick={toggleRuler}>
				<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
					<path d="M2 22L22 2M5.5 18.5L8 16M9.5 14.5L12 12M13.5 10.5L16 8M17.5 6.5L20 4" />
				</svg>
				Ruler
			</button>
			{#if rulerActive && rulerStep === 2}
				<button class="ruler-clear" onclick={clearRuler}>Clear</button>
			{/if}
		</div>

		<div class="toolbar-right">
			{#if rulerDistance}
				<div class="ruler-result">
					<span class="ruler-result-label">Distance:</span>
					<span class="ruler-result-val">
						{#if unitSystem === 'SI'}
							{rulerDistance.mm} mm
						{:else}
							{rulerDistance.in}"
						{/if}
					</span>
				</div>
			{/if}

			{#if dimensions}
				<div class="dim-values">
					{#if unitSystem === 'SI'}
						<div class="dim-item"><span class="dim-label">W</span><span class="dim-val">{dimensions.mm.w} mm</span></div>
						<div class="dim-item"><span class="dim-label">H</span><span class="dim-val">{dimensions.mm.h} mm</span></div>
						<div class="dim-item"><span class="dim-label">D</span><span class="dim-val">{dimensions.mm.d} mm</span></div>
					{:else}
						<div class="dim-item"><span class="dim-label">W</span><span class="dim-val">{dimensions.in.w}"</span></div>
						<div class="dim-item"><span class="dim-label">H</span><span class="dim-val">{dimensions.in.h}"</span></div>
						<div class="dim-item"><span class="dim-label">D</span><span class="dim-val">{dimensions.in.d}"</span></div>
					{/if}
				</div>
			{/if}
		</div>
	</div>
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
		margin: 8px 12px 0;
		border-radius: 2px;
		overflow: hidden;
		position: relative;
	}

	/* Ruler hint overlay */
	.ruler-hint {
		position: absolute;
		top: 10px;
		left: 50%;
		transform: translateX(-50%);
		background: rgba(0, 0, 0, 0.7);
		border: 1px solid rgba(255, 255, 0, 0.3);
		padding: 5px 14px;
		font-size: 0.65rem;
		letter-spacing: 0.08em;
		color: rgba(255, 255, 100, 0.8);
		white-space: nowrap;
		z-index: 2;
		pointer-events: none;
	}

	/* Toolbar */
	.stl-toolbar {
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 8px 16px;
		margin: 0 12px 10px;
		background: rgba(187, 68, 255, 0.05);
		border: 1px solid rgba(187, 68, 255, 0.12);
		border-radius: 2px;
		gap: 12px;
		flex-wrap: wrap;
	}

	.toolbar-left {
		display: flex;
		align-items: center;
		gap: 6px;
	}

	.toolbar-right {
		display: flex;
		align-items: center;
		gap: 16px;
	}

	.dim-toggle {
		display: flex;
		gap: 2px;
	}

	.dim-btn {
		background: rgba(255, 255, 255, 0.04);
		border: 1px solid rgba(255, 255, 255, 0.08);
		color: rgba(255, 255, 255, 0.45);
		padding: 3px 10px;
		font-size: 0.62rem;
		font-weight: 500;
		letter-spacing: 0.08em;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.dim-btn.active {
		background: rgba(187, 68, 255, 0.15);
		border-color: rgba(187, 68, 255, 0.4);
		color: #bb44ff;
	}

	.dim-btn:hover {
		border-color: rgba(187, 68, 255, 0.3);
		color: rgba(255, 255, 255, 0.7);
	}

	.ruler-btn {
		display: flex;
		align-items: center;
		gap: 5px;
		background: rgba(255, 255, 255, 0.04);
		border: 1px solid rgba(255, 255, 255, 0.08);
		color: rgba(255, 255, 255, 0.45);
		padding: 3px 10px;
		font-size: 0.62rem;
		font-weight: 500;
		letter-spacing: 0.08em;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.ruler-btn.active {
		background: rgba(255, 255, 0, 0.1);
		border-color: rgba(255, 255, 0, 0.4);
		color: #ffff66;
	}

	.ruler-btn:hover {
		border-color: rgba(255, 255, 0, 0.3);
		color: rgba(255, 255, 255, 0.7);
	}

	.ruler-clear {
		background: rgba(255, 100, 68, 0.08);
		border: 1px solid rgba(255, 100, 68, 0.25);
		color: rgba(255, 100, 68, 0.7);
		padding: 3px 10px;
		font-size: 0.62rem;
		font-weight: 500;
		letter-spacing: 0.08em;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.ruler-clear:hover {
		background: rgba(255, 100, 68, 0.15);
		border-color: rgba(255, 100, 68, 0.5);
		color: #ff6644;
	}

	.ruler-result {
		display: flex;
		align-items: center;
		gap: 6px;
		padding: 2px 10px;
		background: rgba(255, 255, 0, 0.06);
		border: 1px solid rgba(255, 255, 0, 0.2);
		border-radius: 2px;
	}

	.ruler-result-label {
		font-size: 0.6rem;
		font-weight: 600;
		letter-spacing: 0.1em;
		color: rgba(255, 255, 100, 0.6);
	}

	.ruler-result-val {
		font-size: 0.75rem;
		font-weight: 500;
		letter-spacing: 0.06em;
		color: #ffff66;
		font-variant-numeric: tabular-nums;
	}

	.dim-values {
		display: flex;
		gap: 16px;
	}

	.dim-item {
		display: flex;
		align-items: center;
		gap: 6px;
	}

	.dim-label {
		font-size: 0.6rem;
		font-weight: 600;
		letter-spacing: 0.1em;
		color: rgba(187, 68, 255, 0.6);
	}

	.dim-val {
		font-size: 0.72rem;
		font-weight: 400;
		letter-spacing: 0.06em;
		color: rgba(255, 255, 255, 0.7);
		font-variant-numeric: tabular-nums;
	}

	@media (max-width: 768px) {
		.stl-modal {
			width: 96vw;
			height: 70vh;
		}
		.stl-toolbar {
			flex-direction: column;
			align-items: flex-start;
		}
		.toolbar-right {
			flex-wrap: wrap;
		}
	}
</style>
