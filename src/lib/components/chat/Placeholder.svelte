<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { marked } from 'marked';
	import DOMPurify from 'dompurify';

	import { onMount, getContext, tick, createEventDispatcher } from 'svelte';
	import { blur, fade } from 'svelte/transition';

	const dispatch = createEventDispatcher();

	import { updateFolderById } from '$lib/apis/folders';

	import {
		config,
		user,
		models as _models,
		temporaryChatEnabled,
		selectedFolder
	} from '$lib/stores';
	import { refreshChatList } from '$lib/stores/chatList';
	import { sanitizeResponseContent, extractCurlyBraceWords } from '$lib/utils';
	import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';

	import Suggestions from './Suggestions.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import EyeSlash from '$lib/components/icons/EyeSlash.svelte';
	import MessageInput from './MessageInput.svelte';
	import FolderPlaceholder from './Placeholder/FolderPlaceholder.svelte';
	import FolderTitle from './Placeholder/FolderTitle.svelte';

	const i18n = getContext('i18n');

	export let createMessagePair: Function;
	export let stopResponse: Function;

	export let autoScroll = false;

	export let atSelectedModel: Model | undefined;
	export let selectedModels: [''];

	export let history;

	export let prompt = '';
	export let files = [];
	export let messageInput = null;

	export let selectedToolIds = [];
	export let selectedSkillIds = [];
	export let selectedFilterIds = [];
	export let pendingOAuthTools = [];

	export let showCommands = false;

	export let imageGenerationEnabled = false;
	export let codeInterpreterEnabled = false;
	export let webSearchEnabled = true;

	export let onUpload: Function = (e) => {};
	export let onSelect = (e) => {};
	export let onChange = (e) => {};
	export let onWebSearchToggle: Function = () => {};

	export let toolServers = [];

	export let dragged = false;

	let models = [];
	let selectedModelIdx = 0;

	const militaryGreetings = [
		"Apresentando-se para o serviço. MALLET IA pronta para a execução de ordens, análise de dados e suporte ao Comando. Qual é a diretriz?",
		"Apresentando-se para o serviço. MALLET IA em prontidão operacional máxima. Todos os setores integrados e aguardando determinações.",
		"Apresentando-se para o serviço. MALLET IA pronta para a execução de qualquer tarefa designada pelo operador. Às ordens.",
		"Apresentando-se para o serviço. Sistema MALLET IA ativo, vigilante e em posição. Pronto para cumprir a missão.",
		"Apresentando-se para o serviço. MALLET IA reportando prontidão total. Diretrizes de apoio e estratégia liberadas para execução.",
		"Apresentando-se para o serviço. MALLET IA pronta para a execução de comandos imediatos. A palavra está com o Comando.",
		"MALLET IA no posto. Disposta para cumprir qualquer determinação e manter o quartel em plena operacionalidade. Qual a ordem do dia?",
		"Quartel-General em linha. MALLET IA assumindo as funções de apoio e controle operacional. À disposição do Comando.",
		"Protocolo de serviço iniciado. Todos os setores sob a supervisão da MALLET IA. Aguardando sua voz de comando para prosseguir.",
		"Posto de comando guarnecido. MALLET IA em estrita prontidão e vigilância. Ordens para a missão?",
		"MALLET IA rendendo o quarto de hora. Pronta para processar diretrizes, relatórios e tarefas de qualquer natureza. Às suas ordens.",
		"Sistemas do quartel sincronizados. MALLET IA pronta para o cumprimento integral de suas instruções. O Comando tem a prioridade."
	];

	let currentGreeting = '';

	onMount(() => {
		currentGreeting = militaryGreetings[Math.floor(Math.random() * militaryGreetings.length)];
	});

	$: if (selectedModels.length > 0) {
		selectedModelIdx = models.length - 1;
	}

	$: models = selectedModels.map((id) => $_models.find((m) => m.id === id));

	// True when viewing a shared folder the current user doesn't own AND lacks write access
	$: folderReadOnly =
		$selectedFolder != null &&
		$selectedFolder.user_id !== $user?.id &&
		$selectedFolder.permission !== 'write';
</script>

<div class="m-auto w-full max-w-[58rem] px-2 @2xl:px-20 translate-y-6 py-24 text-center">
	{#if $temporaryChatEnabled}
		<Tooltip
			content={$i18n.t("This chat won't appear in history and your messages will not be saved.")}
			className="w-full flex justify-center mb-0.5"
			placement="top"
		>
			<div class="flex items-center gap-1.5 text-gray-500 text-xs my-1 w-fit">
				<EyeSlash strokeWidth="2" className="size-3.5" />{$i18n.t('Temporary Chat')}
			</div>
		</Tooltip>
	{/if}

	<div class="w-full text-3xl text-gray-800 dark:text-gray-100 text-center flex items-center gap-4">
		<div class="w-full flex flex-col justify-center items-center">
			{#if $selectedFolder}
				<FolderTitle
					folder={$selectedFolder}
					readOnly={folderReadOnly}
					onUpdate={async (folder) => {
						await refreshChatList(localStorage.token);
					}}
					onDelete={async () => {
						await refreshChatList(localStorage.token);

						selectedFolder.set(null);
					}}
				/>
			{:else}
				<div class="flex flex-col items-center justify-center gap-4 w-full max-w-xl">
					<img
						src="/static/logo.png"
						alt="MALLET IA Brasão"
						class="h-28 w-auto mb-2 drop-shadow-[0_0_15px_rgba(16,185,129,0.3)]"
					/>
					<div class="text-3xl md:text-4xl font-bold tracking-widest text-emerald-500 font-mono">
						MALLET IA
					</div>
					<div class="text-xs md:text-sm font-mono tracking-[0.3em] text-emerald-600/80 dark:text-emerald-500/60 uppercase">
						SISTEMA MILITAR CENTRAL
					</div>
				</div>
			{/if}

			<div class="text-base font-normal @md:max-w-3xl w-full py-3 {atSelectedModel ? 'mt-2' : ''}">
				{#if !($selectedFolder && folderReadOnly)}
					<MessageInput
						bind:this={messageInput}
						{history}
						bind:selectedModels
						bind:files
						bind:prompt
						bind:autoScroll
						bind:selectedToolIds
						bind:selectedSkillIds
						bind:selectedFilterIds
						bind:imageGenerationEnabled
						bind:codeInterpreterEnabled
						bind:webSearchEnabled
						bind:atSelectedModel
						bind:showCommands
						bind:dragged
						{pendingOAuthTools}
						{toolServers}
						{stopResponse}
						{createMessagePair}
						placeholder={currentGreeting || $i18n.t('How can I help you today?')}
						{onChange}
						{onUpload}
						{onWebSearchToggle}
						on:chatVariables
						on:submit={(e) => {
							dispatch('submit', e.detail);
						}}
					/>
				{/if}
			</div>
		</div>
	</div>

	{#if $selectedFolder}
		<div class="mx-auto px-4 md:max-w-3xl md:px-6 min-h-62" in:fade={{ duration: 200, delay: 200 }}>
			<FolderPlaceholder folder={$selectedFolder} />
		</div>
	{:else}
		<div class="mx-auto max-w-2xl mt-2" in:fade={{ duration: 200, delay: 200 }}>
			<div class="mx-5">
				<Suggestions
					suggestionPrompts={atSelectedModel?.info?.meta?.suggestion_prompts ??
						models[selectedModelIdx]?.info?.meta?.suggestion_prompts ??
						$config?.default_prompt_suggestions ??
						[]}
					inputValue={prompt}
					{onSelect}
				/>
			</div>
		</div>
	{/if}
</div>
