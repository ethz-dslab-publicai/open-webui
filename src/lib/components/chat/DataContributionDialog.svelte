<script lang="ts">
	import { onMount, getContext, createEventDispatcher, onDestroy, tick } from 'svelte';
	import * as FocusTrap from 'focus-trap';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	import { fade } from 'svelte/transition';
	import { flyAndScale } from '$lib/utils/transitions';

	export let cancelLabel = $i18n.t('Cancel');
	export let confirmLabel = $i18n.t('Submit');

	export let onConfirm = () => {};

	export let show = false;

	$: if (show) {
		init();
	}

	let formData = {};

	let modalElement = null;
	let mounted = false;

	let focusTrap: FocusTrap.FocusTrap | null = null;

	const init = () => {
		formData = {};
	};

	const handleKeyDown = (event: KeyboardEvent) => {
		if (event.key === 'Escape') {
			console.log('Escape');
			show = false;
			dispatch('cancel');
		}
	};

	const confirmHandler = async () => {
		show = false;
		await tick();
		await onConfirm();
		// TODO: Add form data to event
		dispatch('confirm');
	};

	onMount(() => {
		mounted = true;
	});

	$: if (mounted) {
		if (show && modalElement) {
			document.body.appendChild(modalElement);
			focusTrap = FocusTrap.createFocusTrap(modalElement);
			focusTrap.activate();

			window.addEventListener('keydown', handleKeyDown);
			document.body.style.overflow = 'hidden';
		} else if (modalElement) {
			focusTrap.deactivate();

			window.removeEventListener('keydown', handleKeyDown);
			document.body.removeChild(modalElement);

			document.body.style.overflow = 'unset';
		}
	}

	onDestroy(() => {
		show = false;
		if (focusTrap) {
			focusTrap.deactivate();
		}
		if (modalElement) {
			document.body.removeChild(modalElement);
		}
	});
</script>

{#if show}
	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<div
		bind:this={modalElement}
		class=" fixed top-0 right-0 left-0 bottom-0 bg-black/60 w-full h-screen max-h-[100dvh] flex justify-center z-99999999 overflow-hidden overscroll-contain"
		in:fade={{ duration: 10 }}
		on:mousedown={() => {
			show = false;
		}}
	>
		<div
			class=" m-auto max-w-full w-[40rem] mx-2 bg-white/95 dark:bg-gray-950/95 backdrop-blur-sm rounded-4xl max-h-[100dvh] shadow-3xl border border-white dark:border-gray-900"
			in:flyAndScale
			on:mousedown={(e) => {
				e.stopPropagation();
			}}
		>
			<div class="px-[1.75rem] py-6 flex flex-col">
				<div class=" text-lg font-medium dark:text-gray-200 mb-2.5">
					{$i18n.t("Choose what you'd like to share")}
				</div>

				<div class="text-sm my-1 px-1 py-0.5 bg-yellow-500/20 border-l-2 border-yellow-500 rounded">
					⚠️ <b>{$i18n.t('Notice: ')}</b> {$i18n.t('You are about to share part of a chat publicly!')}
				</div>
				<div class=" text-sm text-gray-500 flex-1">
					{$i18n.t('Please review the following carefully before proceeding. No data will be sent until your choices are finalized.')}
				</div>

				<slot>

					<form class="mt-4 text-sm">
						<div class="font-medium mb-1">{$i18n.t('User Data')}</div>
						<div class="form-group mb-4 text-xs">
							<label for="id" class="flex flex-row justify-between cursor-pointer">
								{$i18n.t('ID')}
								<select
									id="id"
									name="id"
									required
									class="rounded-full bg-gray-50 dark:bg-gray-900 border border-gray-100 dark:border-gray-850 pl-2 pr-8 h-min-content cursor-pointer"
								>
									<option value="anonymous">{$i18n.t('Anonymous')}</option>
									<option value="pseudonym">{$i18n.t('Pseudonym')}</option>
									<option value="huggingfaceId">{$i18n.t('Huggingface ID')}</option>
								</select>
							</label>
						</div>

						<div class="font-medium mb-1">{$i18n.t('Chat Data')}</div>
						<div class="form-group mb-4 text-xs">
							<label for="assessment" class="flex flex-row justify-between cursor-pointer">
								{$i18n.t('Assessment')}
								<select
									id="assessment"
									name="assessment"
									class="rounded-full bg-gray-50 dark:bg-gray-900 border border-gray-100 dark:border-gray-850 pl-2 pr-8 h-min-content cursor-pointer"
								>
									<option value="good">{$i18n.t('Good')}</option>
									<option value="bad">{$i18n.t('Bad')}</option>
									<option value="mixed">{$i18n.t('Mixed')}</option>
								</select>
							</label>
							<label for="reactions" class="flex flex-row justify-between cursor-pointer">
								{$i18n.t('Include reactions?')}
								<input
									type="checkbox"
									id="reactions"
									name="reactions"
									class="h-3.5 w-3.5 rounded-1"
								/>
							</label>
						</div>

						<div class="font-medium mb-1">{$i18n.t('Additional Notes')}</div>
						<div class="form-group mb-4 text-xs">
							<textarea
								id="notes"
								name="notes"
								rows="3"
								placeholder="{$i18n.t('Any additional information you would like to provide...')}"
								class="w-full rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-100 dark:border-gray-850 p-2 resize-none"
							></textarea>
						</div>

					</form>

				</slot>

				<div class="mt-6 flex justify-between gap-1.5">
					<button
						class="text-sm bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-850 dark:hover:bg-gray-800 dark:text-white font-medium w-full py-2 rounded-3xl transition"
						on:click={() => {
							show = false;
							dispatch('cancel');
						}}
						type="button"
					>
						{cancelLabel}
					</button>
					<button
						class="text-sm bg-gray-900 hover:bg-gray-850 text-gray-100 dark:bg-gray-100 dark:hover:bg-white dark:text-gray-800 font-medium w-full py-2 rounded-3xl transition"
						on:click={() => {
							confirmHandler();
						}}
						type="button"
					>
						{confirmLabel}
					</button>
				</div>
			</div>
		</div>
	</div>
{/if}

<style>
	.modal-content {
		animation: scaleUp 0.1s ease-out forwards;
	}

	@keyframes scaleUp {
		from {
			transform: scale(0.985);
			opacity: 0;
		}
		to {
			transform: scale(1);
			opacity: 1;
		}
	}
</style>
