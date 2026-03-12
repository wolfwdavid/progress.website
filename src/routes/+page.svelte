<script lang="ts">
	import CrystariumCanvas from '$lib/CrystariumCanvas.svelte';
	import STLViewer from '$lib/STLViewer.svelte';
	import { base } from '$app/paths';

	let selectedNode = $state<any>(null);
	let showSTLViewer = $state(false);
	let activeRole = $state('all');
	let showInstructions = $state(true);
	let showSummary = $state<string | null>(null);
	let lastRoleClickTime = 0;
	let lastRoleClicked = '';
	let summaryTimer: ReturnType<typeof setTimeout> | null = null;

	const roles = [
		{ key: 'all', label: 'ALL', color: '#ffffff' },
		{ key: 'commando', label: 'LTC', color: '#ff3333', description: 'Level The Curve — Access. Independence. Dignity. Improving accessibility and independence for people with disabilities.' },
		{ key: 'ravager', label: 'CORP', color: '#33aaff', description: 'LTC Foundation Corp — Empowering individuals without limits. A 501(c)(3) nonprofit removing barriers and expanding opportunities.' },
		{ key: 'sentinel', label: 'MART', color: '#ffd700', description: 'LTC Market Place — The Accessible Marketplace. Buy and sell adaptive products, assistive technology, and accessible equipment at fair prices.' },
		{ key: 'medic', label: 'PIL', color: '#33ff88', description: 'LTC Internship Pilot Program — A 10-week engineering internship designing assistive devices for people with disabilities.' },
		{ key: 'synergist', label: 'PROD', color: '#bb44ff', description: 'LTC Products — Adaptive devices designed by Level The Curve to maximize independence for people with disabilities.' },
		{ key: 'saboteur', label: 'SAB', color: '#ff44aa' }
	] as const;

	const roleWebsites: Record<string, string> = {
		commando: 'https://wolfwdavid.github.io/ltc.main/',
		ravager: 'https://www.ltcfoundationcorp.com/',
		sentinel: 'https://wolfwdavid.github.io/ltc.marketplace/',
		medic: '/LTC_Internship_Pilot_Program.pdf',
		synergist: ''
	};

	function handleSelect(node: any) {
		selectedNode = node;
		if (showInstructions) showInstructions = false;

		// Open STL viewer when Eating Tool node is clicked
		if (node?.label === 'Eating Tool' && node?.role === 'synergist') {
			showSTLViewer = true;
			return;
		}

		const role = node?.role;
		if (role && role in roleWebsites) {
			const now = Date.now();
			if (now - lastRoleClickTime < 400 && lastRoleClicked === role) {
				// Double click — cancel summary and navigate to website
				if (summaryTimer) clearTimeout(summaryTimer);
				summaryTimer = null;
				lastRoleClickTime = 0;
				lastRoleClicked = '';
				showSummary = null;
				if (roleWebsites[role]) window.open(roleWebsites[role], '_blank');
			} else {
				// First click — delay summary to allow double-click
				lastRoleClickTime = now;
				lastRoleClicked = role;
				if (summaryTimer) clearTimeout(summaryTimer);
				summaryTimer = setTimeout(() => {
					showSummary = role;
					summaryTimer = null;
				}, 400);
			}
		} else {
			lastRoleClickTime = 0;
			lastRoleClicked = '';
			if (summaryTimer) {
				clearTimeout(summaryTimer);
				summaryTimer = null;
			}
		}
	}

	function getRoleName(role: string): string {
		const map: Record<string, string> = {
			core: 'Crystal Core',
			commando: 'Commando',
			ravager: 'Ravager',
			sentinel: 'Sentinel',
			medic: 'Medic',
			synergist: 'Synergist',
			saboteur: 'Saboteur'
		};
		return map[role] || role;
	}

	function getRoleColor(role: string): string {
		const map: Record<string, string> = {
			core: '#ffffff',
			commando: '#ff3333',
			ravager: '#33aaff',
			sentinel: '#ffd700',
			medic: '#33ff88',
			synergist: '#bb44ff',
			saboteur: '#ff44aa'
		};
		return map[role] || '#888';
	}
</script>

<svelte:head>
	<title>CRYSTARIUM</title>
	<meta
		name="description"
		content="An interactive Crystarium visualization inspired by Final Fantasy XIII"
	/>
</svelte:head>

<CrystariumCanvas onselect={handleSelect} {activeRole} />

<!-- UI OVERLAY -->
<div class="ui-overlay">
	<!-- Title -->
	<div class="title-bar">
		<h1 class="title">CRYSTARIUM</h1>
		<div class="title-line"></div>
	</div>

	<!-- Role Filter -->
	<div class="role-filter">
		{#each roles as role}
			<button
				class="role-btn"
				class:active={activeRole === role.key}
				class:has-tooltip={'description' in role && !!role.description}
				style="--role-color: {role.color}"
				onclick={() => (activeRole = role.key)}
			>
				<span class="role-dot" style="background: {role.color}"></span>
				{role.label}
				{#if 'description' in role && role.description}
					<span class="role-tooltip">{role.description}</span>
				{/if}
			</button>
		{/each}
	</div>

	<!-- Selected Node Info -->
	{#if selectedNode}
		<div class="info-panel" style="--panel-color: {getRoleColor(selectedNode.role)}">
			<div class="info-header">
				<div class="info-role-badge" style="background: {getRoleColor(selectedNode.role)}">
					{getRoleName(selectedNode.role)}
				</div>
				<span class="info-type">
					{selectedNode.type === 'core'
						? 'CRYSTAL'
						: selectedNode.type === 'large'
							? 'MASTERY'
							: selectedNode.type === 'medium'
								? 'ABILITY'
								: 'ENHANCEMENT'}
				</span>
			</div>
			<h2 class="info-name">{selectedNode.label}</h2>
			<p class="info-desc">{selectedNode.description}</p>
			<div class="info-status">
				{#if selectedNode.active}
					<span class="status-active">CRYSTALLIZED</span>
				{:else}
					<span class="status-locked">LOCKED</span>
				{/if}
			</div>
		</div>
	{/if}

	<!-- Summary Modals -->
	{#if showSummary}
		<div class="summary-backdrop" onclick={() => (showSummary = null)} role="presentation"></div>

		{#if showSummary === 'commando'}
			<div class="summary-panel" style="--accent: #ff3333; --accent-glow: rgba(255, 51, 51, 0.5)">
				<button class="summary-close" onclick={() => (showSummary = null)}>&times;</button>
				<div class="summary-header">
					<div class="summary-icon"></div>
					<h2 class="summary-title">LEVEL THE CURVE</h2>
					<p class="summary-tagline">Access. Independence. Dignity. Designing for all.</p>
				</div>
				<div class="summary-body">
					<p>Level The Curve is a social enterprise focused on <strong>accessibility and disability inclusion</strong>. Founded by co-founders Stefan and Eli, the organization designs and manufactures adaptive products for people with disabilities.</p>

					<div class="summary-section">
						<h3>Products</h3>
						<p>The <strong>EZ Adapt</strong> product line features sleek, affordable devices that assist with Activities of Daily Living (ADLs), designed to maximize user independence.</p>
					</div>

					<div class="summary-section">
						<h3>Services</h3>
						<ul>
							<li>Product consultations</li>
							<li>Donation opportunities to support the mission</li>
							<li>Newsletter updates on developments &amp; fundraisers</li>
						</ul>
					</div>

					<div class="summary-section">
						<h3>Partners</h3>
						<div class="summary-partners">
							<span class="partner-tag">NYU Ability Project</span>
							<span class="partner-tag">El Barrio's Artspace</span>
							<span class="partner-tag">Mount Sinai Hospital</span>
						</div>
					</div>
				</div>
				<a class="summary-link" href="https://wolfwdavid.github.io/ltc.main/" target="_blank" rel="noopener noreferrer">
					Visit Level The Curve
				</a>
			</div>
		{/if}

		{#if showSummary === 'ravager'}
			<div class="summary-panel" style="--accent: #33aaff; --accent-glow: rgba(51, 170, 255, 0.5)">
				<button class="summary-close" onclick={() => (showSummary = null)}>&times;</button>
				<div class="summary-header">
					<div class="summary-icon"></div>
					<h2 class="summary-title">LTC FOUNDATION CORP</h2>
					<p class="summary-tagline">Empowering individuals without limits.</p>
				</div>
				<div class="summary-body">
					<p><strong>LTC Foundation Corp</strong> is a registered 501(c)(3) nonprofit organization based in New York, focused on removing barriers and expanding opportunities for people facing challenges.</p>

					<div class="summary-section">
						<h3>Mission</h3>
						<p>Empowering individuals without limits through community engagement, advocacy, and direct support programs.</p>
					</div>

					<div class="summary-section">
						<h3>Get Involved</h3>
						<ul>
							<li>Community events and activities</li>
							<li>Donate via integrated PayPal</li>
							<li>Sign up for development &amp; fundraiser updates</li>
						</ul>
					</div>

					<div class="summary-section">
						<h3>Connect</h3>
						<div class="summary-partners">
							<span class="partner-tag">Instagram @levelthecurve</span>
							<span class="partner-tag">LinkedIn</span>
						</div>
					</div>
				</div>
				<a class="summary-link" href="https://www.ltcfoundationcorp.com/" target="_blank" rel="noopener noreferrer">
					Visit LTC Foundation Corp
				</a>
			</div>
		{/if}

		{#if showSummary === 'sentinel'}
			<div class="summary-panel" style="--accent: #ffd700; --accent-glow: rgba(255, 215, 0, 0.5)">
				<button class="summary-close" onclick={() => (showSummary = null)}>&times;</button>
				<div class="summary-header">
					<div class="summary-icon"></div>
					<h2 class="summary-title">LTC MARKET PLACE</h2>
					<p class="summary-tagline">The Marketplace That Levels The Playing Field.</p>
				</div>
				<div class="summary-body">
					<p><strong>LTC Market Place</strong> is an online marketplace created by the disability community, dedicated to buying and selling adaptive products, assistive technology, and accessible equipment at fair prices.</p>

					<div class="summary-section">
						<h3>Mission</h3>
						<p>Combating the higher costs disabled individuals face when purchasing adaptive goods by providing an accessible, community-driven marketplace.</p>
					</div>

					<div class="summary-section">
						<h3>Categories</h3>
						<ul>
							<li>Adaptive products &amp; assistive technology</li>
							<li>Accessible equipment</li>
							<li>Daily living aids</li>
						</ul>
					</div>

					<div class="summary-section">
						<h3>Features</h3>
						<div class="summary-partners">
							<span class="partner-tag">Buy &amp; Sell</span>
							<span class="partner-tag">Fair Pricing</span>
							<span class="partner-tag">Community Driven</span>
						</div>
					</div>
				</div>
				<a class="summary-link" href="https://wolfwdavid.github.io/ltc.marketplace/" target="_blank" rel="noopener noreferrer">
					Visit LTC Market Place
				</a>
			</div>
		{/if}

		{#if showSummary === 'medic'}
			<div class="summary-panel" style="--accent: #33ff88; --accent-glow: rgba(51, 255, 136, 0.5)">
				<button class="summary-close" onclick={() => (showSummary = null)}>&times;</button>
				<div class="summary-header">
					<div class="summary-icon"></div>
					<h2 class="summary-title">INTERNSHIP PILOT PROGRAM</h2>
					<p class="summary-tagline">Assistive Device Design &amp; Adaptive Technology.</p>
				</div>
				<div class="summary-body">
					<p>A <strong>10-week engineering internship</strong> where teams of 2–3 interns design, prototype, and validate new assistive devices — guided by LTC's lived-experience philosophy. Top projects may join the EZ Adapt product line.</p>

					<div class="summary-section">
						<h3>Program Details</h3>
						<ul>
							<li>Duration: 10 Weeks (June 2 – August 8)</li>
							<li>Cohort: 6–10 Interns in teams of 2–3</li>
							<li>Location: LTC Design Lab, New York, NY</li>
							<li>Mentors: LTC Co-Founders + Licensed P.E. Advisors</li>
						</ul>
					</div>

					<div class="summary-section">
						<h3>Program Phases</h3>
						<ul>
							<li>Weeks 1–2: Research &amp; Problem Discovery</li>
							<li>Weeks 3–4: Conceptual Design &amp; Engineering Analysis</li>
							<li>Weeks 5–6: User Feedback &amp; Design Review</li>
							<li>Weeks 7–8: Prototype Build &amp; Iterative Testing</li>
							<li>Weeks 9–10: Documentation &amp; Capstone Pitch</li>
						</ul>
					</div>

					<div class="summary-section">
						<h3>Core Values</h3>
						<div class="summary-partners">
							<span class="partner-tag">Affordability (&le;$30)</span>
							<span class="partner-tag">Equity</span>
							<span class="partner-tag">Empowerment</span>
						</div>
					</div>
				</div>
				<a class="summary-link" href="/LTC_Internship_Pilot_Program.pdf" target="_blank" rel="noopener noreferrer">
					View Full Program Document
				</a>
			</div>
		{/if}

		{#if showSummary === 'synergist'}
			<div class="summary-panel" style="--accent: #bb44ff; --accent-glow: rgba(187, 68, 255, 0.5)">
				<button class="summary-close" onclick={() => (showSummary = null)}>&times;</button>
				<div class="summary-header">
					<div class="summary-icon"></div>
					<h2 class="summary-title">LTC PRODUCTS</h2>
					<p class="summary-tagline">Adaptive devices designed for independence &amp; dignity.</p>
				</div>
				<div class="summary-body">
					<p>The <strong>EZ Adapt</strong> product line by Level The Curve — affordable, sleek assistive devices that empower people with disabilities in their daily lives.</p>
				</div>
				<div class="product-grid">
					<button class="product-node" style="--node-hue: 270" onclick={() => { showSummary = null; showSTLViewer = true; }}>
						<div class="product-node-orb"></div>
						<span class="product-node-label">Eating Tool</span>
						<span class="product-node-badge">3D</span>
					</button>
					<button class="product-node" style="--node-hue: 290">
						<div class="product-node-orb"></div>
						<span class="product-node-label">Lawrence Chair</span>
					</button>
					<button class="product-node" style="--node-hue: 250">
						<div class="product-node-orb"></div>
						<span class="product-node-label">Universal Eating Tool</span>
					</button>
					<button class="product-node" style="--node-hue: 310">
						<div class="product-node-orb"></div>
						<span class="product-node-label">Vulcan Grip</span>
					</button>
					<button class="product-node" style="--node-hue: 230">
						<div class="product-node-orb"></div>
						<span class="product-node-label">Leg Bag</span>
					</button>
					<button class="product-node" style="--node-hue: 330">
						<div class="product-node-orb"></div>
						<span class="product-node-label">Manual Chair Ramp</span>
					</button>
				</div>
			</div>
		{/if}
	{/if}

	<!-- Instructions -->
	{#if showInstructions}
		<div class="instructions">
			<p>Drag to rotate &bull; Scroll to zoom &bull; Click nodes to crystallize</p>
		</div>
	{/if}

	<!-- STL 3D Viewer -->
	{#if showSTLViewer}
		<STLViewer url="{base}/eating-tool.stl" onclose={() => (showSTLViewer = false)} />
	{/if}
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
		background: #000;
		font-family:
			'Segoe UI',
			-apple-system,
			BlinkMacSystemFont,
			sans-serif;
		color: #fff;
		user-select: none;
	}

	.ui-overlay {
		position: fixed;
		inset: 0;
		pointer-events: none;
		z-index: 10;
	}

	/* Title */
	.title-bar {
		position: absolute;
		top: 30px;
		left: 40px;
	}

	.title {
		font-size: 1.6rem;
		font-weight: 300;
		letter-spacing: 0.6em;
		color: rgba(255, 255, 255, 0.85);
		margin: 0;
		text-shadow:
			0 0 20px rgba(150, 130, 255, 0.4),
			0 0 60px rgba(100, 80, 200, 0.15);
	}

	.title-line {
		width: 200px;
		height: 1px;
		background: linear-gradient(90deg, rgba(150, 130, 255, 0.5), transparent);
		margin-top: 8px;
	}

	/* Role Filter */
	.role-filter {
		position: absolute;
		top: 30px;
		right: 40px;
		display: flex;
		gap: 4px;
		pointer-events: all;
	}

	.role-btn {
		background: rgba(255, 255, 255, 0.04);
		border: 1px solid rgba(255, 255, 255, 0.08);
		color: rgba(255, 255, 255, 0.6);
		padding: 6px 14px;
		font-size: 0.7rem;
		font-weight: 500;
		letter-spacing: 0.15em;
		cursor: pointer;
		transition: all 0.3s ease;
		display: flex;
		align-items: center;
		gap: 6px;
		backdrop-filter: blur(10px);
		text-decoration: none;
	}

	.role-btn:hover {
		background: rgba(255, 255, 255, 0.08);
		border-color: var(--role-color);
		color: #fff;
	}

	.role-btn.active {
		background: rgba(255, 255, 255, 0.1);
		border-color: var(--role-color);
		color: #fff;
		box-shadow: 0 0 15px rgba(150, 130, 255, 0.15);
	}

	.role-dot {
		width: 6px;
		height: 6px;
		border-radius: 50%;
		display: inline-block;
	}

	.has-tooltip {
		position: relative;
	}

	.role-tooltip {
		display: none;
		position: absolute;
		top: calc(100% + 10px);
		right: 0;
		width: 260px;
		padding: 12px 14px;
		background: rgba(8, 6, 20, 0.85);
		border: 1px solid var(--role-color, rgba(255, 255, 255, 0.1));
		backdrop-filter: blur(16px);
		font-size: 0.7rem;
		font-weight: 400;
		letter-spacing: 0.04em;
		line-height: 1.5;
		color: rgba(255, 255, 255, 0.65);
		text-align: left;
		white-space: normal;
		z-index: 20;
	}

	.has-tooltip:hover .role-tooltip {
		display: block;
		animation: fadeIn 0.2s ease;
	}

	/* Info Panel */
	.info-panel {
		position: absolute;
		bottom: 80px;
		right: 40px;
		width: 300px;
		background: rgba(8, 6, 20, 0.75);
		border: 1px solid rgba(255, 255, 255, 0.08);
		border-left: 2px solid var(--panel-color);
		backdrop-filter: blur(20px);
		padding: 20px 24px;
		pointer-events: all;
		animation: slideIn 0.3s ease;
	}

	@keyframes slideIn {
		from {
			opacity: 0;
			transform: translateX(20px);
		}
		to {
			opacity: 1;
			transform: translateX(0);
		}
	}

	.info-header {
		display: flex;
		align-items: center;
		gap: 10px;
		margin-bottom: 12px;
	}

	.info-role-badge {
		font-size: 0.6rem;
		font-weight: 600;
		letter-spacing: 0.15em;
		padding: 2px 10px;
		color: #000;
	}

	.info-type {
		font-size: 0.6rem;
		letter-spacing: 0.2em;
		color: rgba(255, 255, 255, 0.35);
	}

	.info-name {
		font-size: 1.2rem;
		font-weight: 300;
		letter-spacing: 0.1em;
		margin: 0 0 8px;
		color: var(--panel-color);
		text-shadow: 0 0 20px var(--panel-color);
	}

	.info-desc {
		font-size: 0.8rem;
		line-height: 1.5;
		color: rgba(255, 255, 255, 0.5);
		margin: 0 0 14px;
	}

	.info-status {
		font-size: 0.65rem;
		letter-spacing: 0.2em;
	}

	.status-active {
		color: #66ffaa;
		text-shadow: 0 0 10px rgba(100, 255, 170, 0.3);
	}

	.status-locked {
		color: rgba(255, 255, 255, 0.3);
	}

	/* Core Summary Modal */
	.summary-backdrop {
		position: fixed;
		inset: 0;
		background: rgba(0, 0, 0, 0.6);
		z-index: 50;
		pointer-events: all;
		animation: fadeIn 0.3s ease;
	}

	@keyframes fadeIn {
		from { opacity: 0; }
		to { opacity: 1; }
	}

	.summary-panel {
		position: fixed;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		width: min(520px, 90vw);
		max-height: 80vh;
		overflow-y: auto;
		background: rgba(10, 8, 28, 0.92);
		border: 1px solid color-mix(in srgb, var(--accent) 25%, transparent);
		border-top: 2px solid var(--accent);
		backdrop-filter: blur(24px);
		padding: 32px 36px;
		z-index: 60;
		pointer-events: all;
		animation: summaryIn 0.4s ease;
	}

	@keyframes summaryIn {
		from {
			opacity: 0;
			transform: translate(-50%, -50%) scale(0.92);
		}
		to {
			opacity: 1;
			transform: translate(-50%, -50%) scale(1);
		}
	}

	.summary-close {
		position: absolute;
		top: 12px;
		right: 16px;
		background: none;
		border: none;
		color: rgba(255, 255, 255, 0.4);
		font-size: 1.6rem;
		cursor: pointer;
		line-height: 1;
		padding: 4px 8px;
		transition: color 0.2s;
	}

	.summary-close:hover {
		color: var(--accent);
	}

	.summary-header {
		text-align: center;
		margin-bottom: 24px;
	}

	.summary-icon {
		width: 14px;
		height: 14px;
		background: var(--accent);
		border-radius: 50%;
		margin: 0 auto 14px;
		box-shadow: 0 0 20px var(--accent-glow), 0 0 40px color-mix(in srgb, var(--accent) 20%, transparent);
	}

	.summary-title {
		font-size: 1.3rem;
		font-weight: 300;
		letter-spacing: 0.4em;
		color: var(--accent);
		margin: 0 0 8px;
		text-shadow: 0 0 20px color-mix(in srgb, var(--accent) 35%, transparent);
	}

	.summary-tagline {
		font-size: 0.72rem;
		letter-spacing: 0.12em;
		color: rgba(255, 255, 255, 0.4);
		margin: 0;
		font-style: italic;
	}

	.summary-body {
		color: rgba(255, 255, 255, 0.6);
		font-size: 0.82rem;
		line-height: 1.7;
	}

	.summary-body p {
		margin: 0 0 16px;
	}

	.summary-body strong {
		color: rgba(255, 255, 255, 0.85);
		font-weight: 500;
	}

	.summary-section {
		margin-bottom: 18px;
	}

	.summary-section h3 {
		font-size: 0.68rem;
		font-weight: 500;
		letter-spacing: 0.2em;
		text-transform: uppercase;
		color: color-mix(in srgb, var(--accent) 70%, transparent);
		margin: 0 0 8px;
		padding-bottom: 4px;
		border-bottom: 1px solid color-mix(in srgb, var(--accent) 12%, transparent);
	}

	.summary-section ul {
		margin: 0;
		padding-left: 18px;
	}

	.summary-section li {
		margin-bottom: 4px;
	}

	.summary-partners {
		display: flex;
		flex-wrap: wrap;
		gap: 8px;
	}

	.partner-tag {
		background: color-mix(in srgb, var(--accent) 8%, transparent);
		border: 1px solid color-mix(in srgb, var(--accent) 15%, transparent);
		padding: 4px 12px;
		font-size: 0.7rem;
		letter-spacing: 0.06em;
		color: rgba(255, 255, 255, 0.55);
	}

	.summary-link {
		display: block;
		text-align: center;
		margin-top: 24px;
		padding: 10px 20px;
		background: color-mix(in srgb, var(--accent) 10%, transparent);
		border: 1px solid color-mix(in srgb, var(--accent) 30%, transparent);
		color: var(--accent);
		text-decoration: none;
		font-size: 0.75rem;
		letter-spacing: 0.12em;
		transition: all 0.3s ease;
	}

	.summary-link:hover {
		background: color-mix(in srgb, var(--accent) 20%, transparent);
		border-color: var(--accent);
		color: #fff;
		box-shadow: 0 0 20px color-mix(in srgb, var(--accent) 15%, transparent);
	}

	/* Product Grid */
	.product-grid {
		display: grid;
		grid-template-columns: repeat(3, 1fr);
		gap: 16px;
		margin-top: 24px;
	}

	.product-node {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 10px;
		padding: 18px 10px;
		background: rgba(187, 68, 255, 0.04);
		border: 1px solid rgba(187, 68, 255, 0.12);
		border-radius: 4px;
		cursor: pointer;
		transition: all 0.3s ease;
	}

	.product-node:hover {
		background: rgba(187, 68, 255, 0.1);
		border-color: rgba(187, 68, 255, 0.4);
		transform: translateY(-2px);
		box-shadow: 0 4px 20px rgba(187, 68, 255, 0.15);
	}

	.product-node-orb {
		width: 28px;
		height: 28px;
		border-radius: 50%;
		background: radial-gradient(circle at 40% 35%,
			hsl(var(--node-hue), 80%, 75%),
			hsl(var(--node-hue), 70%, 45%)
		);
		box-shadow:
			0 0 12px hsla(var(--node-hue), 80%, 60%, 0.5),
			0 0 30px hsla(var(--node-hue), 80%, 60%, 0.2);
		animation: orbPulse 3s ease-in-out infinite;
	}

	@keyframes orbPulse {
		0%, 100% { box-shadow: 0 0 12px hsla(var(--node-hue), 80%, 60%, 0.5), 0 0 30px hsla(var(--node-hue), 80%, 60%, 0.2); }
		50% { box-shadow: 0 0 18px hsla(var(--node-hue), 80%, 60%, 0.7), 0 0 40px hsla(var(--node-hue), 80%, 60%, 0.3); }
	}

	.product-node-label {
		font-size: 0.68rem;
		font-weight: 500;
		letter-spacing: 0.08em;
		color: rgba(255, 255, 255, 0.6);
		text-align: center;
		line-height: 1.3;
	}

	.product-node:hover .product-node-label {
		color: rgba(255, 255, 255, 0.9);
	}

	.product-node-badge {
		font-size: 0.55rem;
		font-weight: 600;
		letter-spacing: 0.1em;
		padding: 1px 6px;
		background: rgba(187, 68, 255, 0.2);
		border: 1px solid rgba(187, 68, 255, 0.4);
		color: #bb44ff;
		border-radius: 2px;
	}

	/* Instructions */
	.instructions {
		position: absolute;
		bottom: 30px;
		left: 50%;
		transform: translateX(-50%);
		animation: fadeInUp 1s ease 0.5s both;
	}

	.instructions p {
		font-size: 0.75rem;
		letter-spacing: 0.15em;
		color: rgba(255, 255, 255, 0.3);
		margin: 0;
	}

	@keyframes fadeInUp {
		from {
			opacity: 0;
			transform: translateX(-50%) translateY(10px);
		}
		to {
			opacity: 1;
			transform: translateX(-50%) translateY(0);
		}
	}

	/* Responsive */
	@media (max-width: 768px) {
		.title-bar {
			top: 16px;
			left: 16px;
		}
		.title {
			font-size: 1rem;
			letter-spacing: 0.3em;
		}
		.role-filter {
			top: auto;
			bottom: 70px;
			left: 16px;
			right: 16px;
			justify-content: center;
			flex-wrap: wrap;
		}
		.role-btn {
			padding: 5px 10px;
			font-size: 0.6rem;
		}
		.info-panel {
			left: 16px;
			right: 16px;
			bottom: 140px;
			width: auto;
		}
		.summary-panel {
			padding: 24px 20px;
		}
		.product-grid {
			grid-template-columns: repeat(2, 1fr);
			gap: 12px;
		}
	}
</style>
