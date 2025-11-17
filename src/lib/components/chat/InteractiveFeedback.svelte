<script lang="ts">
	import { createEventDispatcher, onMount } from 'svelte';
	import { toast } from 'svelte-sonner';

	export let interactiveProps = {}; // { action, messages, model, messageId }
	const dispatch = createEventDispatcher();

	// form state
	// identification: 'anonymous' | 'pseudonym' | 'huggingface'
	let identification_type: 'anonymous' | 'pseudonym' | 'huggingface' = 'anonymous';
	let identification_markdown = '';
	// backward-compatible user_id (optional simple identifier)
	let user_id = '';
    let huggingface_token = '';

	// sharing license / privacy level: 'no_privacy' | 'medium' | 'high'
	let privacy_license: 'no_privacy' | 'medium' | 'high' = 'medium';

	// how the user wants their data to be used
	let usage_model_training = true;
	let usage_qa = true;
	let usage_analytics = true;
	let usage_other = false;
	let usage_other_text = '';

	// ratings 1..10
	let rating_accuracy = 6;
	let rating_consistency = 6;
	let rating_followed = 6;

	let comments = '';
	
    let include_chat_preview = false;
    let expanded = false;


	// prefill from context (optional)
	onMount(() => {
		// interactiveProps may contain suggested defaults
		const payload = interactiveProps?.action?.payload ?? {};
		if (payload.default_identification) identification_type = payload.default_identification;
		if (payload.default_privacy_license) privacy_license = payload.default_privacy_license;
		if (payload.default_ratings) {
			rating_accuracy = payload.default_ratings.accuracy ?? rating_accuracy;
			rating_consistency = payload.default_rating_consistency ?? rating_consistency;
			rating_followed = payload.default_ratings.followed ?? rating_followed;
		}
		// prefill user id from local storage or auth
		const stored = localStorage.getItem('user_id');
		if (stored) user_id = stored;
	});

	const getChatSnippet = (messages: any[], maxChars = 400) => {
		if (!messages) return null;
		try {
			const text = messages
				.map((m) => {
					const role = m.role ?? m.author ?? '';
					const content = m.content?.text ?? m.text ?? m.content ?? '';
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
		// Build payload
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
				followed_instructions: rating_followed
			},
			comments,
			created_at: new Date().toISOString(),
			action_id: interactiveProps?.action?.id ?? 'share_feedback',
			model: interactiveProps?.model ?? null,
			message_id: interactiveProps?.messageId ?? null,
			chat_preview: include_chat_preview ? interactiveProps?.messages ?? null : null,
			chat_snippet: include_chat_preview ? getChatSnippet(interactiveProps?.messages ?? []) : null
		};

		// optimistic UI: show loading toast
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
			// error is shown by toast.promise already
			console.error('submit feedback failed', err);
		}
	};
</script>

<div class="fixed inset-0 z-50 flex">
	<div
        class="absolute inset-0 bg-black/40"
        on:click={close}
        aria-hidden="true">
    </div>

	<div class="ml-auto relative h-full w-full max-w-md bg-white dark:bg-gray-850 shadow-xl overflow-y-auto">

        <div class="p-4 space-y-2 pt-14 pb-35">

            <header class="flex items-center justify-between">
                <h2 class="text-base font-medium">Share Feedback</h2>
                <button on:click={close} class="p-1 text-lg">✕</button>
            </header>

            <main class="p-2 space-y-4">

                <!-- Identification -->
                <div class="border rounded-lg p-4 pt-2 bg-gray-50 dark:bg-gray-800/50">
                    <h3 class="text-sm font-semibold">Identification</h3>

                    <div class="flex items-center pt-1">
                        <select bind:value={identification_type} class="rounded border p-2 w-full">
                            <option value="anonymous">Anonymous</option>
                            <option value="huggingface">Hugging Face token</option>
                            <option value="pseudonym">Pseudonym</option>
                        </select>
                    </div>

                    {#if identification_type === 'pseudonym'}
                        <div>
                            <input class="mt-2 w-full rounded border p-2" bind:value={user_id} placeholder="Enter your pseudonym" />
                        </div>
                    {/if}

                    {#if identification_type === 'huggingface'}
                        <div>
                            <input class="mt-2 w-full rounded border p-2" bind:value={huggingface_token} placeholder="Enter your Hugging Face token" />
                        </div>
                    {/if}
                </div>

                <!-- Privacy -->
                <div class="border rounded-lg p-4 pt-2 bg-gray-50 dark:bg-gray-800/50">
                    <h3 class="text-sm font-semibold">Privacy level</h3>

                    <div class="pt-1">
                        <select bind:value={privacy_license} class="rounded border p-2 w-full">
                            <option value="high">High privacy — maximum protection</option>
                            <option value="medium">Medium privacy — limited sharing</option>
                            <option value="no_privacy">No privacy — share everything</option>
                        </select>
                    </div>
                </div>

                <!-- Ratings -->
                <div class="border rounded-lg p-4 pt-2 bg-gray-50 dark:bg-gray-800/50">
                    <h3 class="text-sm font-semibold">Chat ratings (1–10)</h3>

                    <div class="pt-1 space-y-4">

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

                        <!-- Following instructions -->
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

                <!-- Data Usage -->
                <div class="border rounded-lg p-4 pt-2 bg-gray-50 dark:bg-gray-800/50">
                    <h3 class="text-sm font-semibold">How may we use your feedback?</h3>

                    <div class="flex flex-col gap-2 pt-1 text-sm">
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

                        <!-- "Other" input -->
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

                <!-- Additional Comments -->
                <div class="border rounded-lg p-4 pt-2 bg-gray-50 dark:bg-gray-800/50">
                    <h3 class="text-sm pb-1 font-semibold">Additional Comments</h3>
                    <textarea 
                        rows="2"
                        bind:value={comments}
                        class="w-full mt-2 p-2 rounded border"
                        placeholder="Anything else to share? (optional)"
                    ></textarea>
                </div>


                <!-- Chat Preview -->
                <div class="border rounded-lg p-2 pt-2 bg-gray-50 dark:bg-gray-800/50">
                    <div class="flex items-center gap-2">
                        <input id="includePreview" type="checkbox" bind:checked={include_chat_preview} />
                        <label for="includePreview" class="text-sm font-semibold">Preview shared chats</label>
                    </div>

                    {#if include_chat_preview}
                        <div class="chat-preview-container">
                            
                            <!-- Expand / collapse -->
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

            <footer class="flex justify-end gap-55 mt-2 pt-5 border-t dark:border-gray-800">
                <button class="px-2 py-2 rounded-lg border border-gray-300 hover:bg-gray-100 transition" on:click={close}>Cancel</button>
                <button class="px-2 py-2 rounded-lg bg-blue-600 text-white text-sm hover:bg-blue-700 transition" on:click={submit}>Submit Feedback</button>
            </footer>
        
        </div>
	</div>
</div>

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
