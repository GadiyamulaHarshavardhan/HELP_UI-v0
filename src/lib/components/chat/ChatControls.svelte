<script lang="ts">
    import { SvelteFlowProvider } from '@xyflow/svelte';
    import { Pane, PaneResizer } from 'paneforge';

    import { onDestroy, onMount } from 'svelte';
    import {
        mobile,
        showControls,
        showCallOverlay,
        showOverview,
        showArtifacts,
        isAdmin
    } from '$lib/stores';

    import Modal from '../common/Modal.svelte';
    import Controls from './Controls/Controls.svelte';
    import CallOverlay from './MessageInput/CallOverlay.svelte';
    import Drawer from '../common/Drawer.svelte';
    import Overview from './Overview.svelte';
    import EllipsisVertical from '../icons/EllipsisVertical.svelte';
    import Artifacts from './Artifacts.svelte';

    export let history;
    export let models = [];

    export let chatId = null;

    export let chatFiles = [];
    export let params = {};

    export let eventTarget: EventTarget;
    export let submitPrompt: Function;
    export let stopResponse: Function;
    export let showMessage: Function;
    export let files;
    export let modelId;

    export let pane;

    let mediaQuery;
    let largeScreen = false;
    let dragged = false;
    let minSize = 0;

    const openPane = () => {
        if (!$isAdmin) return;

        const size = parseInt(localStorage?.chatControlsSize) || minSize;
        if (pane && pane.resize) pane.resize(size);
    };

    const handleMediaQuery = async (e) => {
        largeScreen = e.matches;

        if ($showCallOverlay) {
            showCallOverlay.set(false);
            await tick();
            showCallOverlay.set(true);
        }

        if (!largeScreen) pane = null;
    };

    onMount(() => {
        mediaQuery = window.matchMedia('(min-width: 1024px)');
        mediaQuery.addEventListener('change', handleMediaQuery);
        handleMediaQuery(mediaQuery);

        const container = document.getElementById('chat-container');
        minSize = Math.floor((350 / container.clientWidth) * 100);

        const resizeObserver = new ResizeObserver((entries) => {
            for (const entry of entries) {
                const width = entry.contentRect.width;
                const percentage = (350 / width) * 100;
                minSize = Math.floor(percentage);

                if ($showControls && $isAdmin && pane && pane.isExpanded() && pane.getSize() < minSize) {
                    pane.resize(minSize);
                }
            }
        });

        resizeObserver.observe(container);

        return () => {
            mediaQuery.removeEventListener('change', handleMediaQuery);
            resizeObserver.disconnect();
        };
    });

    onDestroy(() => {
        showControls.set(false);
    });

    const closeHandler = () => {
        showControls.set(false);
        showOverview.set(false);
        showArtifacts.set(false);
        if ($showCallOverlay) showCallOverlay.set(false);
    };

    $: if (!chatId) {
        closeHandler();
    }

    // Force reset showControls if user is not admin
    $: {
        if (!$isAdmin && $showControls) {
            showControls.set(false);
        }
    }
</script>

<SvelteFlowProvider>
    {#if !largeScreen}
        <!-- Mobile View -->
        {#if $showControls && $isAdmin}
            <Drawer
                show={$showControls}
                onClose={() => {
                    showControls.set(false);
                }}
            >
                <div class="h-full">
                    {#if $showCallOverlay}
                        <div class="h-full max-h-[100dvh] bg-white text-gray-700 dark:bg-black dark:text-gray-300 flex justify-center">
                            <CallOverlay
                                bind:files
                                {submitPrompt}
                                {stopResponse}
                                {modelId}
                                {chatId}
                                {eventTarget}
                                on:close={() => {
                                    showControls.set(false);
                                }}
                            />
                        </div>
                    {:else if $showArtifacts}
                        <Artifacts {history} />
                    {:else if $showOverview}
                        <Overview
                            {history}
                            on:nodeclick={(e) => {
                                showMessage(e.detail.node.data.message);
                            }}
                            on:close={() => {
                                showControls.set(false);
                            }}
                        />
                    {:else}
                        <Controls
                            on:close={() => {
                                showControls.set(false);
                            }}
                            {models}
                            bind:chatFiles
                            bind:params
                        />
                    {/if}
                </div>
            </Drawer>
        {/if}
    {:else}
        <!-- Desktop View -->
        {#if $isAdmin}
            <PaneResizer
                class="relative flex w-2 items-center justify-center bg-background group cursor-pointer z-50"
                on:click={() => showControls.set(!$showControls)}
            >
                <div class="z-10 flex h-7 w-5 items-center justify-center rounded-xs">
                    <EllipsisVertical className="size-4 invisible group-hover:visible" />
                </div>
            </PaneResizer>
        {/if}

        <Pane
            bind:pane
            defaultSize={0}
            onResize={(size) => {
                if ($showControls && $isAdmin && pane?.isExpanded()) {
                    if (size < minSize) pane.resize(minSize);
                    localStorage.chatControlsSize = size >= minSize ? size : 0;
                }
            }}
            onCollapse={() => {
                if ($isAdmin) showControls.set(false);
            }}
            collapsible={$isAdmin}
            class="z-10"
        >
            {#if $showControls && $isAdmin}
                <div class="flex max-h-full min-h-full">
                    <div class="w-full px-4 py-4 bg-white dark:shadow-lg dark:bg-gray-850 border border-gray-100 dark:border-gray-850 z-40 pointer-events-auto overflow-y-auto scrollbar-hidden">
                        {#if $showCallOverlay}
                            <div class="w-full h-full flex justify-center">
                                <CallOverlay
                                    bind:files
                                    {submitPrompt}
                                    {stopResponse}
                                    {modelId}
                                    {chatId}
                                    {eventTarget}
                                    on:close={() => showControls.set(false)}
                                />
                            </div>
                        {:else if $showArtifacts}
                            <Artifacts {history} overlay={dragged} />
                        {:else if $showOverview}
                            <Overview
                                {history}
                                on:nodeclick={(e) => {
                                    if (e.detail.node.data.message.favorite) {
                                        history.messages[e.detail.node.data.message.id].favorite = true;
                                    } else {
                                        history.messages[e.detail.node.data.message.id].favorite = null;
                                    }
                                    showMessage(e.detail.node.data.message);
                                }}
                                on:close={() => showControls.set(false)}
                            />
                        {:else}
                            <Controls
                                on:close={() => showControls.set(false)}
                                {models}
                                bind:chatFiles
                                bind:params
                            />
                        {/if}
                    </div>
                </div>
            {/if}
        </Pane>
    {/if}
</SvelteFlowProvider>