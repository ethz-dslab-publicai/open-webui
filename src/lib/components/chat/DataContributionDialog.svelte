<script lang="ts">
	import { onMount, getContext, createEventDispatcher, onDestroy, tick } from 'svelte';
	import * as FocusTrap from 'focus-trap';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	import { fade } from 'svelte/transition';
	import { flyAndScale } from '$lib/utils/transitions';

	export let cancelLabel = $i18n?.t?.('Cancel') ?? 'Cancel';
	export let confirmLabel = $i18n?.t?.('Submit') ?? 'Submit';

	export let onConfirm: (data?: any) => Promise<any> | void = () => {};

	export let show = false;
	export let interactiveProps: any = {};

	// Form state
	let identification_type: string = 'anonymous';
	let user_id: string = '';
	let huggingface_token: string = '';

	let privacy_license: string = 'medium';

	let rating_accuracy = 5;
	let rating_consistency = 5;
	let rating_followed = 5;

	let rating_tags = [];
	const available_rating_tags = [
		"Relevant answer",
		"Too short",
		"Too long / wordy",
		"Helpful reasoning",
		"Poor reasoning",
		"Incorrect",
		"Partially correct",
		"Unclear",
		"Off-topic"
	];

	let usage_model_training = true;
	let usage_qa = true;
	let usage_analytics = true;
	let usage_other = false;
	let usage_other_text = '';

	let comments = '';

	let include_chat_preview = false;
	let expanded = false;

	// Show more texts
	let show_more_header = false;
	let show_more_identification = false;
	let show_more_privacy = false;
	let show_more_usage = false;

	// Default form values (used by `init()` to reset the form)
	const DEFAULTS = {
		identification_type: 'anonymous',
		user_id: '',
		huggingface_token: '',
		privacy_license: 'medium',
		rating_accuracy: 5,
		rating_consistency: 5,
		rating_followed: 5,
		rating_tags: [],
		usage_model_training: true,
		usage_qa: true,
		usage_analytics: true,
		usage_other: false,
		usage_other_text: '',
		comments: '',
		include_chat_preview: false,
		expanded: false,
		show_more_header: false,
		show_more_identification: false,
		show_more_privacy: false,
		show_more_usage: false,
	};


	$: if (show) {
		init();
	}

	let modalElement: HTMLElement | null = null;
	let mounted = false;

	let focusTrap: FocusTrap.FocusTrap | null = null;

	const init = () => {
		// reset all form fields to the canonical defaults
		({
			identification_type,
			user_id,
			huggingface_token,
			privacy_license,
			rating_accuracy,
			rating_consistency,
			rating_followed,
			usage_model_training,
			usage_qa,
			usage_analytics,
			usage_other,
			usage_other_text,
			comments,
			include_chat_preview,
			expanded,
			show_more_header,
		} = { ...DEFAULTS });
	};

	const handleKeyDown = (event: KeyboardEvent) => {
		if (event.key === 'Escape') {
			show = false;
			dispatch('cancel');
		}
	};

	const close = () => {
		show = false;
		dispatch('cancel');
	};

	const submit = async () => {
		const data = {
			identification_type,
			user_id,
			huggingface_token,
			privacy_license,
			rating_accuracy,
			rating_consistency,
			rating_followed,
			rating_tags,
			usage: {
				model_training: usage_model_training,
				qa: usage_qa,
				analytics: usage_analytics,
				other: usage_other ? usage_other_text : null,
			},
			comments,
			include_chat_preview,
			interactive: interactiveProps,
		};

		show = false;
		await tick();
		try {
			await onConfirm?.(data);
		} catch (err) {
			console.error('onConfirm handler failed', err);
		}
		dispatch('confirm', data);
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
			if (focusTrap) focusTrap.deactivate();

			window.removeEventListener('keydown', handleKeyDown);
			if (document.body.contains(modalElement)) {
				document.body.removeChild(modalElement);
			}

			document.body.style.overflow = 'unset';
		}
	}

	onDestroy(() => {
		show = false;
		if (focusTrap) {
			focusTrap.deactivate();
		}
		if (modalElement && document.body.contains(modalElement)) {
			document.body.removeChild(modalElement);
		}
	});
</script>

{#if show}
	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<div
		bind:this={modalElement}
		class="fixed inset-0 z-50 flex"
		in:fade={{ duration: 10 }}
		on:mousedown={() => {
			// clicking outside closes
			show = false;
		}}
	>
		<div
			class="absolute inset-0 bg-black/40"
			on:click={close}
			aria-hidden="true"
		></div>

		<div class="ml-auto relative h-full w-full max-w-md bg-white dark:bg-gray-850 shadow-xl overflow-y-auto" in:flyAndScale on:mousedown={(e) => e.stopPropagation()}>

			<div class="p-4 space-y-2 pt-4 pb-6">

				<header class="flex items-center justify-between">
					<h2 class="text-xl font-semibold">
						Share Feedback

						<button 
							type="button"
							class="ml-3 text-blue-600 underline text-sm hover:text-blue-800"
							on:click={() => show_more_header = !show_more_header}
						>
							{show_more_header ? "Hide" : "Learn more"}
						</button>
					</h2>

					<button on:click={close} class="p-1 text-lg">✕</button>
				</header>

				{#if show_more_header}
					<div transition:fade={{ duration: 150 }} class="mt-0 text-sm text-gray-600 dark:text-gray-300">
						This feedback helps us improve response quality and model training.
						We never expose your private data, and you control how it is used.
					</div>
				{/if}

				<main class="p-2 space-y-4">

					<!-- ───────────────────── IDENTIFICATION ───────────────────── -->
					<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
						<div class="flex justify-between items-center">
							<h3 class="text-sm font-semibold">Identification</h3>

							<button
								class="text-blue-600 underline text-xs hover:text-blue-800"
								on:click={() => show_more_identification = !show_more_identification}
							>
								{show_more_identification ? "Hide" : "Learn more"}
							</button>
						</div>

						{#if show_more_identification}
							<div class="text-xs text-gray-600 dark:text-gray-400 mt-1" transition:fade={{duration:120}}>
								Choose how (or whether) to identify yourself. Anonymous feedback is fully allowed.
							</div>
						{/if}

						<div class="pt-2">
							<select bind:value={identification_type} class="rounded border p-2 w-full">
								<option value="anonymous">Anonymous</option>
								<option value="huggingface">Hugging Face token</option>
								<option value="pseudonym">Pseudonym</option>
							</select>
						</div>

						{#if identification_type === 'pseudonym'}
							<input class="mt-2 w-full rounded border p-2" bind:value={user_id} placeholder="Your pseudonym" />
						{/if}

						{#if identification_type === 'huggingface'}
							<input class="mt-2 w-full rounded border p-2" bind:value={huggingface_token} placeholder="Your Hugging Face token" />
						{/if}
					</div>

					<!-- ───────────────────── PRIVACY ───────────────────── -->
					<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
						<div class="flex justify-between items-center">
							<h3 class="text-sm font-semibold">Privacy level</h3>

							<button
								class="text-blue-600 underline text-xs hover:text-blue-800"
								on:click={() => show_more_privacy = !show_more_privacy}
							>
								{show_more_privacy ? "Hide" : "Learn more"}
							</button>
						</div>

						{#if show_more_privacy}
							<div class="text-xs text-gray-600 dark:text-gray-400 mt-1" transition:fade={{duration:120}}>
								Control how much of your feedback can be stored or shared. “High privacy” shares the least.
							</div>
						{/if}

						<select bind:value={privacy_license} class="rounded border p-2 mt-2 w-full">
							<option value="high">High privacy — maximum protection</option>
							<option value="medium">Medium privacy — limited sharing</option>
							<option value="no_privacy">No privacy — fully shareable</option>
						</select>
					</div>

					<!-- ───────────────────── RATINGS ───────────────────── -->
					<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
						<div class="flex justify-between items-center">
							<h3 class="text-sm font-semibold">Chat ratings (1–10)</h3>
						</div>

						<div class="pt-3 space-y-4">
							<!-- Accuracy -->
							<div class="flex items-center gap-3">
								<div class="w-40 text-sm">Accuracy</div>
								<input type="range" min="1" max="10" step="1" bind:value={rating_accuracy} class="flex-1 range-rule"/>
								<div class="w-8 text-right text-sm">{rating_accuracy}</div>
							</div>

							<!-- Consistency -->
							<div class="flex items-center gap-3">
								<div class="w-40 text-sm">Consistency</div>
								<input type="range" min="1" max="10" step="1" bind:value={rating_consistency} class="flex-1 range-rule"/>
								<div class="w-8 text-right text-sm">{rating_consistency}</div>
							</div>

							<!-- Followed Instructions -->
							<div class="flex items-center gap-3">
								<div class="w-40 text-sm">Followed Instructions</div>
								<input type="range" min="1" max="10" step="1" bind:value={rating_followed} class="flex-1 range-rule"/>
								<div class="w-8 text-right text-sm">{rating_followed}</div>
							</div>
						</div>
					</div>

					<!-- Rating tags -->
					<div class="mt-4">
						<h4 class="text-xs font-semibold text-gray-700 dark:text-gray-300 mb-2">
							Optional tags
						</h4>

						<div class="flex flex-wrap gap-2">
							{#each available_rating_tags as tag}
								<button
									type="button"
									class="px-2 py-1 rounded-full text-xs border transition
										{rating_tags.includes(tag)
											? 'bg-blue-600 text-white border-blue-600'
											: 'bg-white dark:bg-gray-700 border-gray-300 dark:border-gray-600 text-gray-700 dark:text-gray-300'}"
									on:click={() => {
										rating_tags = rating_tags.includes(tag)
											? rating_tags.filter(t => t !== tag)
											: [...rating_tags, tag];
									}}
								>
									{tag}
								</button>
							{/each}
						</div>
					</div>


					<!-- ───────────────────── DATA USAGE ───────────────────── -->
					<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
						<div class="flex justify-between items-center">
							<h3 class="text-sm font-semibold">How may we use your feedback?</h3>

							<button
								class="text-blue-600 underline text-xs hover:text-blue-800"
								on:click={() => show_more_usage = !show_more_usage}
							>
								{show_more_usage ? "Hide" : "Learn more"}
							</button>
						</div>

						{#if show_more_usage}
							<div class="text-xs text-gray-600 dark:text-gray-400 mt-1" transition:fade={{duration:120}}>
								Choosing “Model training” is the most helpful option, as it allows future improvements.
							</div>
						{/if}

						<div class="flex flex-col gap-2 pt-2 text-sm">
							<label><input type="checkbox" bind:checked={usage_model_training}/> <span class="ml-2">Model training — most helpful</span></label>
							<label><input type="checkbox" bind:checked={usage_qa}/> <span class="ml-2">QA / internal review</span></label>
							<label><input type="checkbox" bind:checked={usage_analytics}/> <span class="ml-2">Analytics / metrics</span></label>

							<label class="flex items-center gap-2">
								<input type="checkbox" bind:checked={usage_other} />
								<span class="ml-2">Other</span>

								{#if usage_other}
									<input class="ml-2 rounded border p-2 flex-1" bind:value={usage_other_text} placeholder="Describe"/>
								{/if}
							</label>
						</div>
					</div>

					<!-- ───────────────────── COMMENTS ───────────────────── -->
					<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
						<h3 class="text-sm font-semibold pb-1">Additional Comments</h3>
						<textarea rows="2" bind:value={comments} class="w-full mt-2 p-2 rounded border" placeholder="Anything else to share? (optional)"></textarea>
					</div>

					<!-- ───────────────────── CHAT PREVIEW ───────────────────── -->
					<div class="border rounded-lg p-2 pt-2 bg-gray-50 dark:bg-gray-800/50">
						
						<div class="flex justify-between items-center">
							<label for="includePreview" class="text-sm font-semibold flex items-center gap-2">
								<input id="includePreview" type="checkbox" bind:checked={include_chat_preview}/>
								Preview shared chats
							</label>
						</div>

						{#if include_chat_preview}
							<div class="chat-preview-container mt-2">
								<button class="expand-btn" on:click={() => expanded = !expanded}>
									{expanded ? "Collapse" : "Expand"} preview
								</button>

								<div class="chat-preview" class:expanded={expanded}>
									{#each interactiveProps?.messages ?? [] as m}
										<div class={`bubble ${m.role}`}>
											<span class="bubble-role">{m.role}:</span>
											<span class="bubble-text">
												{typeof m.content === "string"
													? m.content
													: m.content?.text ?? JSON.stringify(m.content)}
											</span>
										</div>
									{/each}
								</div>
							</div>
						{/if}
					</div>

				</main>


				<footer class="flex justify-between mt-2 pt-4 border-t dark:border-gray-800">
					<button
						class="px-2 py-1 text-xs rounded-md border border-gray-300 hover:bg-gray-100 transition"
						on:click={close}
					>
						Cancel
					</button>

					<button
						class="px-2 py-1 text-xs rounded-md bg-blue-600 text-white hover:bg-blue-700 transition"
						on:click={submit}
					>
						Submit Feedback
					</button>
				</footer>
            
			</div>
		</div>
	</div>
{/if}

<style>
	/* 1–10 slider */
	.range-rule {
		appearance: none;
		width: 100%;
		height: 6px;
		border-radius: 4px;
		background:
			linear-gradient(#e5e7eb, #e5e7eb) content-box,
			repeating-linear-gradient(
				to right,
				#9ca3af 0 2px,          /* tick mark color + thickness */
				transparent 0 calc(10% - 2px) /* spacing for 10 steps */
			);
		background-size: 100% 6px, 100% 6px;
		background-repeat: no-repeat;
		padding: 0;
	}

	/* Slider thumb */
	.range-rule::-webkit-slider-thumb {
		appearance: none;
		height: 18px;
		width: 18px;
		border-radius: 50%;
		background: #3b82f6;
		cursor: pointer;
	}
		/* For Firefox */
	.range-rule::-moz-range-thumb {
		height: 18px;
		width: 18px;
		border-radius: 50%;
		background: #3b82f6;
		cursor: pointer;
	}

	.chat-preview-container {
		margin-top: 10px;
	}

	.chat-preview {
		max-height: 180px;        /* collapsed mode */
		overflow-y: auto;
		padding: 8px;
		border-radius: 8px;
		border: 1px solid #e5e7eb;
		transition: max-height 0.25s ease;
	}

	.dark .chat-preview {
		border-color: #374151;
		background: none;   /* IMPORTANT */
	}

	.chat-preview.expanded {
		max-height: 400px;        /* expanded mode */
	}

	.expand-btn {
		font-size: 0.75rem;
		margin-bottom: 6px;
		color: #3b82f6;
		cursor: pointer;
	}

	.bubble {
	margin-bottom: 15px;
	padding: 8px 12px;
	border-radius: 8px;
	max-width: 90%;
	white-space: pre-wrap; /* keeps real line breaks inside messages */
	}

	/* Role + text on the same line */
	.bubble-role,
	.bubble-text {
		margin: 0;
		padding: 0;
		line-height: 1.2;
	}

	.bubble-role {
		font-weight: 700;
		opacity: 0.9;
		margin-right: 0.25rem; /* tiny horizontal gap */
	}

	/* ASSISTANT bubble (right) */
	.bubble.assistant {
		background: #fef9c3;
		color: #854d0e;
		align-self: flex-end;
	}

	/* USER bubble (left) */
	.bubble.user {
		background: #dbeafe;
		color: #1e3a8a;
		align-self: flex-start;
	}

</style>
