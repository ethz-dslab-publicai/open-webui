<script lang="ts">
	import { createEventDispatcher, onMount } from 'svelte';
	import { toast } from 'svelte-sonner';

	export let interactiveProps = {}; // { action, messages, model, messageId }
	const dispatch = createEventDispatcher();

	// form state
	let anonymous = true;
	let user_id = '';
	let share_identification = false;
	let privacy_level = 'default'; // default / low / high
	let rating = 5;
	let comments = '';
	let include_chat_preview = true;

	// prefill from context (optional)
	onMount(() => {
		// interactiveProps may contain a suggested prompt or defaults
		if (interactiveProps?.action?.payload?.default_anonymous !== undefined) {
			anonymous = !!interactiveProps.action.payload.default_anonymous;
		}
		// if you want to prefill user id from local storage or auth:
		const stored = localStorage.getItem('user_id');
		if (stored) user_id = stored;
	});

	const close = () => {
		dispatch('close');
	};

	const submit = async () => {
		// Build payload
		const payload = {
			anonymous,
			user_id: anonymous ? null : user_id || null,
			share_identification,
			privacy_level,
			rating,
			comments,
			created_at: new Date().toISOString(),
			action_id: interactiveProps?.action?.id ?? 'share_feedback',
			model: interactiveProps?.model ?? null,
			message_id: interactiveProps?.messageId ?? null,
			chat_preview: include_chat_preview ? interactiveProps?.messages ?? null : null
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

<div class="fixed inset-0 z-50 flex items-center justify-center">
	<div class="absolute inset-0 bg-black/40" on:click={close} aria-hidden="true"></div>

	<div class="relative w-[94%] max-w-lg bg-white dark:bg-gray-850 rounded-lg shadow-xl overflow-hidden">
		<header class="flex items-center justify-between px-4 py-2 border-b dark:border-gray-800">
			<div class="font-medium">Share Feedback</div>
			<button class="px-2 py-1" on:click={close} aria-label="Close">✕</button>
		</header>

		<main class="p-4 space-y-3">
			<div class="flex items-center gap-2">
				<input id="anonymous" type="checkbox" bind:checked={anonymous} />
				<label for="anonymous" class="text-sm">Submit anonymously</label>
			</div>

			{#if !anonymous}
				<div>
					<label class="text-sm">Your identifier (optional)</label>
					<input class="mt-1 w-full rounded border p-2" bind:value={user_id} placeholder="email or username" />
					<div class="text-xs text-gray-500">Provide this only if you want follow-up.</div>
				</div>
			{/if}

			<div>
				<label class="text-sm">Privacy settings</label>
				<div class="flex gap-2 mt-1">
					<button class="px-3 py-1 rounded border" on:click={() => (privacy_level = 'default')}>Default</button>
					<button class="px-3 py-1 rounded border" on:click={() => (privacy_level = 'low')}>Low</button>
					<button class="px-3 py-1 rounded border" on:click={() => (privacy_level = 'high')}>High</button>
					<span class="ml-2 text-xs text-gray-500">Selected: {privacy_level}</span>
				</div>
			</div>

			<div>
				<label class="text-sm">Rate this chat</label>
				<div class="flex items-center gap-2 mt-1">
					{#each [1,2,3,4,5] as s}
						<button class="px-2 py-1 rounded" class:selected={s === rating} on:click={() => (rating = s)}>{s}★</button>
					{/each}
				</div>
			</div>

			<div>
				<label class="text-sm">Comments</label>
				<textarea rows="4" bind:value={comments} class="w-full mt-1 p-2 rounded border" placeholder="Anything to share?"></textarea>
			</div>

			<div class="flex items-center gap-2">
				<input id="includePreview" type="checkbox" bind:checked={include_chat_preview} />
				<label for="includePreview" class="text-sm">Include chat preview</label>
			</div>
		</main>

		<footer class="flex gap-2 justify-end p-3 border-t dark:border-gray-800">
			<button class="px-3 py-1 rounded bg-gray-100" on:click={close}>Cancel</button>
			<button class="px-4 py-1 rounded bg-black text-white" on:click={submit}>Submit feedback</button>
		</footer>
	</div>
</div>

<style>
	/* relies on Tailwind classes — adjust if necessary */
	button[selected] { background: #111; color: #fff; }
</style>