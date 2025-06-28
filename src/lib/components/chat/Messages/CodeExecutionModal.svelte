<script lang="ts">
	import { getContext } from 'svelte';
	import CodeBlock from './CodeBlock.svelte';
	import Modal from '$lib/components/common/Modal.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import Badge from '$lib/components/common/Badge.svelte';
	const i18n = getContext('i18n');

	export let show = false;
	export let codeExecution = null;
</script>

<Modal size="lg" bind:show>
	<div class="bg-[#f7f7f8] dark:bg-[#232427] rounded-lg">
		<div class="flex justify-between items-center px-5 py-3 border-b border-[#e5e5e6] dark:border-[#3d3d3f]">
			<div class="flex items-center gap-3">
				{#if codeExecution?.result}
					<div>
						{#if codeExecution.result?.error}
							<Badge type="error" content="Error" class="bg-[#fde8e8] dark:bg-[#3a1a1a] text-[#d93025] dark:text-[#f28b82]" />
						{:else if codeExecution.result?.output}
							<Badge type="success" content="Success" class="bg-[#e6f4ea] dark:bg-[#1e3a21] text-[#188038] dark:text-[#81c995]" />
						{:else}
							<Badge type="warning" content="Incomplete" class="bg-[#fef7e0] dark:bg-[#3a2e0f] text-[#b88a00] dark:text-[#ffc533]" />
						{/if}
					</div>
				{/if}

				<div class="flex gap-2 items-center text-[#40414f] dark:text-[#e0e0e0]">
					{#if !codeExecution?.result}
						<div>
							<Spinner className="size-4 text-[#4e9efd]" />
						</div>
					{/if}

					<h3 class="font-medium">
						{#if codeExecution?.name}
							Code Execution: {codeExecution?.name}
						{:else}
							Code Execution
						{/if}
					</h3>
				</div>
			</div>
			<button
				class="text-[#8e8e9e] dark:text-[#a0a0a5] hover:text-[#40414f] dark:hover:text-[#e0e0e0]"
				on:click={() => {
					show = false;
					codeExecution = null;
				}}
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					viewBox="0 0 20 20"
					fill="currentColor"
					class="w-5 h-5"
				>
					<path
						d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z"
					/>
				</svg>
			</button>
		</div>

		<div class="flex flex-col w-full p-4">
			<div class="flex flex-col w-full dark:text-gray-200 overflow-y-auto max-h-[22rem] scrollbar-hidden">
				<div class="flex flex-col w-full mb-4">
					<CodeBlock
						id="code-exec-{codeExecution?.id}-code"
						lang={codeExecution?.language ?? ''}
						code={codeExecution?.code ?? ''}
						className="border border-[#e5e5e6] dark:border-[#3d3d3f]"
						editorClassName={codeExecution?.result &&
						(codeExecution?.result?.error || codeExecution?.result?.output)
							? 'rounded-b-none'
							: ''}
						stickyButtonsClassName="top-0"
						run={false}
					/>
				</div>

				{#if codeExecution?.result && (codeExecution?.result?.error || codeExecution?.result?.output)}
					<div class="bg-white dark:bg-[#1e1e20] border border-[#e5e5e6] dark:border-[#3d3d3f] rounded-b-lg p-4 flex flex-col gap-4">
						{#if codeExecution?.result?.error}
							<div>
								<div class="text-xs text-[#8e8e9e] dark:text-[#a0a0a5] mb-2 font-mono">ERROR</div>
								<pre class="text-sm text-[#d93025] dark:text-[#f28b82] bg-[#f7f7f8] dark:bg-[#2e2e32] p-2 rounded overflow-x-auto">{codeExecution?.result?.error}</pre>
							</div>
						{/if}
						{#if codeExecution?.result?.output}
							<div>
								<div class="text-xs text-[#8e8e9e] dark:text-[#a0a0a5] mb-2 font-mono">OUTPUT</div>
								<pre class="text-sm text-[#40414f] dark:text-[#e0e0e0] bg-[#f7f7f8] dark:bg-[#2e2e32] p-2 rounded overflow-x-auto">{codeExecution?.result?.output}</pre>
							</div>
						{/if}
					</div>
				{/if}
				
				{#if codeExecution?.result?.files && codeExecution?.result?.files.length > 0}
					<div class="mt-4 pt-4 border-t border-[#e5e5e6] dark:border-[#3d3d3f]">
						<div class="text-sm font-medium text-[#40414f] dark:text-[#e0e0e0] mb-2">
							Files
						</div>
						<ul class="grid gap-2">
							{#each codeExecution?.result?.files as file}
								<li>
									<a 
										href={file.url} 
										target="_blank"
										class="flex items-center gap-2 text-sm text-[#4e9efd] hover:underline"
									>
										<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-4 h-4">
											<path fill-rule="evenodd" d="M4.5 2A1.5 1.5 0 003 3.5v13A1.5 1.5 0 004.5 18h11a1.5 1.5 0 001.5-1.5V7.621a1.5 1.5 0 00-.44-1.06l-4.12-4.122A1.5 1.5 0 0011.378 2H4.5z" clip-rule="evenodd" />
										</svg>
										{file.name}
									</a>
								</li>
							{/each}
						</ul>
					</div>
				{/if}
			</div>
		</div>
	</div>
</Modal>