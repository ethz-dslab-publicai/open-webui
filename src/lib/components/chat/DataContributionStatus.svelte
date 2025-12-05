<script lang="ts">
	import dayjs from 'dayjs';

	export let contributionStatus: any;
</script>

<div class="text-xs text-gray-500 bg-gray-50/50 dark:bg-gray-950/50 border border-gray-100 dark:border-gray-850 rounded-3xl px-4 py-2 mx-auto my-4 w-fit max-w-[95%]">
    {#if contributionStatus.status === 'submitted'}
        <p class="text-center mb-1">
            You submitted a data contribution on {dayjs(contributionStatus.timestamp).format('LLLL')}.
        </p>
        <p>
            You can view the status of your contribution via the ID: <a href="{contributionStatus.pr_url}">{contributionStatus.contribution_id}</a>. Thank you for helping to improve our models!
        </p>
        <!-- <p class="mt-1 text-center">
            Thank you for helping to improve our models!
        </p> -->
    {:else if contributionStatus.status === 'cancelled'}
        You cancelled a data contribution on {dayjs(contributionStatus.timestamp).format('LLLL')}.
    {/if}
</div>
