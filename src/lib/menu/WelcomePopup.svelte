<script lang="ts">
	import GameSelection from '$lib/components/GameSelection.svelte';
	import Popup from '$lib/components/Popup.svelte';
	import BigButton from '$lib/components/BigButton.svelte';
	import PathPref from '$lib/prefs/PathPref.svelte';

	import type { Prefs, R2ImportData } from '$lib/models';

	import { invokeCommand } from '$lib/invoke';
	import { onMount } from 'svelte';
	import ImportR2Flow from '$lib/import/ImportR2Flow.svelte';
	import Icon from '@iconify/svelte';

	export let open = false;

	let stage: 'gameSelect' | 'importProfiles' | 'settings' | 'end' = 'gameSelect';

	let importFrom: 'r2modman' | 'thunderstore' = 'r2modman';
	let importData: R2ImportData = {
		r2modman: undefined,
		thunderstore: undefined
	};

	let importFlow: ImportR2Flow;

	let prefs: Prefs | null = null;

	$: importText =
		importData.r2modman && importData.thunderstore
			? 'r2modman or Thunderstore Mod Manager'
			: importData.r2modman
				? 'r2modman'
				: 'Thunderstore Mod Manager';

	onMount(async () => {
		if (await invokeCommand<boolean>('is_first_run')) {
			open = true;
			prefs = await invokeCommand('get_prefs');
		}
	});

	async function onSelectGame() {
		let result = await invokeCommand<R2ImportData>('get_r2modman_info');

		if (!result.r2modman && !result.thunderstore) {
			stage = 'settings';
			return;
		}

		importData = result;

		if (result.r2modman) {
			result.r2modman.include = result.r2modman.profiles.map(() => true);
			importFrom = 'r2modman';
		}

		if (result.thunderstore) {
			result.thunderstore.include = result.thunderstore.profiles.map(() => true);
			importFrom = 'thunderstore';
		}

		stage = 'importProfiles';
	}

	async function importProfiles() {
		await importFlow.doImport();
		stage = 'settings';
	}

	function set<T>(update: (value: T, prefs: Prefs) => void) {
		return async (value: T) => {
			if (prefs === null) return;

			update(value, prefs);
			await invokeCommand('set_prefs', { value: prefs });
		};
	}
</script>

<Popup title="Welcome to Gale!" canClose={stage === 'end'} bind:open maxWidth="[55%]">
	<div class="text-slate-300">
		{#if stage === 'gameSelect'}
			To get started, select a game to mod:
			<GameSelection onSelect={onSelectGame} />
		{:else if stage === 'importProfiles' && importData}
			<p>
				You can choose automatically transfer profiles from {importText} to Gale.
			</p>

			<p class="mt-1">
				The process may take a couple of minutes, depending on how many mods and profiles there are
				to import. It will also transfer configs and cached mods.
			</p>

			<p class="mt-1">
				You can always import profiles later by going to <b>Import > ...from r2modman</b>.
			</p>

			<ImportR2Flow bind:importData bind:importFrom bind:this={importFlow} />

			<div class="flex mt-2 gap-1.5">
				<BigButton color="gray" on:click={() => (stage = 'gameSelect')}>Back</BigButton>
				<div class="flex-grow" />
				<BigButton color="gray" on:click={() => (stage = 'settings')}>Skip</BigButton>
				<BigButton color="green" on:click={importProfiles}>Import</BigButton>
			</div>
		{:else if stage === 'settings'}
			<p>Lastly, make sure your settings are correct.</p>

			<p class="mt-1">
				You can always edit these later by going to <Icon icon="mdi:settings" class="inline mb-1" />
				<b>Settings</b>.
			</p>

			<div class="flex flex-col mt-3 gap-1">
				{#if prefs !== null}
					<PathPref
						label="Steam executable"
						type="file"
						value={prefs.steamExePath ?? null}
						set={set((value, prefs) => (prefs.steamExePath = value ?? undefined))}
					>
						Path to the Steam executable.
					</PathPref>

					<PathPref
						label="Steam library"
						type="dir"
						value={prefs.steamLibraryDir ?? null}
						set={set((value, prefs) => (prefs.steamLibraryDir = value ?? undefined))}
					>
						Path to the Steam game library. This should <b>contain</b> the 'steamapps' directory.
					</PathPref>

					<PathPref
						label="Gale data directory"
						type="dir"
						value={prefs.dataDir}
						set={set((value, prefs) => (prefs.dataDir = value))}
					>
						Directory where mods and profiles are stored.
					</PathPref>
				{/if}
			</div>

			<div class="flex mt-3 justify-between">
				<BigButton
					color="gray"
					on:click={() =>
						(stage =
							importData.r2modman || importData.thunderstore ? 'importProfiles' : 'gameSelect')}
					>Back</BigButton
				>
				<BigButton color="green" on:click={() => (stage = 'end')}>Next</BigButton>
			</div>
		{:else if stage === 'end'}
			<p>That's it, you're all set up to start modding!</p>

			<p class="mt-1">
				If you have any questions or need help, feel free to ask in the <a
					href="https://discord.gg/lcmod"
					target="_blank"
					class="text-green-400 hover:underline">Lethal Company Modding Discord server</a
				>.
			</p>
		{/if}
	</div>
</Popup>
