<script lang="ts">
	import { createEventDispatcher, onMount } from 'svelte';
	import { toast } from 'svelte-sonner';

	export let interactiveProps = {}; // { action, messages, model, messageId }
	const dispatch = createEventDispatcher();

	// identification: 'anonymous' | 'pseudonym' | 'huggingface'
	let identification_type: 'anonymous' | 'pseudonym' | 'huggingface' = 'anonymous';
	let identification_markdown = '';
	let user_id = '';
	let huggingface_token = '';

	// privacy: 'no_privacy' | 'medium' | 'high'
	let privacy_license: 'no_privacy' | 'medium' | 'high' = 'medium';

	// ratings 1..10
	let rating_accuracy = 5;
	let rating_consistency = 5;
	let rating_followed = 5;

	// rating tags
	let rating_tags: string[] = [];
	const available_rating_tags = [
		'Relevant answer',
		'Too short',
		'Too long / wordy',
		'Helpful reasoning',
		'Poor reasoning',
		'Incorrect',
		'Partially correct',
		'Unclear',
		'Off-topic'
	];

	// usage
	let usage_model_training = true;
	let usage_qa = true;
	let usage_analytics = true;
	let usage_other = false;
	let usage_other_text = '';

	let comments = '';

	// header "Learn more"
	let show_header_info_modal = false;

	// Main "Learn more" content and URLs
	const HUGGINGFACE_TOKENS_DOC_URL = 'https://huggingface.co/docs/hub/en/security-tokens';
	const HUGGINGFACE_TOKENS_SETTINGS_URL = 'https://huggingface.co/settings/tokens';
	const DATALICENSES_URL = 'https://datalicenses.org';
	const DEFAULT_FAQ_URL = 'https://example.com/flywheel-faq';
	const DEFAULT_PRIVACY_POLICY_URL = 'https://example.com/privacy';

	let shareFeedbackInfo = `
	<p>
	You can send specific chats to a public repository to share your good, bad, or interesting chats and help build better public AI.
	These chats can be used by anyone, subject to the experimental "AI preference signals" and the formal "licenses" you attach to the chats.
	</p>

	<p>
	By default, your chats are not used directly for R&amp;D. We may compute de-identified aggregate stats (for example, total message volume) to operate the service.
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
		<a href="${HUGGINGFACE_TOKENS_DOC_URL}" class="text-blue-600 underline" target="_blank" rel="noreferrer">
			Token docs
		</a>).
		</li>
		<li>
		Choose a Data Licensing Intent (declarative). Examples:
		"AI developers who open-source only", "AI developers who contribute back",
		"Public bodies only". Learn more:
		<a href="${DATALICENSES_URL}" class="text-blue-600 underline" target="_blank" rel="noreferrer">
			datalicenses.org
		</a>.
		</li>

		<li>
		Optional: Link your Hugging Face account to author PRs as you.  
		Create a short-lived write token at  
		<a href="${HUGGINGFACE_TOKENS_SETTINGS_URL}" class="text-blue-600 underline" target="_blank" rel="noreferrer">
			Token settings
		</a>.
		</li>

		<li>Close Chat Controls once you're done, then click the "Sharing" button again.</li>
	</ol>

	<p class="mt-4">
	<strong>Data FAQ:</strong>
	<a href="${DEFAULT_FAQ_URL}" class="text-blue-600 underline" target="_blank" rel="noreferrer">${DEFAULT_FAQ_URL}</a>
	<br>
	<strong>Privacy Policy:</strong>
	<a href="${DEFAULT_PRIVACY_POLICY_URL}" class="text-blue-600 underline" target="_blank" rel="noreferrer">${DEFAULT_PRIVACY_POLICY_URL}</a>
	</p>
	`;

	// derive messages once (tries a few shapes, just in case)
	$: chatMessages =
		(interactiveProps as any)?.messages ??
		(interactiveProps as any)?.chat?.messages ??
		[];

	// prefill from context (optional)
	onMount(() => {
		const payload = (interactiveProps as any)?.action?.payload ?? {};
		if (payload.default_identification) identification_type = payload.default_identification;
		if (payload.default_privacy_license) privacy_license = payload.default_privacy_license;
		if (payload.default_ratings) {
			rating_accuracy = payload.default_ratings.accuracy ?? rating_accuracy;
			// keeping original key for backward compatibility if it exists
			rating_consistency = payload.default_rating_consistency ?? rating_consistency;
			rating_followed = payload.default_ratings.followed ?? rating_followed;
		}
		const stored = localStorage.getItem('user_id');
		if (stored) user_id = stored;
	});

	const getChatSnippet = (messages: any[], maxChars = 400) => {
		if (!messages) return null;
		try {
			const text = messages
				.map((m) => {
					const role = m.role ?? (m as any).author ?? '';
					const content = (m as any).content?.text ?? (m as any).text ?? (m as any).content ?? '';
					return (role ? `${role}: ` : '') + (typeof content === 'string' ? content : JSON.stringify(content));
				})
				.join('\n\n');
			return text.length > maxChars ? text.slice(0, maxChars) + '…' : text;
		} catch (e) {
			return JSON.stringify(messages).slice(0, maxChars) + '…';
		}
	};

	const close = () => {
		dispatch('close');
	};

	const submit = async () => {
		const payload = {
			identification_type,
			identification_markdown: identification_type === 'anonymous' ? null : identification_markdown || null,
			user_id: identification_type === 'anonymous' ? null : user_id || null,
			privacy_license,
			data_usage: {
				model_training: usage_model_training,
				qa: usage_qa,
				analytics: usage_analytics,
				other: usage_other ? usage_other_text || null : null
			},
			ratings: {
				accuracy: rating_accuracy,
				consistency: rating_consistency,
				followed_instructions: rating_followed,
				tags: rating_tags
			},
			comments,
			created_at: new Date().toISOString(),
			action_id: (interactiveProps as any)?.action?.id ?? 'share_feedback',
			model: (interactiveProps as any)?.model ?? null,
			message_id: (interactiveProps as any)?.messageId ?? null,
		};

		const t = toast.promise(
			(async () => {
				const res = await fetch('/api/feedback', {
					method: 'POST',
					headers: { 'Content-Type': 'application/json' },
					body: JSON.stringify(payload)
				});
				if (!res.ok) {
					const text = await res.text();
					throw new Error(text || `Server returned ${res.status}`);
				}
				return res.json();
			})(),
			{
				loading: 'Submitting feedback...',
				success: 'Thanks — feedback submitted.',
				error: (err) => `Failed to submit: ${err.message}`
			},
			{ duration: 3000 }
		);

		try {
			await t;
			dispatch('submit', { payload });
			close();
		} catch (err) {
			console.error('submit feedback failed', err);
		}
	};
</script>

<!-- Feedback modal -->
<div
	bind:this={modalElement}
	class="fixed inset-0 z-50 flex items-start justify-end"
	in:fade={{ duration: 50 }}
	on:click={() => (show = false)}
>
	<div
		class="absolute inset-0 bg-black/40 backdrop-blur-sm"
		aria-hidden="true"
	></div>

	<div
		class="
			relative 
			h-auto max-h-[90vh]
			w-full max-w-sm 
			bg-white dark:bg-gray-850 
			shadow-2xl rounded-xl
			overflow-hidden 
			m-4
		"
		in:flyAndScale
		on:click|stopPropagation
	>
    <div class="overflow-y-auto max-h-[85vh] p-3 space-y-2 pt-2 pb-4">
			<!-- Header row -->
			<header class="flex items-center justify-between">
				<h2 class="text-xl font-semibold">
					Share Feedback

					<button
						type="button"
						class="ml-3 text-blue-600 underline text-sm hover:text-blue-800"
						on:click={() => (show_header_info_modal = true)}
					>
						Learn more
					</button>
				</h2>

				<button on:click={close} class="p-1 text-lg">✕</button>
			</header>

			<!-- MAIN LEARN MORE MODAL -->
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

						<!-- Scrollable body -->
						<div class="p-4 text-[0.9rem] leading-relaxed space-y-3 overflow-y-auto">
							{@html shareFeedbackInfo}
						</div>
					</div>
				</div>
			{/if}

			<main class="p-2 space-y-4 text-[0.80rem] leading-snug">
				<!-- IDENTIFICATION -->
				<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
					<h3 class="text-sm font-semibold">Identification</h3>

					<div class="pt-2">
						<select bind:value={identification_type} class="rounded border p-2 w-full">
							<option value="anonymous">Anonymous</option>
							<option value="huggingface">Hugging Face token</option>
							<option value="pseudonym">Pseudonym</option>
						</select>
					</div>

					{#if identification_type === 'pseudonym'}
						<input
							class="mt-2 w-full rounded border p-2"
							bind:value={user_id}
							placeholder="Your pseudonym"
						/>
					{/if}

					{#if identification_type === 'huggingface'}
						<input
							class="mt-2 w-full rounded border p-2"
							bind:value={huggingface_token}
							placeholder="Your Hugging Face token"
						/>
					{/if}
				</div>

				<!-- PRIVACY -->
				<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
					<h3 class="text-sm font-semibold">Privacy level</h3>

					<select bind:value={privacy_license} class="rounded border p-2 mt-2 w-full">
						<option value="high">High privacy — maximum protection</option>
						<option value="medium">Medium privacy — limited sharing</option>
						<option value="no_privacy">No privacy — fully shareable</option>
					</select>
				</div>

				<!-- RATINGS -->
				<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
					<h3 class="text-sm font-semibold">Chat ratings (1–10)</h3>

					<div class="pt-3 space-y-4">
						<!-- Accuracy -->
						<div class="flex items-center gap-3">
							<div class="w-40 text-sm">Accuracy</div>
							<input
								type="range"
								min="1"
								max="10"
								step="1"
								bind:value={rating_accuracy}
								class="flex-1 range-rule"
							/>
							<div class="w-8 text-right text-sm">{rating_accuracy}</div>
						</div>

						<!-- Consistency -->
						<div class="flex items-center gap-3">
							<div class="w-40 text-sm">Consistency</div>
							<input
								type="range"
								min="1"
								max="10"
								step="1"
								bind:value={rating_consistency}
								class="flex-1 range-rule"
							/>
							<div class="w-8 text-right text-sm">{rating_consistency}</div>
						</div>

						<!-- Followed Instructions -->
						<div class="flex items-center gap-3">
							<div class="w-40 text-sm">Followed Instructions</div>
							<input
								type="range"
								min="1"
								max="10"
								step="1"
								bind:value={rating_followed}
								class="flex-1 range-rule"
							/>
							<div class="w-8 text-right text-sm">{rating_followed}</div>
						</div>
					</div>
				</div>

				<!-- RATING TAGS -->
				<div class="mt-1">
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
										? rating_tags.filter((t) => t !== tag)
										: [...rating_tags, tag];
								}}
							>
								{tag}
							</button>
						{/each}
					</div>
				</div>

				<!-- DATA USAGE -->
				<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
					<h3 class="text-sm font-semibold">How may we use your feedback?</h3>

					<div class="flex flex-col gap-2 pt-2 text-sm">
						<label>
							<input type="checkbox" bind:checked={usage_model_training} />
							<span class="ml-2">Model training — most helpful</span>
						</label>

						<label>
							<input type="checkbox" bind:checked={usage_qa} />
							<span class="ml-2">QA / internal review</span>
						</label>

						<label>
							<input type="checkbox" bind:checked={usage_analytics} />
							<span class="ml-2">Analytics / metrics</span>
						</label>

						<label class="flex items-center gap-2">
							<input type="checkbox" bind:checked={usage_other} />
							<span class="ml-2">Other</span>

							{#if usage_other}
								<input
									class="ml-2 rounded border p-2 flex-1"
									bind:value={usage_other_text}
									placeholder="Describe"
								/>
							{/if}
						</label>
					</div>
				</div>

				<!-- COMMENTS -->
				<div class="border rounded-lg p-4 bg-gray-50 dark:bg-gray-800/50">
					<h3 class="text-sm font-semibold pb-1">Additional Comments</h3>
					<textarea
						rows="2"
						bind:value={comments}
						class="w-full mt-2 p-2 rounded border"
						placeholder="Anything else to share? (optional)"
					></textarea>
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

<style>
	/* slider */
	.range-rule {
		appearance: none;
		width: 100%;
		height: 6px;
		border-radius: 4px;
		background:
			linear-gradient(#e5e7eb, #e5e7eb) content-box,
			repeating-linear-gradient(
				to right,
				#9ca3af 0 2px,
				transparent 0 calc(10% - 2px)
			);
		background-size: 100% 6px, 100% 6px;
		background-repeat: no-repeat;
		padding: 0;
	}

	.range-rule::-webkit-slider-thumb {
		appearance: none;
		height: 18px;
		width: 18px;
		border-radius: 50%;
		background: #165dfb;
		cursor: pointer;
	}

	.range-rule::-moz-range-thumb {
		height: 18px;
		width: 18px;
		border-radius: 50%;
		background: #165dfb;
		cursor: pointer;
	}

</style>