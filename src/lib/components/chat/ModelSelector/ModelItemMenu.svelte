<script lang="ts">
	import { DropdownMenu } from 'bits-ui';
	import { flyAndScale } from '$lib/utils/transitions';

	import { getContext } from 'svelte';

	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Link from '$lib/components/icons/Link.svelte';
	import Eye from '$lib/components/icons/Eye.svelte';
	import EyeSlash from '$lib/components/icons/EyeSlash.svelte';
	import { settings } from '$lib/stores';

	const i18n = getContext('i18n');

	export let show = false;
	export let model;

	export let pinModelHandler: (modelId: string) => void = () => {};
	export let copyLinkHandler: Function = () => {};

	export let onClose: Function = () => {};
</script>

<DropdownMenu.Root
    bind:open={show}
    closeFocus={false}
    onOpenChange={(state) => {
        if (state === false) {
            onClose();
        }
    }}
    typeahead={false}
>
    <DropdownMenu.Trigger>
        <Tooltip content={$i18n.t('More')} className="group-hover/item:opacity-100 opacity-0">
            <slot />
        </Tooltip>
    </DropdownMenu.Trigger>

    <DropdownMenu.Content
        strategy="fixed"
        class="w-full max-w-[200px] text-sm rounded-lg px-1 py-1 z-[9999999] bg-[#ffffff] dark:bg-[#2e2e32] border border-[#e5e5e6] dark:border-[#3d3d3f] shadow-sm"
        sideOffset={5}
        side="bottom"
        align="end"
        transition={flyAndScale}
    >
        <button
            type="button"
            class="flex rounded-md py-2 px-3 w-full hover:bg-[#f7f7f8] dark:hover:bg-[#38383b] transition items-center gap-3 text-[#40414f] dark:text-[#e0e0e0]"
            on:click={(e) => {
                e.stopPropagation();
                e.preventDefault();
                pinModelHandler(model?.id);
                show = false;
            }}
        >
            {#if ($settings?.pinnedModels ?? []).includes(model?.id)}
                <EyeSlash className="size-4 text-[#8e8e9e] dark:text-[#a0a0a5]" />
            {:else}
                <Eye className="size-4 text-[#8e8e9e] dark:text-[#a0a0a5]" />
            {/if}
            <div class="flex items-center text-sm">
                {#if ($settings?.pinnedModels ?? []).includes(model?.id)}
                    {$i18n.t('Hide from Sidebar')}
                {:else}
                    {$i18n.t('Keep in Sidebar')}
                {/if}
            </div>
        </button>

        <button
            type="button"
            class="flex rounded-md py-2 px-3 w-full hover:bg-[#f7f7f8] dark:hover:bg-[#38383b] transition items-center gap-3 text-[#40414f] dark:text-[#e0e0e0]"
            on:click={(e) => {
                e.stopPropagation();
                e.preventDefault();
                copyLinkHandler();
                show = false;
            }}
        >
            <Link className="size-4 text-[#8e8e9e] dark:text-[#a0a0a5]" />
            <div class="flex items-center text-sm">{$i18n.t('Copy Link')}</div>
        </button>
    </DropdownMenu.Content>
</DropdownMenu.Root>