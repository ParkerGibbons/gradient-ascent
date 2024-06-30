<script lang="ts">
	import { Button } from '$lib/components/ui/button';
	import { Input } from '$lib/components/ui/input';
	import { Label } from '$lib/components/ui/label';
	import { Slider } from '$lib/components/ui/slider';
	import { Switch } from '$lib/components/ui/switch';
	import {
		Select,
		SelectContent,
		SelectItem,
		SelectTrigger,
		SelectValue
	} from '$lib/components/ui/select';
	import { createEventDispatcher } from 'svelte';

	export let settings: {
		colors: string[];
		speed: number;
		isAnimated: boolean;
		animationPreset: string;
		scale: number;
		chaos: number;
	};

	const dispatch = createEventDispatcher();

	let speedValue = [settings.speed];
	let scaleValue = [settings.scale];
	let chaosValue = [settings.chaos];

	$: settings.speed = speedValue[0];
	$: settings.scale = scaleValue[0];
	$: settings.chaos = chaosValue[0];

	// Whenever any property of `settings` changes, reassign `settings` to a new object
	$: settings = { ...settings, speed: speedValue[0], scale: scaleValue[0], chaos: chaosValue[0] };

	const animationPresets = ['wave', 'spiral', 'noise', 'ripple'];

	function setAnimationPreset(preset: string) {
		settings.animationPreset = preset;
		dispatch('settingsChange', settings);
	}

	function addColor() {
		if (settings.colors.length < 6) {
			settings.colors = [...settings.colors, '#000000'];
		}
	}

	function removeColor(index: number) {
		if (settings.colors.length > 2) {
			settings.colors = settings.colors.filter((_, i) => i !== index);
		}
	}

	function randomizeColors() {
		settings.colors = settings.colors.map(
			() =>
				'#' +
				Math.floor(Math.random() * 16777215)
					.toString(16)
					.padStart(6, '0')
		);
	}

	function randomizeAll() {
		randomizeColors();
		settings.speed = Math.random() * 9.9 + 0.1;
		speedValue = [settings.speed];
		settings.isAnimated = Math.random() < 0.5;
	}
</script>

<div class="bg-card space-y-4 rounded-lg p-4 shadow">
	<div class="space-y-2">
		<Label>Animation Preset</Label>
		<div class="flex flex-wrap gap-2">
			{#each animationPresets as preset}
				<Button on:click={() => setAnimationPreset(preset)}>{preset}</Button>
			{/each}
		</div>
	</div>

	{#each settings.colors as color, i}
		<div class="space-y-2">
			<Label for={`color${i + 1}`}>Color {i + 1}</Label>
			<div class="flex items-center space-x-2">
				<Input type="color" id={`color${i + 1}`} bind:value={settings.colors[i]} />
				{#if settings.colors.length > 2}
					<Button variant="outline" size="icon" on:click={() => removeColor(i)}>-</Button>
				{/if}
			</div>
		</div>
	{/each}
	{#if settings.colors.length < 6}
		<Button variant="outline" on:click={addColor}>Add Color</Button>
	{/if}
	<div class="space-y-2">
		<Label for="speed">Speed: {settings.speed.toFixed(1)}</Label>
		<Slider id="speed" min={0.1} max={10} step={0.1} bind:value={speedValue} />
	</div>

	<div class="space-y-2">
		<Label for="scale">Scale: {settings.scale.toFixed(1)}</Label>
		<Slider id="scale" min={0.1} max={5} step={0.1} bind:value={scaleValue} />
	</div>

	<div class="space-y-2">
		<Label for="chaos">Chaos: {settings.chaos.toFixed(2)}</Label>
		<Slider id="chaos" min={0} max={1} step={0.01} bind:value={chaosValue} />
	</div>

	<div class="flex items-center space-x-2">
		<Switch id="animated" bind:checked={settings.isAnimated} />
		<Label for="animated">Animated</Label>
	</div>

	<Button on:click={randomizeAll}>Randomize All</Button>
</div>
