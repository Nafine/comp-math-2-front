<script lang="ts">
	import functionPlot from 'function-plot';

	type Mode = 'single' | 'system';
	type SingleMethod = 'chord' | 'secant' | 'iteration';
	type SystemMethod = 'newton';

	interface SingleEquation {
		id: number;
		label: string;
		fn: string;
	}

	interface SystemEquation {
		id: number;
		label: string;
		eq1: string;
		eq2: string;
	}

	interface SolveResponse {
		x: number;
		y?: number;
		dx?: number;
		dy?: number;
		iterationCount: number;
	}

	interface SinglePayload {
		type: 'single';
		equationId: number;
		method: SingleMethod;
		a: number;
		b: number;
		tolerance: number;
	}

	interface SystemPayload {
		type: 'system';
		equationId: number;
		method: SystemMethod;
		x0: number;
		y0: number;
		tolerance: number;
	}

	type Payload = SinglePayload | SystemPayload;

	const singleEqs: SingleEquation[] = [
		{ id: 0, label: "x³ + 2.84x² - 5.606x - 14.766 = 0", fn: "x^3 + 2.84*x^2 - 5.606*x - 14.766" },
		{ id: 1, label: "x³ - 1.89x² - 2x + 1.76 = 0", fn: "x^3 - 1.89*x^2 - 2*x + 1.76" },
		{ id: 2, label: "sin(3x) - 0.5 = 0", fn: "sin(3*x) - 0.5" }
	];

	const systemEqs: SystemEquation[] = [
		{ id: 0, label: "cos(x-1) + y = 0.5  И  x - cos(y) = 3", eq1: "cos(x-1) + y - 0.5", eq2: "x - cos(y) - 3" },
		{ id: 1, label: "sin(x+y) = 1.5x - 0.1  И  x² + 2y² = 1", eq1: "sin(x+y) - 1.5*x + 0.1", eq2: "x^2 + 2*y^2 - 1" }
	];

	let mode = $state<Mode>('single');
	
	let selectedSingleEq = $state<number>(0);
	let selectedSystemEq = $state<number>(0);

	let a = $state<number>(-5);
	let b = $state<number>(5);
	let singleMethod = $state<SingleMethod>('chord');
	
	let x0 = $state<number>(0);
	let y0 = $state<number>(0);
	let systemMethod = $state<SystemMethod>('newton');

	let tolerance = $state<number>(0.001);

	let result = $state<SolveResponse | null>(null);
	let isLoading = $state<boolean>(false);
	let errorMsg = $state<string | null>(null);

	let plotContainer: HTMLDivElement | undefined = $state();

	$effect(() => {
		if (plotContainer) {
			const currentMode = mode;
			const sEq = selectedSingleEq;
			const sysEq = selectedSystemEq;
			drawPlot();
		}
	});

	function drawPlot(): void {
		if (!plotContainer) return;

		try {
			const data: any[] = mode === 'single'
				? [{ fn: singleEqs[selectedSingleEq].fn, color: '#ff5722', nSamples: 5000}]
				: [
					{ fn: systemEqs[selectedSystemEq].eq1, fnType: 'implicit', color: '#2196f3' },
					{ fn: systemEqs[selectedSystemEq].eq2, fnType: 'implicit', color: '#4caf50' }
				  ];

			functionPlot({
				target: plotContainer,
				width: 600,
				height: 400,
				grid: true,
				xAxis: { domain: [-10, 10] },
				yAxis: { domain: [-10, 10] },
				data
			});
		} catch (err) {
			console.error("Ошибка построения графика:", err);
		}
	}

	async function handleSubmit(): Promise<void> {
        const baseUrl = import.meta.env.VITE_API_URL || 'http://localhost:8080';

		isLoading = true;
		errorMsg = null;
		result = null;

		let payload: Payload;

		if (mode === 'single') {
			payload = {
				type: 'single',
				equationId: selectedSingleEq,
				method: singleMethod,
				a,
				b,
				tolerance,
			};
		} else {
			payload = {
				type: 'system',
				equationId: selectedSystemEq,
				method: systemMethod,
				x0,
				y0,
				tolerance,
			};
		}

		try {
        const res = await fetch(`/api/solve`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });

        const data = await res.json();

        if (!res.ok) {
            throw new Error(`Ошибка: ${data.error}` || `Ошибка сервера: ${res.status}`);
        }
        
        result = data as SolveResponse;

		} catch (err) {
			errorMsg = err instanceof Error ? err.message : String(err);
		} finally {
			isLoading = false;
		}
	}
</script>

<main class="container">
	<h1>Вычислительная математика 2 Зенченков P3215</h1>

	<div class="layout">
		<div class="controls">
			<div class="section">
				<h3>Тип задачи</h3>
				<label>
					<input type="radio" bind:group={mode} value="single"> Нелинейное уравнение
				</label>
				<label>
					<input type="radio" bind:group={mode} value="system"> Система нелинейных уравнений
				</label>
			</div>

			<div class="section">
				<h3>Выбор уравнения</h3>
				{#if mode === 'single'}
					{#each singleEqs as eq}
						<label>
							<input type="radio" bind:group={selectedSingleEq} value={eq.id}> {eq.label}
						</label>
					{/each}
				{:else}
					{#each systemEqs as eq}
						<label>
							<input type="radio" bind:group={selectedSystemEq} value={eq.id}> {eq.label}
						</label>
					{/each}
				{/if}
			</div>

			<div class="section">
				<h3>Метод решения</h3>
				{#if mode === 'single'}
					<select bind:value={singleMethod}>
						<option value="chord">Метод хорд</option>
						<option value="secant">Метод секущих</option>
						<option value="iteration">Метод простых итераций</option>
					</select>
				{:else}
					<select bind:value={systemMethod}>
						<option value="newton">Метод Ньютона</option>
					</select>
				{/if}
			</div>

			<div class="section">
				<h3>Параметры</h3>
				{#if mode === 'single'}
					<div class="input-group">
						<label>Интервал [a]: <input type="number" step="0.1" bind:value={a}></label>
						<label>Интервал [b]: <input type="number" step="0.1" bind:value={b}></label>
					</div>
				{:else}
					<div class="input-group">
						<label>Начальное x₀: <input type="number" step="0.1" bind:value={x0}></label>
						<label>Начальное y₀: <input type="number" step="0.1" bind:value={y0}></label>
					</div>
				{/if}
				<label style="margin-top: 10px; display: block;">
					Погрешность (ε): <input type="number" min="0" step="0.001" bind:value={tolerance}>
				</label>
			</div>

			<button onclick={handleSubmit} disabled={isLoading}>
				{isLoading ? 'Вычисляем...' : 'Решить'}
			</button>

			{#if result}
				<div class="result success">
					<h4>Результат:</h4>
					<p><b>X:</b> {result.x}</p>
					{#if result.y !== undefined}
						<p><b>Y:</b> {result.y}</p>
					{/if}
					{#if result.dx !== undefined && result.dy !== undefined}
						<p><b>Dx:</b> {result.dx}</p>
						<p><b>Dy:</b> {result.dy}</p>
					{/if}
					<p><b>Количество итераций:</b> {result.iterationCount}</p>
				</div>
			{/if}

			{#if errorMsg}
				<div class="result error">
					<p>{errorMsg}</p>
				</div>
			{/if}
		</div>

		<div class="chart">
			<h3>График</h3>
			<div bind:this={plotContainer}></div>
		</div>
	</div>
</main>

<style>
	.container { max-width: 1000px; margin: 0 auto; padding: 20px; font-family: sans-serif; }
	.layout { display: flex; gap: 40px; }
	.controls { flex: 1; }
	.chart { flex: 1; display: flex; flex-direction: column; align-items: center; }
	.section { margin-bottom: 20px; padding: 15px; border: 1px solid #ddd; border-radius: 8px; }
	h3 { margin-top: 0; margin-bottom: 15px; font-size: 16px; color: #333; }
	label { display: block; margin-bottom: 8px; cursor: pointer; }
	.input-group { display: flex; gap: 10px; }
	.input-group label { display: flex; flex-direction: column; font-size: 14px; }
	input[type="number"], select { padding: 6px; margin-top: 4px; border: 1px solid #ccc; border-radius: 4px; }
	button { padding: 10px 20px; background: #007bff; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; width: 100%; }
	button:hover { background: #0056b3; }
	button:disabled { background: #aaa; cursor: not-allowed; }
	.result { margin-top: 20px; padding: 15px; border-radius: 8px; }
	.success { background: #e8f5e9; border: 1px solid #c8e6c9; }
	.error { background: #ffebee; border: 1px solid #ffcdd2; color: #c62828; }
</style>