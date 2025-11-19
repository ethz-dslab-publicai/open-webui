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

	let show_chat_preview_modal = false;
	let expanded = false;

	// Show more texts
	let show_header_info_modal = false;
	let show_more_identification = false;
	let show_more_privacy = false;
	let show_more_usage = false;

	// Main Learn more 
	const HUGGINGFACE_TOKENS_DOC_URL = "https://huggingface.co/docs/hub/en/security-tokens";
	const HUGGINGFACE_TOKENS_SETTINGS_URL = "https://huggingface.co/settings/tokens";
	const DATALICENSES_URL = "https://datalicenses.org";
	const DEFAULT_FAQ_URL = "https://example.com/flywheel-faq";
	const DEFAULT_PRIVACY_POLICY_URL = "https://example.com/privacy";

	let shareFeedbackInfo = `
	<p>
	You can send specific chats to a public repository to share your good, bad, or interesting chats and help build better public AI.
	These chats can be used by anyone, subject to the experimental "AI preference signals" and the formal "licenses" you attach to the chats.
	</p>

	<p>
	By default, your chats are not used directly for R&D. We may compute de-identified aggregate stats (for example, total message volume) to operate the service.
	</p>

	<p>
	You can always delete chats at any time or use temporary mode to ensure chats are not stored or used for any purpose.
	</p>

	<h4 class="font-semibold mt-3">How to set up public sharing:</h4>
	<ol class="list-decimal ml-5 space-y-1">
		<li>Controls (top right) → Valves → Functions → Sharing</li>
		<li>Toggle <strong>"Public Sharing Available"</strong> ON (Green)</li>
		<li>
		Choose how you show up: Anonymous, Deterministic Pseudonym, or your Hugging Face account
		(requires a write token; learn more:
		<a href="${HUGGINGFACE_TOKENS_DOC_URL}" class="text-blue-600 underline" target="_blank">
			Token docs
		</a>).
		</li>
		<li>
		Choose a Data Licensing Intent (declarative). Examples:
		"AI developers who open-source only", "AI developers who contribute back",
		"Public bodies only". Learn more:
		<a href="${DATALICENSES_URL}" class="text-blue-600 underline" target="_blank">
			datalicenses.org
		</a>.
		</li>

		<li>
		Optional: Link your Hugging Face account to author PRs as you.  
		Create a short-lived write token at  
		<a href="${HUGGINGFACE_TOKENS_SETTINGS_URL}" class="text-blue-600 underline" target="_blank">
			Token settings
		</a>.
		</li>

		<li>Close Chat Controls once you're done, then click the "Sharing" button again.</li>
	</ol>

	<p class="mt-4">
	<strong>Data FAQ:</strong>
	<a href="${DEFAULT_FAQ_URL}" class="text-blue-600 underline" target="_blank">${DEFAULT_FAQ_URL}</a>
	<br>
	<strong>Privacy Policy:</strong>
	<a href="${DEFAULT_PRIVACY_POLICY_URL}" class="text-blue-600 underline" target="_blank">${DEFAULT_PRIVACY_POLICY_URL}</a>
	</p>
	`;

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
		show_chat_preview_modal: false,
		expanded: false,
		show_header_info_modal: false,
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
			show_chat_preview_modal,
			expanded,
			show_header_info_modal,
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
			show_chat_preview_modal,
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
		class="fixed inset-0 z-50 flex items-center justify-center"
		in:fade={{ duration: 50 }}
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

		<div
			class="
				ml-auto relative 
				h-auto max-h-[90vh] 
				w-full max-w-sm 
				bg-white dark:bg-gray-850 
				shadow-2xl rounded-xl
				overflow-y-auto 
				m-4
			"
			in:flyAndScale
			on:mousedown={(e) => e.stopPropagation()}
		>

			<div class="overflow-y-auto max-h-[85vh] rounded-xl p-3 space-y-2 pt-2 pb-4">

				<header class="flex items-center justify-between">
					<h2 class="text-xl font-semibold">
						Share Feedback

						<button 
							type="button"
							class="ml-3 text-blue-600 underline text-sm hover:text-blue-800"
							on:click={() => show_header_info_modal = true}
						>
							Learn more
						</button>
					</h2>

					<button on:click={close} class="p-1 text-lg">✕</button>
				</header>

				{#if show_header_info_modal}
				<div
					class="fixed inset-0 z-[60] flex items-center justify-center 
						bg-black/50 backdrop-blur-sm p-4 overflow-hidden"
					on:click={() => (show_header_info_modal = false)}
				>
					<div
						class="bg-white dark:bg-gray-900 rounded-xl shadow-2xl 
							max-w-2xl w-full max-h-[80vh] 
							border border-blue-600/40 overflow-hidden"
						on:click|stopPropagation
					>
						<!-- Header -->
						<div
							class="flex justify-between items-center px-4 py-3
								border-b border-gray-200 dark:border-gray-700"
						>
							<h3 class="text-lg font-semibold">About Public Sharing</h3>
							<button
								class="text-xl px-2 hover:text-red-500"
								on:click={() => (show_header_info_modal = false)}
							>
								✕
							</button>
						</div>

						<!-- Scrollable body (THIS is the real fix) -->
						<div class="p-4 text-[0.9rem] leading-relaxed space-y-3 overflow-y-auto">
							{@html shareFeedbackInfo}
						</div>
					</div>
				</div>
				{/if}

				<main class="p-2 space-y-4 text-[0.80rem] leading-snug">

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
							Tags
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

					<!-- ───────────────────── CHAT PREVIEW BUTTON ───────────────────── -->
					<div class="text-center">
						<button 
							class="px-3 py-1 text-xs rounded-md border border-blue-600 
	       						text-blue-700 dark:text-blue-600
	       						hover:bg-blue-50 dark:hover:bg-blue-900/20 transition"
							on:click={() => show_chat_preview_modal = true}
						>
							Preview shared chats
						</button>
					</div>

					{#if show_chat_preview_modal}
					<div
						class="fixed inset-0 z-[60] flex items-center justify-center 
							bg-black/50 backdrop-blur-sm p-4 overflow-hidden"
						on:click={() => (show_chat_preview_modal = false)}
					>
						<div
							class="bg-white dark:bg-gray-900 rounded-xl shadow-2xl 
								max-w-3xl w-full max-h-[85vh]
								border border-blue-600/40 overflow-hidden"
							on:click|stopPropagation
						>
							<!-- Header -->
							<div class="flex justify-between items-center px-4 py-3 
										border-b border-gray-200 dark:border-gray-700">
								<h3 class="text-lg font-semibold">Preview Shared Chats</h3>
								<button
									class="text-xl px-2 hover:text-red-500"
									on:click={() => (show_chat_preview_modal = false)}
								>
									✕
								</button>
							</div>

							<!-- Scrollable content -->
							<div class="p-4 overflow-y-auto max-h-[75vh] space-y-4">

								{#each interactiveProps?.messages ?? [] as m}
									<div class="p-3 rounded-lg border bg-gray-50 dark:bg-gray-800">
										<div class="font-semibold mb-1 capitalize text-blue-700 dark:text-blue-300">
											{m.role}
										</div>

										<div class="text-sm leading-relaxed whitespace-pre-wrap">
											{typeof m.content === "string"
												? m.content
												: m.content?.text ?? JSON.stringify(m.content)}
										</div>
									</div>
								{/each}

							</div>
						</div>
					</div>
					{/if}

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
		background: #165dfb;
		cursor: pointer;
	}
		/* For Firefox */
	.range-rule::-moz-range-thumb {
		height: 18px;
		width: 18px;
		border-radius: 50%;
		background: #165dfb;
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
