<script lang="ts">
	import { writable } from 'svelte/store';
	import GradientCanvas from '$lib/components/GradientCanvas.svelte';
	import ControlPanel from '$lib/components/ControlPanel.svelte';
	import ShaderCode from '$lib/components/ShaderCode.svelte';
	import { Tabs, TabsContent, TabsList, TabsTrigger } from '$lib/components/ui/tabs';

	const settings = writable({
		colors: ['#ff0000', '#0000ff'],
		speed: 1,
		isAnimated: true,
		animationPreset: 'noise',
		scale: 1,
		chaos: 0
	});

	let activeTab = 'preview';

	// This function is called whenever `settings` should be updated from `ControlPanel`
	function updateSettings(newSettings) {
		$settings = { ...newSettings };
	}
</script>

<div class="flex h-screen flex-col">
	<Tabs value={activeTab} onValueChange={(value) => (activeTab = value)} class="w-full pt-3 pl-1">
		<TabsList>
			<TabsTrigger value="preview">Preview</TabsTrigger>
			<TabsTrigger value="code">Shader Code</TabsTrigger>
		</TabsList>
		<TabsContent value="preview" class="flex-grow">
			<div class="flex h-full flex-col md:flex-row">
				<div class="flex-grow">
					<GradientCanvas settings={$settings} />
				</div>
				<div class="w-full p-4 md:w-64">
					<ControlPanel bind:settings={$settings} on:settingsChange={updateSettings} />
				</div>
			</div>
		</TabsContent>
		<TabsContent value="code" class="flex-grow">
			<ShaderCode settings={$settings} />
		</TabsContent>
	</Tabs>
</div>
