<script lang="ts">
	import CrystariumCanvas from '$lib/CrystariumCanvas.svelte';

	let selectedNode = $state<any>(null);
	let activeRole = $state('all');
	let showInstructions = $state(true);
	let showCoreSummary = $state(false);
	let lastCoreClickTime = 0;
	let coreSummaryTimer: ReturnType<typeof setTimeout> | null = null;

	const roles = [
		{ key: 'all', label: 'ALL', color: '#ffffff' },
		{ key: 'commando', label: 'LTC', color: '#ff3333', description: 'Level The Curve — Access. Independence. Dignity. Improving accessibility and independence for people with disabilities.' },
		{ key: 'ravager', label: 'RAV', color: '#33aaff' },
		{ key: 'sentinel', label: 'SEN', color: '#ffd700' },
		{ key: 'medic', label: 'MED', color: '#33ff88' },
		{ key: 'synergist', label: 'SYN', color: '#bb44ff' },
		{ key: 'saboteur', label: 'SAB', color: '#ff44aa' }
	] as const;

	function handleSelect(node: any) {
		selectedNode = node;
		if (showInstructions) showInstructions = false;
		if (node?.role === 'commando') {
			const now = Date.now();
			if (now - lastCoreClickTime < 400) {
				// Double click — cancel summary and navigate to website
				if (coreSummaryTimer) clearTimeout(coreSummaryTimer);
				coreSummaryTimer = null;
				lastCoreClickTime = 0;
				showCoreSummary = false;
				window.open('https://wolfwdavid.github.io/ltc.main/', '_blank');
			} else {
				// First click — delay summary to allow double-click
				lastCoreClickTime = now;
				if (coreSummaryTimer) clearTimeout(coreSummaryTimer);
				coreSummaryTimer = setTimeout(() => {
					showCoreSummary = true;
					coreSummaryTimer = null;
				}, 400);
			}
		} else {
			lastCoreClickTime = 0;
			if (coreSummaryTimer) {
				clearTimeout(coreSummaryTimer);
				coreSummaryTimer = null;
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

	<!-- Core Summary Modal -->
	{#if showCoreSummary}
		<div class="summary-backdrop" onclick={() => (showCoreSummary = false)} role="presentation"></div>
		<div class="summary-panel">
			<button class="summary-close" onclick={() => (showCoreSummary = false)}>&times;</button>
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

	<!-- Instructions -->
	{#if showInstructions}
		<div class="instructions">
			<p>Drag to rotate &bull; Scroll to zoom &bull; Click nodes to crystallize</p>
		</div>
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
		border: 1px solid rgba(255, 60, 60, 0.25);
		border-top: 2px solid #ff3333;
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
		color: #ff3333;
	}

	.summary-header {
		text-align: center;
		margin-bottom: 24px;
	}

	.summary-icon {
		width: 14px;
		height: 14px;
		background: #ff3333;
		border-radius: 50%;
		margin: 0 auto 14px;
		box-shadow: 0 0 20px rgba(255, 51, 51, 0.5), 0 0 40px rgba(255, 51, 51, 0.2);
	}

	.summary-title {
		font-size: 1.3rem;
		font-weight: 300;
		letter-spacing: 0.4em;
		color: #ff3333;
		margin: 0 0 8px;
		text-shadow: 0 0 20px rgba(255, 51, 51, 0.35);
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
		color: rgba(255, 51, 51, 0.7);
		margin: 0 0 8px;
		padding-bottom: 4px;
		border-bottom: 1px solid rgba(255, 51, 51, 0.12);
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
		background: rgba(255, 51, 51, 0.08);
		border: 1px solid rgba(255, 51, 51, 0.15);
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
		background: rgba(255, 51, 51, 0.1);
		border: 1px solid rgba(255, 51, 51, 0.3);
		color: #ff6b6b;
		text-decoration: none;
		font-size: 0.75rem;
		letter-spacing: 0.12em;
		transition: all 0.3s ease;
	}

	.summary-link:hover {
		background: rgba(255, 51, 51, 0.2);
		border-color: #ff3333;
		color: #fff;
		box-shadow: 0 0 20px rgba(255, 51, 51, 0.15);
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
	}
</style>
