<script lang="ts">
	import { onMount } from 'svelte';

	// ═══════════════════════════════════════════
	// TYPES
	// ═══════════════════════════════════════════

	interface CNode {
		id: number;
		x: number;
		y: number;
		z: number;
		type: 'core' | 'large' | 'medium' | 'small';
		role: string;
		label: string;
		description: string;
		active: boolean;
		connections: number[];
	}

	interface Particle {
		x: number;
		y: number;
		z: number;
		vx: number;
		vy: number;
		vz: number;
		size: number;
		alpha: number;
		color: string;
		life: number;
		maxLife: number;
	}

	interface Star {
		x: number;
		y: number;
		size: number;
		brightness: number;
		twinkleSpeed: number;
		twinklePhase: number;
	}

	interface EnergyPulse {
		connIdx: number;
		t: number;
		speed: number;
		color: string;
	}

	interface Projected {
		id: number;
		sx: number;
		sy: number;
		scale: number;
		depth: number;
	}

	// ═══════════════════════════════════════════
	// PROPS
	// ═══════════════════════════════════════════

	let {
		onselect,
		activeRole
	}: {
		onselect?: (node: CNode | null) => void;
		activeRole?: string;
	} = $props();

	// ═══════════════════════════════════════════
	// CONSTANTS
	// ═══════════════════════════════════════════

	const ROLES: Record<string, { color: string; glow: string; name: string }> = {
		commando: { color: '#ff3333', glow: '#ff6b6b', name: 'Commando' },
		ravager: { color: '#33aaff', glow: '#66ccff', name: 'Ravager' },
		sentinel: { color: '#ffd700', glow: '#ffe44d', name: 'Sentinel' },
		medic: { color: '#33ff88', glow: '#66ffaa', name: 'Medic' },
		synergist: { color: '#bb44ff', glow: '#dd77ff', name: 'Synergist' },
		saboteur: { color: '#ff44aa', glow: '#ff77cc', name: 'Saboteur' }
	};

	const ROLE_ABILITIES: Record<string, string[]> = {
		commando: [
			'Attack',
			'Ruin',
			'Blitz',
			'Launch',
			'Ruinga',
			'Smite',
			'Scourge',
			'Adrenaline',
			'Blindside'
		],
		ravager: [
			'Fire',
			'Thunder',
			'Blizzard',
			'Aero',
			'Fira',
			'Thundara',
			'Blizzara',
			'Aerora',
			'Firaga'
		],
		sentinel: [
			'Provoke',
			'Steelguard',
			'Vendetta',
			'Entrench',
			'Mediguard',
			'Deathward',
			'Elude',
			'Fringeward'
		],
		medic: ['Cure', 'Cura', 'Esuna', 'Raise', 'Curasa', 'Curaja', 'Cheer', 'Reraise'],
		synergist: [
			'Haste',
			'Protect',
			'Shell',
			'Bravery',
			'Faith',
			'Veil',
			'Vigilance',
			'Enfire'
		],
		saboteur: [
			'Deprotect',
			'Deshell',
			'Slow',
			'Poison',
			'Imperil',
			'Curse',
			'Dispel',
			'Fog',
			'Pain'
		]
	};

	const STAT_LABELS = [
		'HP +100',
		'HP +200',
		'HP +300',
		'STR +4',
		'STR +6',
		'STR +8',
		'MAG +4',
		'MAG +6',
		'MAG +8',
		'ATB +1'
	];

	// ═══════════════════════════════════════════
	// STATE
	// ═══════════════════════════════════════════

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let animId: number;
	let bgCanvas: HTMLCanvasElement;
	let bgCtx: CanvasRenderingContext2D;

	let nodes: CNode[] = [];
	let connections: [number, number][] = [];
	let particles: Particle[] = [];
	let stars: Star[] = [];
	let pulses: EnergyPulse[] = [];

	let W = 0;
	let H = 0;

	// Camera
	let rotX = -0.25;
	let rotY = 0;
	let zoom = 1;
	let autoRotating = true;
	let autoRotateTimer: ReturnType<typeof setTimeout>;

	// Mouse
	let dragging = false;
	let dragSX = 0;
	let dragSY = 0;
	let dragRX = 0;
	let dragRY = 0;

	let selectedNode = $state<CNode | null>(null);
	let hoveredNode: CNode | null = null;

	// Time
	let time = 0;
	let lastTs = 0;

	// Seeded random for deterministic generation
	let seed = 42;
	function seededRandom(): number {
		seed = (seed * 16807 + 0) % 2147483647;
		return (seed - 1) / 2147483646;
	}

	// ═══════════════════════════════════════════
	// GENERATION
	// ═══════════════════════════════════════════

	function generateCrystarium(): void {
		nodes = [];
		connections = [];
		seed = 42;
		let nid = 0;

		// Core node
		nodes.push({
			id: nid++,
			x: 0,
			y: 0,
			z: 0,
			type: 'core',
			role: 'core',
			label: 'L\'Cie Crystal',
			description: 'The heart of the Crystarium. All paths of destiny begin here.',
			active: true,
			connections: []
		});

		const roleKeys = Object.keys(ROLES);

		for (let ri = 0; ri < roleKeys.length; ri++) {
			const role = roleKeys[ri];
			const abilities = [...ROLE_ABILITIES[role]];
			const baseAngle = (ri / roleKeys.length) * Math.PI * 2;
			let abilIdx = 0;

			// Role crystal near center
			const rcId = nid;
			const rcR = 55;
			nodes.push({
				id: nid++,
				x: rcR * Math.cos(baseAngle),
				y: (seededRandom() - 0.5) * 20,
				z: rcR * Math.sin(baseAngle),
				type: 'large',
				role,
				label: ROLES[role].name,
				description: `The path of the ${ROLES[role].name}. Enter to unlock new powers.`,
				active: false,
				connections: []
			});
			connections.push([0, rcId]);

			let prevId = rcId;

			// Main branch: 16 nodes spiraling outward
			const mainLen = 16;
			for (let i = 0; i < mainLen; i++) {
				const t = (i + 1) / mainLen;
				const r = 80 + t * 300;
				const angle = baseAngle + t * Math.PI * 1.4;
				const y =
					Math.sin(t * Math.PI * 2.5 + ri * 0.5) * 90 +
					Math.cos(t * Math.PI * 1.7) * 35;

				let type: CNode['type'] = 'small';
				let label = STAT_LABELS[Math.floor(seededRandom() * STAT_LABELS.length)];
				let desc = `Enhances ${label.split(' ')[0]} capacity.`;

				// Ability nodes at regular intervals
				if (i % 4 === 3 && abilIdx < abilities.length) {
					type = 'medium';
					label = abilities[abilIdx++];
					desc = `Learn the ability: ${label}.`;
				}
				// Grand node at the end
				if (i === mainLen - 1) {
					type = 'large';
					if (abilIdx < abilities.length) {
						label = abilities[abilIdx++];
						desc = `Master technique: ${label}. Crowning achievement of this path.`;
					}
				}

				const id = nid++;
				nodes.push({
					id,
					x: r * Math.cos(angle) + (seededRandom() - 0.5) * 15,
					y: y + (seededRandom() - 0.5) * 10,
					z: r * Math.sin(angle) + (seededRandom() - 0.5) * 15,
					type,
					role,
					label,
					description: desc,
					active: false,
					connections: []
				});
				connections.push([prevId, id]);
				prevId = id;

				// Sub-branches at nodes 4, 9, 13
				if (i === 4 || i === 9 || i === 13) {
					const subLen = 3 + Math.floor(seededRandom() * 3);
					let subPrev = id;
					const forkDir = i === 4 ? -1 : i === 9 ? 1 : -1;

					for (let j = 0; j < subLen; j++) {
						const st = (j + 1) / subLen;
						const sr = r + st * 70;
						const sa = angle + forkDir * (0.2 + st * 0.35);
						const sy = y + Math.sin(st * Math.PI) * 50 * forkDir;

						let sType: CNode['type'] = 'small';
						let sLabel = STAT_LABELS[Math.floor(seededRandom() * STAT_LABELS.length)];
						let sDesc = `Enhances ${sLabel.split(' ')[0]} capacity.`;

						if (j === subLen - 1 && abilIdx < abilities.length) {
							sType = 'medium';
							sLabel = abilities[abilIdx++];
							sDesc = `Learn the ability: ${sLabel}.`;
						}

						const sid = nid++;
						nodes.push({
							id: sid,
							x: sr * Math.cos(sa) + (seededRandom() - 0.5) * 10,
							y: sy + (seededRandom() - 0.5) * 8,
							z: sr * Math.sin(sa) + (seededRandom() - 0.5) * 10,
							type: sType,
							role,
							label: sLabel,
							description: sDesc,
							active: false,
							connections: []
						});
						connections.push([subPrev, sid]);
						subPrev = sid;
					}
				}
			}
		}

		// Build adjacency
		for (const [a, b] of connections) {
			nodes[a].connections.push(b);
			nodes[b].connections.push(a);
		}
	}

	// ═══════════════════════════════════════════
	// 3D MATH
	// ═══════════════════════════════════════════

	function rotatePoint(px: number, py: number, pz: number): [number, number, number] {
		const cY = Math.cos(rotY),
			sY = Math.sin(rotY);
		let rx = px * cY - pz * sY;
		let rz = px * sY + pz * cY;
		const cX = Math.cos(rotX),
			sX = Math.sin(rotX);
		const ry = py * cX - rz * sX;
		rz = py * sX + rz * cX;
		return [rx, ry, rz];
	}

	function project(px: number, py: number, pz: number): { sx: number; sy: number; scale: number; depth: number } {
		const [rx, ry, rz] = rotatePoint(px, py, pz);
		const fov = 800;
		const viewDist = 600;
		const d = rz + viewDist;
		const scale = (fov / Math.max(d, 50)) * zoom;
		return {
			sx: rx * scale + W / 2,
			sy: ry * scale + H / 2,
			scale,
			depth: rz
		};
	}

	// ═══════════════════════════════════════════
	// STARS & PARTICLES
	// ═══════════════════════════════════════════

	function generateStars(): void {
		stars = [];
		for (let i = 0; i < 700; i++) {
			stars.push({
				x: Math.random(),
				y: Math.random(),
				size: Math.random() * 1.8 + 0.3,
				brightness: Math.random() * 0.6 + 0.4,
				twinkleSpeed: Math.random() * 3 + 0.5,
				twinklePhase: Math.random() * Math.PI * 2
			});
		}
	}

	function createAmbientParticle(): Particle {
		const a = Math.random() * Math.PI * 2;
		const r = Math.random() * 420;
		const colors = ['#aaccff', '#ccddff', '#eeeeff', '#ffccff', '#ccffee', '#ddccff'];
		return {
			x: r * Math.cos(a),
			y: (Math.random() - 0.5) * 400,
			z: r * Math.sin(a),
			vx: (Math.random() - 0.5) * 0.2,
			vy: (Math.random() - 0.5) * 0.15,
			vz: (Math.random() - 0.5) * 0.2,
			size: Math.random() * 2 + 0.5,
			alpha: Math.random() * 0.4 + 0.1,
			color: colors[Math.floor(Math.random() * colors.length)],
			life: Math.random() * 800,
			maxLife: 600 + Math.random() * 800
		};
	}

	function initParticles(): void {
		particles = [];
		for (let i = 0; i < 180; i++) {
			particles.push(createAmbientParticle());
		}
	}

	function updateParticles(dt: number): void {
		for (let i = 0; i < particles.length; i++) {
			const p = particles[i];
			p.x += p.vx * dt;
			p.y += p.vy * dt;
			p.z += p.vz * dt;
			p.life += dt;
			if (p.life > p.maxLife) {
				particles[i] = createAmbientParticle();
			}
		}
	}

	// ═══════════════════════════════════════════
	// ENERGY PULSES
	// ═══════════════════════════════════════════

	function updatePulses(dt: number): void {
		// Spawn
		if (Math.random() < 0.03 * dt) {
			const activeConns: number[] = [];
			for (let i = 0; i < connections.length; i++) {
				const [a, b] = connections[i];
				if (nodes[a].active && nodes[b].active) activeConns.push(i);
			}
			if (activeConns.length > 0) {
				const ci = activeConns[Math.floor(Math.random() * activeConns.length)];
				const [, b] = connections[ci];
				const role = nodes[b].role === 'core' ? 'commando' : nodes[b].role;
				pulses.push({
					connIdx: ci,
					t: 0,
					speed: 0.008 + Math.random() * 0.012,
					color: ROLES[role]?.glow || '#ffffff'
				});
			}
		}

		pulses = pulses.filter((p) => {
			p.t += p.speed * dt;
			return p.t <= 1;
		});
	}

	// ═══════════════════════════════════════════
	// BACKGROUND
	// ═══════════════════════════════════════════

	function renderBackground(): void {
		bgCanvas = document.createElement('canvas');
		bgCanvas.width = W;
		bgCanvas.height = H;
		bgCtx = bgCanvas.getContext('2d')!;

		// Deep cosmic gradient
		const grad = bgCtx.createRadialGradient(W / 2, H / 2, 0, W / 2, H / 2, Math.max(W, H) * 0.8);
		grad.addColorStop(0, '#0c0a1e');
		grad.addColorStop(0.4, '#06061a');
		grad.addColorStop(0.7, '#030312');
		grad.addColorStop(1, '#000008');
		bgCtx.fillStyle = grad;
		bgCtx.fillRect(0, 0, W, H);

		// Nebula clouds
		const nebulae = [
			{ x: W * 0.25, y: H * 0.35, r: W * 0.35, c: [90, 20, 160] },
			{ x: W * 0.75, y: H * 0.3, r: W * 0.3, c: [20, 70, 160] },
			{ x: W * 0.5, y: H * 0.7, r: W * 0.4, c: [20, 110, 130] },
			{ x: W * 0.15, y: H * 0.75, r: W * 0.25, c: [130, 20, 90] },
			{ x: W * 0.85, y: H * 0.65, r: W * 0.3, c: [50, 20, 120] },
			{ x: W * 0.5, y: H * 0.15, r: W * 0.2, c: [30, 50, 140] },
			{ x: W * 0.4, y: H * 0.5, r: W * 0.5, c: [60, 10, 100] }
		];

		for (const n of nebulae) {
			const ng = bgCtx.createRadialGradient(n.x, n.y, 0, n.x, n.y, n.r);
			ng.addColorStop(0, `rgba(${n.c[0]},${n.c[1]},${n.c[2]},0.07)`);
			ng.addColorStop(0.4, `rgba(${n.c[0]},${n.c[1]},${n.c[2]},0.035)`);
			ng.addColorStop(1, 'rgba(0,0,0,0)');
			bgCtx.fillStyle = ng;
			bgCtx.fillRect(0, 0, W, H);
		}

		// Vignette
		const vig = bgCtx.createRadialGradient(W / 2, H / 2, W * 0.25, W / 2, H / 2, Math.max(W, H) * 0.7);
		vig.addColorStop(0, 'rgba(0,0,0,0)');
		vig.addColorStop(1, 'rgba(0,0,0,0.5)');
		bgCtx.fillStyle = vig;
		bgCtx.fillRect(0, 0, W, H);
	}

	// ═══════════════════════════════════════════
	// DRAW FUNCTIONS
	// ═══════════════════════════════════════════

	function drawStars(): void {
		for (const s of stars) {
			const twinkle = Math.sin(time * s.twinkleSpeed + s.twinklePhase) * 0.3 + 0.7;
			ctx.globalAlpha = s.brightness * twinkle;
			ctx.fillStyle = '#fff';
			ctx.beginPath();
			ctx.arc(s.x * W, s.y * H, s.size, 0, Math.PI * 2);
			ctx.fill();
		}
		ctx.globalAlpha = 1;
	}

	function drawParticles(): void {
		ctx.globalCompositeOperation = 'lighter';
		for (const p of particles) {
			const pr = project(p.x, p.y, p.z);
			const lifeRatio = 1 - Math.abs((2 * p.life) / p.maxLife - 1);
			const a = p.alpha * lifeRatio;
			if (a <= 0.01) continue;
			ctx.globalAlpha = a;
			ctx.fillStyle = p.color;
			const sz = Math.max(p.size * pr.scale * 0.5, 0.5);
			ctx.beginPath();
			ctx.arc(pr.sx, pr.sy, sz, 0, Math.PI * 2);
			ctx.fill();
		}
		ctx.globalCompositeOperation = 'source-over';
		ctx.globalAlpha = 1;
	}

	function getNodeRadius(type: string, scale: number, active: boolean): number {
		let base: number;
		switch (type) {
			case 'core':
				base = 16;
				break;
			case 'large':
				base = 10;
				break;
			case 'medium':
				base = 7;
				break;
			default:
				base = 4.5;
				break;
		}
		return base * scale * (active ? 1 : 0.65);
	}

	function getRoleColor(role: string): string {
		if (role === 'core') return '#ffffff';
		return ROLES[role]?.color || '#888888';
	}

	function getRoleGlow(role: string): string {
		if (role === 'core') return '#ffffdd';
		return ROLES[role]?.glow || '#aaaaaa';
	}

	function shouldDim(role: string): boolean {
		if (!activeRole || activeRole === 'all') return false;
		if (role === 'core') return false;
		return role !== activeRole;
	}

	function drawConnections(projected: Projected[]): void {
		for (let ci = 0; ci < connections.length; ci++) {
			const [ai, bi] = connections[ci];
			const a = projected[ai];
			const b = projected[bi];
			const na = nodes[ai];
			const nb = nodes[bi];

			const bothActive = na.active && nb.active;
			const eitherActive = na.active || nb.active;
			const role = nb.role === 'core' ? na.role : nb.role;
			const color = getRoleColor(role);
			const dimmed = shouldDim(role);
			const dimMult = dimmed ? 0.15 : 1;

			// Glow layer
			if (eitherActive && !dimmed) {
				ctx.globalCompositeOperation = 'lighter';
				ctx.strokeStyle = color;
				ctx.globalAlpha = (bothActive ? 0.12 : 0.04) * dimMult;
				ctx.lineWidth = bothActive ? 10 : 5;
				ctx.beginPath();
				ctx.moveTo(a.sx, a.sy);
				ctx.lineTo(b.sx, b.sy);
				ctx.stroke();
				ctx.globalCompositeOperation = 'source-over';
			}

			// Main line
			ctx.globalAlpha = (bothActive ? 0.65 : eitherActive ? 0.25 : 0.06) * dimMult;
			ctx.strokeStyle = color;
			ctx.lineWidth = bothActive ? 2 : 1;
			ctx.beginPath();
			ctx.moveTo(a.sx, a.sy);
			ctx.lineTo(b.sx, b.sy);
			ctx.stroke();
		}
		ctx.globalAlpha = 1;
	}

	function drawPulses(projected: Projected[]): void {
		ctx.globalCompositeOperation = 'lighter';
		for (const pulse of pulses) {
			const [ai, bi] = connections[pulse.connIdx];
			const a = projected[ai];
			const b = projected[bi];

			const px = a.sx + (b.sx - a.sx) * pulse.t;
			const py = a.sy + (b.sy - a.sy) * pulse.t;

			const pg = ctx.createRadialGradient(px, py, 0, px, py, 8);
			pg.addColorStop(0, pulse.color);
			pg.addColorStop(0.5, pulse.color + '66');
			pg.addColorStop(1, 'transparent');
			ctx.globalAlpha = 0.7;
			ctx.fillStyle = pg;
			ctx.beginPath();
			ctx.arc(px, py, 8, 0, Math.PI * 2);
			ctx.fill();
		}
		ctx.globalCompositeOperation = 'source-over';
		ctx.globalAlpha = 1;
	}

	function drawNode(node: CNode, p: Projected): void {
		const isSel = selectedNode?.id === node.id;
		const isHov = hoveredNode?.id === node.id;
		const radius = getNodeRadius(node.type, p.scale, node.active);
		const color = getRoleColor(node.role);
		const glow = getRoleGlow(node.role);
		const depthAlpha = Math.max(0.25, Math.min(1, 1 - p.depth / 900));
		const dimmed = shouldDim(node.role);
		const dimMult = dimmed ? 0.12 : 1;

		if (node.active || isHov) {
			const pulse = node.active ? Math.sin(time * 2.5 + node.id * 0.7) * 0.15 + 0.85 : 1;

			// Outer bloom
			ctx.globalCompositeOperation = 'lighter';
			const bloomR = radius * (isSel ? 6 : isHov ? 5 : 3.5);
			const bg = ctx.createRadialGradient(p.sx, p.sy, 0, p.sx, p.sy, bloomR);
			bg.addColorStop(0, glow + '44');
			bg.addColorStop(0.3, color + '22');
			bg.addColorStop(0.6, color + '0a');
			bg.addColorStop(1, 'transparent');
			ctx.globalAlpha = (isSel ? 0.5 : 0.25) * pulse * depthAlpha * dimMult;
			ctx.fillStyle = bg;
			ctx.beginPath();
			ctx.arc(p.sx, p.sy, bloomR, 0, Math.PI * 2);
			ctx.fill();
			ctx.globalCompositeOperation = 'source-over';

			// Node body
			const ng = ctx.createRadialGradient(p.sx, p.sy, 0, p.sx, p.sy, radius);
			ng.addColorStop(0, '#ffffff');
			ng.addColorStop(0.25, glow);
			ng.addColorStop(0.6, color);
			ng.addColorStop(1, color + '88');
			ctx.globalAlpha = pulse * depthAlpha * dimMult;
			ctx.fillStyle = ng;
			ctx.beginPath();
			ctx.arc(p.sx, p.sy, radius, 0, Math.PI * 2);
			ctx.fill();

			// Core crystal extra effects
			if (node.type === 'core') {
				// Lens flare cross
				ctx.globalCompositeOperation = 'lighter';
				ctx.globalAlpha = 0.3 * pulse * depthAlpha;
				const flareLen = radius * 4;
				for (let fi = 0; fi < 4; fi++) {
					const fa = (fi / 4) * Math.PI + time * 0.3;
					const fg = ctx.createLinearGradient(
						p.sx + Math.cos(fa) * flareLen,
						p.sy + Math.sin(fa) * flareLen,
						p.sx - Math.cos(fa) * flareLen,
						p.sy - Math.sin(fa) * flareLen
					);
					fg.addColorStop(0, 'transparent');
					fg.addColorStop(0.45, '#ffffff00');
					fg.addColorStop(0.5, '#ffffffaa');
					fg.addColorStop(0.55, '#ffffff00');
					fg.addColorStop(1, 'transparent');
					ctx.strokeStyle = fg;
					ctx.lineWidth = 1.5;
					ctx.beginPath();
					ctx.moveTo(p.sx + Math.cos(fa) * flareLen, p.sy + Math.sin(fa) * flareLen);
					ctx.lineTo(p.sx - Math.cos(fa) * flareLen, p.sy - Math.sin(fa) * flareLen);
					ctx.stroke();
				}
				ctx.globalCompositeOperation = 'source-over';
			}

			// Selection ring
			if (isSel) {
				ctx.strokeStyle = '#ffffff';
				ctx.globalAlpha = 0.7 * depthAlpha;
				ctx.lineWidth = 1.5;
				const ringR = radius + 5 + Math.sin(time * 3) * 2;
				ctx.beginPath();
				ctx.arc(p.sx, p.sy, ringR, 0, Math.PI * 2);
				ctx.stroke();

				// Orbiting dot
				const orbAngle = time * 2;
				const orbR = ringR + 3;
				ctx.fillStyle = glow;
				ctx.globalAlpha = 0.9 * depthAlpha;
				ctx.beginPath();
				ctx.arc(
					p.sx + Math.cos(orbAngle) * orbR,
					p.sy + Math.sin(orbAngle) * orbR,
					2,
					0,
					Math.PI * 2
				);
				ctx.fill();
			}
		} else {
			// Inactive node - dim crystal
			const canAct = canActivate(node);
			const inactiveAlpha = canAct ? 0.35 : 0.15;

			ctx.globalAlpha = inactiveAlpha * depthAlpha * dimMult;
			const ig = ctx.createRadialGradient(p.sx, p.sy, 0, p.sx, p.sy, radius);
			ig.addColorStop(0, color + 'cc');
			ig.addColorStop(0.5, color + '44');
			ig.addColorStop(1, color + '08');
			ctx.fillStyle = ig;
			ctx.beginPath();
			ctx.arc(p.sx, p.sy, radius, 0, Math.PI * 2);
			ctx.fill();

			// Soft ring on activatable nodes
			if (canAct && !dimmed) {
				ctx.globalAlpha = (0.15 + Math.sin(time * 2) * 0.08) * depthAlpha;
				ctx.strokeStyle = color;
				ctx.lineWidth = 1;
				ctx.beginPath();
				ctx.arc(p.sx, p.sy, radius + 3, 0, Math.PI * 2);
				ctx.stroke();
			}
		}

		ctx.globalAlpha = 1;
	}

	// ═══════════════════════════════════════════
	// INTERACTION
	// ═══════════════════════════════════════════

	function getNodeAt(mx: number, my: number): CNode | null {
		let best: CNode | null = null;
		let bestD = Infinity;

		for (const node of nodes) {
			const p = project(node.x, node.y, node.z);
			const r = Math.max(getNodeRadius(node.type, p.scale, node.active) * 1.5, 14);
			const dx = mx - p.sx;
			const dy = my - p.sy;
			const d = Math.sqrt(dx * dx + dy * dy);
			if (d < r && d < bestD) {
				best = node;
				bestD = d;
			}
		}
		return best;
	}

	function canActivate(node: CNode): boolean {
		if (node.active) return false;
		return node.connections.some((id) => nodes[id].active);
	}

	function activateNode(node: CNode): void {
		if (!canActivate(node)) return;
		node.active = true;

		// Burst particles
		const color = getRoleGlow(node.role);
		for (let i = 0; i < 25; i++) {
			const a = Math.random() * Math.PI * 2;
			const elev = (Math.random() - 0.5) * Math.PI;
			const spd = 1.5 + Math.random() * 3;
			particles.push({
				x: node.x,
				y: node.y,
				z: node.z,
				vx: Math.cos(a) * Math.cos(elev) * spd,
				vy: Math.sin(elev) * spd,
				vz: Math.sin(a) * Math.cos(elev) * spd,
				size: 1 + Math.random() * 2.5,
				alpha: 0.9,
				color,
				life: 0,
				maxLife: 150 + Math.random() * 250
			});
		}
	}

	function onMouseDown(e: MouseEvent): void {
		dragging = true;
		autoRotating = false;
		clearTimeout(autoRotateTimer);
		dragSX = e.clientX;
		dragSY = e.clientY;
		dragRX = rotX;
		dragRY = rotY;
	}

	function onMouseMove(e: MouseEvent): void {
		const rect = canvas.getBoundingClientRect();
		const mx = e.clientX - rect.left;
		const my = e.clientY - rect.top;

		if (dragging) {
			const dx = e.clientX - dragSX;
			const dy = e.clientY - dragSY;
			rotY = dragRY + dx * 0.005;
			rotX = Math.max(-1.2, Math.min(1.2, dragRX + dy * 0.005));
		} else {
			hoveredNode = getNodeAt(mx, my);
			canvas.style.cursor = hoveredNode ? 'pointer' : 'grab';
		}
	}

	function onMouseUp(e: MouseEvent): void {
		if (dragging) {
			const dx = Math.abs(e.clientX - dragSX);
			const dy = Math.abs(e.clientY - dragSY);

			if (dx < 4 && dy < 4) {
				// Click
				const rect = canvas.getBoundingClientRect();
				const mx = e.clientX - rect.left;
				const my = e.clientY - rect.top;
				const clicked = getNodeAt(mx, my);

				if (clicked) {
					if (canActivate(clicked)) activateNode(clicked);
					selectedNode = clicked;
					onselect?.(clicked);
				} else {
					selectedNode = null;
					onselect?.(null);
				}
			}
		}

		dragging = false;
		autoRotateTimer = setTimeout(() => {
			autoRotating = true;
		}, 4000);
	}

	function onWheel(e: WheelEvent): void {
		e.preventDefault();
		zoom = Math.max(0.3, Math.min(3.5, zoom - e.deltaY * 0.001));
	}

	// Touch handlers
	let touchStartDist = 0;
	let touchStartZoom = 1;

	function onTouchStart(e: TouchEvent): void {
		if (e.touches.length === 1) {
			dragging = true;
			autoRotating = false;
			clearTimeout(autoRotateTimer);
			dragSX = e.touches[0].clientX;
			dragSY = e.touches[0].clientY;
			dragRX = rotX;
			dragRY = rotY;
		} else if (e.touches.length === 2) {
			const dx = e.touches[0].clientX - e.touches[1].clientX;
			const dy = e.touches[0].clientY - e.touches[1].clientY;
			touchStartDist = Math.sqrt(dx * dx + dy * dy);
			touchStartZoom = zoom;
		}
	}

	function onTouchMove(e: TouchEvent): void {
		e.preventDefault();
		if (e.touches.length === 1 && dragging) {
			const dx = e.touches[0].clientX - dragSX;
			const dy = e.touches[0].clientY - dragSY;
			rotY = dragRY + dx * 0.005;
			rotX = Math.max(-1.2, Math.min(1.2, dragRX + dy * 0.005));
		} else if (e.touches.length === 2) {
			const dx = e.touches[0].clientX - e.touches[1].clientX;
			const dy = e.touches[0].clientY - e.touches[1].clientY;
			const dist = Math.sqrt(dx * dx + dy * dy);
			zoom = Math.max(0.3, Math.min(3.5, touchStartZoom * (dist / touchStartDist)));
		}
	}

	function onTouchEnd(): void {
		dragging = false;
		autoRotateTimer = setTimeout(() => {
			autoRotating = true;
		}, 4000);
	}

	// ═══════════════════════════════════════════
	// MAIN RENDER LOOP
	// ═══════════════════════════════════════════

	function frame(ts: number): void {
		const dt = lastTs ? Math.min((ts - lastTs) / 16.667, 3) : 1;
		lastTs = ts;
		time = ts / 1000;

		// Auto rotate
		if (autoRotating && !dragging) {
			rotY += 0.0012 * dt;
		}

		updateParticles(dt);
		updatePulses(dt);

		// Draw background
		ctx.drawImage(bgCanvas, 0, 0);

		// Stars
		drawStars();

		// Project all nodes
		const projected: Projected[] = nodes.map((n) => {
			const p = project(n.x, n.y, n.z);
			return { id: n.id, sx: p.sx, sy: p.sy, scale: p.scale, depth: p.depth };
		});

		// Sort back to front
		const sorted = [...projected].sort((a, b) => a.depth - b.depth);

		// Connections
		drawConnections(projected);

		// Particles (behind nodes)
		drawParticles();

		// Pulses
		drawPulses(projected);

		// Nodes
		for (const p of sorted) {
			drawNode(nodes[p.id], p);
		}

		animId = requestAnimationFrame(frame);
	}

	// ═══════════════════════════════════════════
	// RESIZE
	// ═══════════════════════════════════════════

	function handleResize(): void {
		W = window.innerWidth;
		H = window.innerHeight;
		canvas.width = W;
		canvas.height = H;
		renderBackground();
	}

	// ═══════════════════════════════════════════
	// LIFECYCLE
	// ═══════════════════════════════════════════

	onMount(() => {
		ctx = canvas.getContext('2d')!;
		W = window.innerWidth;
		H = window.innerHeight;
		canvas.width = W;
		canvas.height = H;

		generateCrystarium();
		generateStars();
		initParticles();
		renderBackground();

		window.addEventListener('resize', handleResize);
		animId = requestAnimationFrame(frame);

		return () => {
			cancelAnimationFrame(animId);
			window.removeEventListener('resize', handleResize);
			clearTimeout(autoRotateTimer);
		};
	});
</script>

<canvas
	bind:this={canvas}
	onmousedown={onMouseDown}
	onmousemove={onMouseMove}
	onmouseup={onMouseUp}
	onmouseleave={() => {
		dragging = false;
		hoveredNode = null;
	}}
	onwheel={onWheel}
	ontouchstart={onTouchStart}
	ontouchmove={onTouchMove}
	ontouchend={onTouchEnd}
	oncontextmenu={(e) => e.preventDefault()}
></canvas>

<style>
	canvas {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		display: block;
		touch-action: none;
	}
</style>
