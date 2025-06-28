<script lang="ts">
	import mermaid from 'mermaid';

	import { v4 as uuidv4 } from 'uuid';

	import { getContext, onMount, tick, onDestroy } from 'svelte';
	import { copyToClipboard } from '$lib/utils';

	import 'highlight.js/styles/github-dark.min.css';

	import PyodideWorker from '$lib/workers/pyodide.worker?worker';
	import CodeEditor from '$lib/components/common/CodeEditor.svelte';
	import SvgPanZoom from '$lib/components/common/SVGPanZoom.svelte';
	import { config } from '$lib/stores';
	import { executeCode } from '$lib/apis/utils';
	import { toast } from 'svelte-sonner';
	import ChevronUp from '$lib/components/icons/ChevronUp.svelte';
	import ChevronUpDown from '$lib/components/icons/ChevronUpDown.svelte';
	import CommandLine from '$lib/components/icons/CommandLine.svelte';
	import Cube from '$lib/components/icons/Cube.svelte';

	const i18n = getContext('i18n');

	export let id = '';

	export let onSave = (e) => {};
	export let onUpdate = (e) => {};
	export let onPreview = (e) => {};

	export let save = false;
	export let run = true;
	export let preview = false;
	export let collapsed = false;

	export let token;
	export let lang = '';
	export let code = '';
	export let attributes = {};

	export let className = 'my-2';

	let pyodideWorker: Worker | null = null;

	let _code = '';
	$: if (code) {
		updateCode();
	}

	const updateCode = () => {
		_code = code;
	};

	let _token = null;

	let mermaidHtml = null;

	let highlightedCode = null;
	let executing = false;

	let stdout = null;
	let stderr = null;
	let result = null;
	let files = null;

	let copied = false;
	let saved = false;

	const collapseCodeBlock = () => {
		collapsed = !collapsed;
	};

	const saveCode = () => {
		saved = true;

		code = _code;
		onSave(code);

		setTimeout(() => {
			saved = false;
		}, 1000);
	};

	const copyCode = async () => {
		copied = true;
		await copyToClipboard(code);

		setTimeout(() => {
			copied = false;
		}, 1000);
	};

	const previewCode = () => {
		onPreview(code);
	};

	const checkPythonCode = (str) => {
		// Check if the string contains typical Python syntax characters
		const pythonSyntax = [
			'def ',
			'else:',
			'elif ',
			'try:',
			'except:',
			'finally:',
			'yield ',
			'lambda ',
			'assert ',
			'nonlocal ',
			'del ',
			'True',
			'False',
			'None',
			' and ',
			' or ',
			' not ',
			' in ',
			' is ',
			' with '
		];

		for (let syntax of pythonSyntax) {
			if (str.includes(syntax)) {
				return true;
			}
		}

		// If none of the above conditions met, it's probably not Python code
		return false;
	};

	const executePython = async (code) => {
		result = null;
		stdout = null;
		stderr = null;

		executing = true;

		if ($config?.code?.engine === 'jupyter') {
			const output = await executeCode(localStorage.token, code).catch((error) => {
				toast.error(`${error}`);
				return null;
			});

			if (output) {
				if (output['stdout']) {
					stdout = output['stdout'];
					const stdoutLines = stdout.split('\n');

					for (const [idx, line] of stdoutLines.entries()) {
						if (line.startsWith('data:image/png;base64')) {
							if (files) {
								files.push({
									type: 'image/png',
									data: line
								});
							} else {
								files = [
									{
										type: 'image/png',
										data: line
									}
								];
							}

							if (stdout.startsWith(`${line}\n`)) {
								stdout = stdout.replace(`${line}\n`, ``);
							} else if (stdout.startsWith(`${line}`)) {
								stdout = stdout.replace(`${line}`, ``);
							}
						}
					}
				}

				if (output['result']) {
					result = output['result'];
					const resultLines = result.split('\n');

					for (const [idx, line] of resultLines.entries()) {
						if (line.startsWith('data:image/png;base64')) {
							if (files) {
								files.push({
									type: 'image/png',
									data: line
								});
							} else {
								files = [
									{
										type: 'image/png',
										data: line
									}
								];
							}

							if (result.startsWith(`${line}\n`)) {
								result = result.replace(`${line}\n`, ``);
							} else if (result.startsWith(`${line}`)) {
								result = result.replace(`${line}`, ``);
							}
						}
					}
				}

				output['stderr'] && (stderr = output['stderr']);
			}

			executing = false;
		} else {
			executePythonAsWorker(code);
		}
	};

	const executePythonAsWorker = async (code) => {
    let packages = [
        code.includes('requests') ? 'requests' : null,
        code.includes('bs4') ? 'beautifulsoup4' : null,
        code.includes('numpy') ? 'numpy' : null,
        code.includes('pandas') ? 'pandas' : null,
        code.includes('sklearn') ? 'scikit-learn' : null,
        code.includes('scipy') ? 'scipy' : null,
        code.includes('re') ? 'regex' : null,
        code.includes('seaborn') ? 'seaborn' : null,
        code.includes('sympy') ? 'sympy' : null,
        code.includes('tiktoken') ? 'tiktoken' : null,
        code.includes('matplotlib') ? 'matplotlib' : null,
        code.includes('pytz') ? 'pytz' : null
    ].filter(Boolean);

    console.log('Packages to install:', packages);

    pyodideWorker = new PyodideWorker();

    executing = true;
    stdout = null;
    stderr = null;
    result = null;
    files = null;

    const workerReadyPromise = new Promise((resolve, reject) => {
        const timeout = setTimeout(() => {
            reject(new Error('Worker initialization timed out.'));
        }, 10000);

        pyodideWorker.onmessage = (event) => {
            if (event.data.type === 'worker-ready') {
                clearTimeout(timeout);
                resolve(event.data.pyodideVersion);
            }
        };

        pyodideWorker.onerror = (error) => {
            clearTimeout(timeout);
            reject(error);
        };
    });

    try {
        await workerReadyPromise;

        // Send message to install packages and run code
        pyodideWorker.postMessage({
            id: id,
            type: 'run-python',
            code: code,
            packages: packages
        });

        // Set timeout for execution
        const executionTimeout = setTimeout(() => {
            if (executing) {
                executing = false;
                stderr = 'Execution Time Limit Exceeded';
                pyodideWorker.terminate();
                toast.error('Execution timed out.');
            }
        }, 60000);

        pyodideWorker.onmessage = (event) => {
            clearTimeout(executionTimeout);

            const data = event.data;

            if (data.stdout) stdout = data.stdout;
            if (data.stderr) stderr = data.stderr;
            if (data.result) result = data.result;

            if (data.images && Array.isArray(data.images)) {
                files = data.images.map((img) => ({
                    type: img.type || 'image/png',
                    data: img.data
                }));
            }

            executing = false;

            if (data.error) {
                toast.error(`Error in execution:\n${data.error}`);
            } else if (!stderr) {
                toast.success('Code executed successfully!');
            }
        };

        pyodideWorker.onerror = (event) => {
            executing = false;
            stderr = `Worker error: ${event.message}`;
            toast.error(stderr);
            console.error('Worker error:', event);
        };

        pyodideWorker.onmessageerror = (event) => {
            executing = false;
            toast.error('Message error from worker');
            console.error('Message error:', event);
        };
    } catch (e) {
        executing = false;
        stderr = `Failed to initialize Pyodide: ${e.message}`;
        toast.error(stderr);
        console.error(e);
    }
};

	const drawMermaidDiagram = async () => {
		try {
			if (await mermaid.parse(code)) {
				const { svg } = await mermaid.render(`mermaid-${uuidv4()}`, code);
				mermaidHtml = svg;
			}
		} catch (error) {
			console.log('Error:', error);
		}
	};

	const render = async () => {
		if (lang === 'mermaid' && (token?.raw ?? '').slice(-4).includes('```')) {
			(async () => {
				await drawMermaidDiagram();
			})();
		}

		onUpdate(token);
	};

	$: if (token) {
		if (JSON.stringify(token) !== JSON.stringify(_token)) {
			_token = token;
		}
	}

	$: if (_token) {
		render();
	}

	$: if (attributes) {
		onAttributesUpdate();
	}

	const onAttributesUpdate = () => {
		if (attributes?.output) {
			// Create a helper function to unescape HTML entities
			const unescapeHtml = (html) => {
				const textArea = document.createElement('textarea');
				textArea.innerHTML = html;
				return textArea.value;
			};

			try {
				// Unescape the HTML-encoded string
				const unescapedOutput = unescapeHtml(attributes.output);

				// Parse the unescaped string into JSON
				const output = JSON.parse(unescapedOutput);

				// Assign the parsed values to variables
				stdout = output.stdout;
				stderr = output.stderr;
				result = output.result;
			} catch (error) {
				console.error('Error:', error);
			}
		}
	};

	onMount(async () => {
		console.log('codeblock', lang, code);
		if (token) {
			onUpdate(token);
		}

		if (document.documentElement.classList.contains('dark')) {
			mermaid.initialize({
				startOnLoad: true,
				theme: 'dark',
				securityLevel: 'loose'
			});
		} else {
			mermaid.initialize({
				startOnLoad: true,
				theme: 'default',
				securityLevel: 'loose'
			});
		}
	});

	onDestroy(() => {
		if (pyodideWorker) {
			pyodideWorker.terminate();
		}
	});
</script>
<div>
    <div class="relative {className} flex flex-col rounded-md bg-[#f7f7f8] dark:bg-[#232427] border border-[#e5e5e6] dark:border-[#3d3d3f] mx-auto max-w-4xl overflow-hidden" dir="ltr">
        {#if lang === 'mermaid'}
            {#if mermaidHtml}
                <SvgPanZoom
                    class="border-b border-[#e5e5e6] dark:border-[#3d3d3f] rounded-t-md overflow-hidden"
                    svg={mermaidHtml}
                    content={_token.text}
                />
            {:else}
                <pre class="mermaid p-4 bg-[#f7f7f8] dark:bg-[#232427]">{code}</pre>
            {/if}
        {:else}
            <!-- Header with language tag and controls -->
            <div class="flex items-center justify-between px-3 py-2 bg-[#f0f0f1] dark:bg-[#2e2e32] border-b border-[#e5e5e6] dark:border-[#3d3d3f]">
                <div class="flex items-center gap-2">
                    <div class="inline-flex items-center gap-1.5 px-2 py-1 bg-white dark:bg-[#38383b] rounded text-xs">
                        <div class="w-2 h-2 rounded-full bg-[#4e9efd]"></div>
                        <span class="text-[#40414f] dark:text-[#e0e0e0] font-mono">
                            {lang || 'Code'}
                        </span>
                    </div>
                </div>
                
                <div class="flex items-center gap-1">
                    <button
                        class="flex gap-1 items-center bg-transparent hover:bg-[#e5e5e6] dark:hover:bg-[#38383b] rounded px-2 py-1 text-xs text-[#40414f] dark:text-[#e0e0e0]"
                        on:click={collapseCodeBlock}
                    >
                        <ChevronUpDown className="size-3.5" />
                        <span>{collapsed ? $i18n.t('Expand') : $i18n.t('Collapse')}</span>
                    </button>

                    {#if preview && ['html', 'svg'].includes(lang)}
                        <button
                            class="flex gap-1 items-center run-code-button bg-transparent hover:bg-[#e5e5e6] dark:hover:bg-[#38383b] rounded px-2 py-1 text-xs text-[#4e9efd] dark:text-[#4e9efd]"
                            on:click={previewCode}
                        >
                            <Cube className="size-3.5" />
                            <span>{$i18n.t('Preview')}</span>
                        </button>
                    {/if}

                    {#if ($config?.features?.enable_code_execution ?? true) && (lang.toLowerCase() === 'python' || lang.toLowerCase() === 'py' || (lang === '' && checkPythonCode(code)))}
                        {#if executing}
                            <div class="flex gap-1 items-center run-code-button bg-[#fff8e6] dark:bg-[#3a2e0f] rounded px-2 py-1 text-xs text-[#b88a00] dark:text-[#ffc533] cursor-not-allowed">
                                <div class="w-3 h-3 border-2 border-[#b88a00] dark:border-[#ffc533] border-t-transparent rounded-full animate-spin"></div>
                                <span>{$i18n.t('Running...')}</span>
                            </div>
                        {:else if run}
                            <button
                                class="flex gap-1 items-center run-code-button bg-transparent hover:bg-[#e5e5e6] dark:hover:bg-[#38383b] rounded px-2 py-1 text-xs text-[#2e7d32] dark:text-[#4caf50]"
                                on:click={async () => {
                                    code = _code;
                                    await tick();
                                    executePython(code);
                                }}
                            >
                                <CommandLine className="size-3.5" />
                                <span>{$i18n.t('Run')}</span>
                            </button>
                        {/if}
                    {/if}

                    {#if save}
                        <button
                            class="save-code-button bg-transparent hover:bg-[#e5e5e6] dark:hover:bg-[#38383b] rounded px-2 py-1 text-xs text-[#40414f] dark:text-[#e0e0e0]"
                            on:click={saveCode}
                        >
                            {saved ? $i18n.t('Saved') : $i18n.t('Save')}
                        </button>
                    {/if}

                    <button
                        class="copy-code-button bg-transparent hover:bg-[#e5e5e6] dark:hover:bg-[#38383b] rounded px-2 py-1 text-xs text-[#40414f] dark:text-[#e0e0e0]"
                        on:click={copyCode}
                    >
                        {copied ? $i18n.t('Copied') : $i18n.t('Copy')}
                    </button>
                </div>
            </div>

            <!-- Code editor container -->
            <div class="relative">
                {#if !collapsed}
                    <div class="bg-white dark:bg-[#1e1e20]">
                        <CodeEditor
                            value={code}
                            {id}
                            {lang}
                            onSave={() => {
                                saveCode();
                            }}
                            onChange={(value) => {
                                _code = value;
                            }}
                        />
                    </div>
                {:else}
                    <div class="bg-[#f0f0f1] dark:bg-[#2e2e32] py-2 px-3 flex items-center gap-2 text-xs text-[#8e8e9e] dark:text-[#a0a0a5]">
                        <span class="italic">
                            {$i18n.t('{{COUNT}} hidden lines', {
                                COUNT: code.split('\n').length
                            })}
                        </span>
                    </div>
                {/if}
            </div>

            {#if !collapsed}
                <!-- Plot canvas -->
                <div id="plt-canvas-{id}" class="bg-white dark:bg-[#1e1e20] max-w-full overflow-x-auto scrollbar-hidden"></div>

                <!-- Output section -->
                {#if executing || stdout || stderr || result || files}
                    <div class="bg-[#f0f0f1] dark:bg-[#2e2e32] border-t border-[#e5e5e6] dark:border-[#3d3d3f]">
                        {#if executing}
                            <div class="p-3">
                                <div class="flex items-center gap-2 text-[#b88a00] dark:text-[#ffc533] text-sm">
                                    <div class="w-4 h-4 border-2 border-[#b88a00] dark:border-[#ffc533] border-t-transparent rounded-full animate-spin"></div>
                                    <span>Executing code...</span>
                                </div>
                            </div>
                        {:else}
                            <div class="p-3 space-y-3">
                                {#if stdout || stderr}
                                    <div class="space-y-1">
                                        <div class="flex items-center gap-2 text-xs text-[#8e8e9e] dark:text-[#a0a0a5]">
                                            <div class="w-1.5 h-1.5 rounded-full bg-[#4caf50]"></div>
                                            <span>Output</span>
                                        </div>
                                        <pre class="text-sm bg-white dark:bg-[#1e1e20] p-2 rounded border border-[#e5e5e6] dark:border-[#3d3d3f] overflow-x-auto text-[#40414f] dark:text-[#e0e0e0]">
{stdout || stderr}
                                        </pre>
                                    </div>
                                {/if}
                                
                                {#if result || files}
                                    <div class="space-y-1">
                                        <div class="flex items-center gap-2 text-xs text-[#8e8e9e] dark:text-[#a0a0a5]">
                                            <div class="w-1.5 h-1.5 rounded-full bg-[#4e9efd]"></div>
                                            <span>Result</span>
                                        </div>
                                        
                                        {#if result}
                                            <pre class="text-sm bg-white dark:bg-[#1e1e20] p-2 rounded border border-[#e5e5e6] dark:border-[#3d3d3f] overflow-x-auto text-[#40414f] dark:text-[#e0e0e0]">
{JSON.stringify(result, null, 2)}
                                            </pre>
                                        {/if}
                                        
                                        {#if files}
                                            <div class="grid gap-2 mt-2">
                                                {#each files as file}
                                                    {#if file.type.startsWith('image')}
                                                        <div class="bg-white dark:bg-[#1e1e20] p-1 rounded border border-[#e5e5e6] dark:border-[#3d3d3f]">
                                                            <img 
                                                                src={file.data} 
                                                                alt="Generated output" 
                                                                class="w-full max-w-2xl rounded" 
                                                            />
                                                        </div>
                                                    {/if}
                                                {/each}
                                            </div>
                                        {/if}
                                    </div>
                                {/if}
                            </div>
                        {/if}
                    </div>
                {/if}
            {/if}
        {/if}
    </div>
</div>