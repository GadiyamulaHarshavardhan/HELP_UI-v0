<script lang="ts">
	import Info from '$lib/components/icons/Info.svelte';

	export let content = '';
</script>

<div class="flex my-2 gap-3 border px-4 py-3 border-[#f8d7da] dark:border-[#3a1a1a] bg-[#f8d7da]/30 dark:bg-[#3a1a1a]/30 rounded-lg">
    <div class="self-start mt-0.5">
        <Info className="size-5 text-[#d93025] dark:text-[#f28b82]" />
    </div>

    <div class="self-center text-sm text-[#721c24] dark:text-[#f28b82]">
        {typeof content === 'string' ? content : JSON.stringify(content)}
    </div>
</div>
