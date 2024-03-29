<script lang="ts">
	import ResourceForm from '$lib/components/ResourceForm.svelte';
	import {
		AppShell,
		Modal,
		modalStore,
		RadioGroup,
		RadioItem,
		Toast,
		toastStore,
		type ModalComponent,
		type ModalSettings,
		type ToastSettings
	} from '@skeletonlabs/skeleton';
	import { readText } from '@tauri-apps/api/clipboard';
	import github from 'svelte-highlight/styles/github';
	import { v4 as uuidv4 } from 'uuid';

	import { extensionMap, showNotification, store, type Item, type ItemType } from '$lib/app';
	import Header from '$lib/components/Header.svelte';
	import ItemNote from '$lib/components/ItemNote.svelte';
	import ItemResource from '$lib/components/ItemResource.svelte';
	import ItemSnippet from '$lib/components/ItemSnippet.svelte';
	import ItemUrl from '$lib/components/ItemUrl.svelte';
	import TitleBar from '$lib/components/TitleBar.svelte';
	import { menu } from '@skeletonlabs/skeleton';
	import { emit, listen, TauriEvent, type UnlistenFn } from '@tauri-apps/api/event';
	import { basename } from '@tauri-apps/api/path';
	import { appWindow, WebviewWindow } from '@tauri-apps/api/window';
	import ClipboardJS from 'clipboard';
	import debounce from 'lodash.debounce';
	import { onDestroy, onMount, SvelteComponent } from 'svelte';
	import { writable, type Writable } from 'svelte/store';

	let search = '';
	let fileHover = false;
	let items: Item[] = [];
	let searchItems: Item[] = [];

	const clipboard = new ClipboardJS('.copy');

	const typeFilter: Writable<ItemType | any> = writable('All');

	const eventListeners: Promise<UnlistenFn>[] = [];

	const filterItems = debounce(() => {
		const types = [$typeFilter];
		if ($typeFilter == 'File') {
			types.push('Folder');
		}
		searchItems = items
			.sort((a, b) => {
				if (a.name > b.name) return 1;
				if (a.name < b.name) return -1;
				return 0;
			})
			.filter((e) => {
				const re = new RegExp(search, 'gi');
				return re.test(e.name) && (types.includes(e.type) || $typeFilter == 'All');
			});
	}, 100);

	typeFilter.subscribe(async (e) => {
		filterItems();
	});

	eventListeners.push(listen('store-update', reloadData));

	eventListeners.push(
		store.onChange(async (_) => {
			reloadData();
		})
	);

	async function reloadData() {
		items = await store.values();
		filterItems();
	}

	clipboard.on('success', function (e) {
		const toast: ToastSettings = {
			message: 'Copied!',
			preset: 'primary',
			autohide: true,
			timeout: 1000
		};
		toastStore.trigger(toast);

		e.clearSelection();
	});

	onMount(async () => {
		items = await store.values();
		filterItems();
	});

	onDestroy(() => {
		eventListeners.forEach(async (e) => (await e).call(e));
	});

	eventListeners.push(
		appWindow.listen(TauriEvent.WINDOW_FILE_DROP_HOVER, ({ payload }) => {
			if ((payload as any[]).length) {
				fileHover = true;
			}
		})
	);

	eventListeners.push(
		appWindow.listen(TauriEvent.WINDOW_FILE_DROP_CANCELLED, (_) => {
			fileHover = false;
		})
	);

	eventListeners.push(
		appWindow.listen(TauriEvent.WINDOW_FOCUS, async (_) => {
			const text = await readText();
			if (text && text.startsWith('http')) {
				// if url already added, skip alert
				const match = items.find((e) => e.path == text);
				if (match) return;

				const toast: ToastSettings = {
					message: 'URL Detected from clipoard',
					preset: 'primary',
					autohide: true,
					timeout: 5000,
					action: {
						label: 'Add',
						response: async () => {
							const name = text ? await basename(text) : '';
							createResource('WebURL', {
								type: 'WebURL',
								path: text,
								name
							});
						}
					}
				};
				toastStore.trigger(toast);
			}
		})
	);

	eventListeners.push(
		appWindow.listen(TauriEvent.WINDOW_FILE_DROP, async (event) => {
			fileHover = false;
			const extensions = Object.keys(extensionMap);
			const payload = event.payload as string[];
			const added: any[] = [];

			if (payload.length == 1) {
				const path = payload[0];
				const name = await basename(path);
				const ext = name.split('.').reverse()[0];
				const type = extensions.includes(ext) || name.startsWith('.') ? 'File' : 'Folder';
				editResource({
					path,
					name,
					ext,
					type
				});
			} else {
				for (let i = 0; i < payload.length; i++) {
					const path = payload[i];
					const name = await basename(path);

					if (await store.has(name)) continue;

					const ext = path.split('.').reverse()[0];
					let item: { [key: string]: any } = {
						id: uuidv4(),
						name,
						path,
						ext
					};
					if (extensions.includes(ext) || name.startsWith('.')) {
						item.type = 'File';
					} else {
						item.type = 'Folder';
					}
					added.push(item);
					store.set(`${item.type}:${name}`, item);
				}
			}

			if (added.length) {
				await store.save();
				items = await store.values();

				showNotification(
					`(${added.length}) New Resource Added`,
					added.reduce((a, b) => {
						a += (b.type == 'File' ? '📄' : '📁') + `${b.name} \n`;
						return a;
					}, '\n')
				);
				emit('store-update');
			}
		})
	);

	async function editResource(item?: Item | any) {
		const origName = item?.name;

		showForm(
			'Edit Resource',
			ResourceForm,
			async (res) => {
				if (!res) return;
				res.id = uuidv4();
				if (res.type == 'File') {
					const base = await basename(res.path);
					res.ext = base.split('.').reverse()[0];
				}
				await store.delete(`${res.type}:${origName}`);
				await store.set(`${res.type}:${res.name}`, res);
				await store.save();
				emit('store-update');
			},
			{
				formData: item
			}
		);
	}

	async function createResource(type: ItemType, item?: Item) {
		let title = '';
		switch (type) {
			case 'File':
			case 'Folder':
				title = 'Create File/Folder Resource';
				break;
			case 'WebURL':
				title = 'Create WebURL Resource';
				break;

			case 'Snippet':
				title = 'Create Snippet Resource';
				break;
			case 'Note':
				const untitledCount = items.filter(
					(e) => e.type == 'Note' && e.name == 'Untitled'
				).length;
				const win = new WebviewWindow('create-note', {
					width: 500,
					height: 490,
					decorations: false,
					url: `/note?name=Untitled ${untitledCount + 1}`
				});

				await win.show();
				await appWindow.hide();
				return;
		}
		showForm(
			title,
			ResourceForm,
			async (res: Item) => {
				if (!res) return;
				res.id = uuidv4();
				if (res.type == 'File') {
					const base = await basename(res.path);
					res.ext = base.split('.').reverse()[0];
				}
				await store.delete(`${res.type}:${res.name}`);
				await store.set(`${res.type}:${res.name}`, res);
				await store.save();
				emit('store-update');

				showNotification('New Resource Added', res.name);
			},
			{
				fileFolder: type == 'File',
				formData: item ?? {
					type
				}
			}
		);
	}

	function showForm(
		title: string,
		form: typeof SvelteComponent,
		callback: (res: any) => void,
		props: any
	) {
		const modalComponent: ModalComponent = {
			ref: form,
			props
		};
		const modalSettings: ModalSettings = {
			type: 'component',
			title,
			component: modalComponent,
			response: callback
		};
		modalStore.trigger(modalSettings);
	}

	async function openNote(item: Item) {
		const label = `note-${item.id}`;
		const win = WebviewWindow.getByLabel(label);
		if (win) {
			win.setFocus();
			return;
		}
		new WebviewWindow(label, {
			width: 500,
			height: 490,
			decorations: false,
			url: `/note?name=${item.name}`,
			visible: true,
			alwaysOnTop: true
		});
	}
</script>

{#if fileHover}
	<div
		class="absolute z-[999] w-full h-full bg-surface-500 bg-opacity-50 flex items-center justify-center">
		<span class="text-3xl font-bold">Drop file/folder to add as resource</span>
	</div>
{/if}

<svelte:head>
	{@html github}
	<style>
		pre {
			border-radius: 15px !important;
		}
		.radio-group {
			border-color: transparent !important;
		}
	</style>
</svelte:head>

<AppShell>
	<svelte:fragment slot="header">
		<TitleBar />
		<Header title="Quick Pick">
			<svelte:fragment slot="action">
				<a href="/manage/settings" class="mr-2">
					<i class="mdi mdi-cog text-2xl" />
				</a>
			</svelte:fragment>
		</Header>
		<div class="p-1">
			<div class="w-full card flex items-center">
				<div class="flex flex-nowrap bg-surface-200-700-token rounded-full w-full">
					<div class="relative flex-grow flex whitespace-nowrap">
						<i class="mdi mdi-magnify absolute left-3 top-[50%] translate-y-[-50%]" />
						<input
							type="text"
							class="px-7"
							autocomplete="off"
							autocapitalize="off"
							autocorrect="off"
							bind:value={search}
							on:input={filterItems} />
						{#if search}
							<button
								class="absolute right-3 top-[50%] translate-y-[-50%]"
								on:click={() => {
									search = '';
									filterItems();
								}}>
								<i class="mdi mdi-backspace" />
							</button>
						{/if}
					</div>
					<div class="whitespace-nowrap">
						<RadioGroup selected={typeFilter} padding="px-2 py-1">
							<RadioItem value="All">All</RadioItem>
							<RadioItem value="File">
								<i class="mdi mdi-folder-file" />
							</RadioItem>
							<RadioItem value="WebURL">
								<i class="mdi mdi-web" />
							</RadioItem>
							<RadioItem value="Snippet">
								<i class="mdi mdi-code-braces" />
							</RadioItem>
							<RadioItem value="Note">
								<i class="mdi mdi-note-edit" />
							</RadioItem>
						</RadioGroup>
					</div>
				</div>
				<span class="divider-vertical h-4 ml-3" />
				<span class="relative">
					{#if $typeFilter != 'All'}
						<button
							class="whitespace-nowrap px-2 mr-3"
							on:click={() => createResource($typeFilter)}>Add Item</button>
					{/if}
					<button
						class="whitespace-nowrap px-2 mr-3"
						class:hidden={$typeFilter != 'All'}
						use:menu={{ menu: 'create-menu' }}>Add Item</button>
					<nav class="list-nav card p-2 w-34 shadow-xl" data-menu="create-menu">
						<ul>
							<li>
								<button
									class="option w-full"
									on:click={() => createResource('File')}>
									<i class="mdi mdi-folder mr-2" />
									File/Folder</button>
							</li>
							<li>
								<button
									class="option w-full"
									on:click={() => createResource('WebURL')}>
									<i class="mdi mdi-web mr-2" />
									WebURL</button>
							</li>
							<li>
								<button
									on:click={() => createResource('Snippet')}
									class="option w-full">
									<i class="mdi mdi-code-braces mr-2" />
									Snippet</button>
							</li>
							<li>
								<button
									on:click={() => createResource('Note')}
									class="option w-full">
									<i class="mdi mdi-file-edit mr-2" />
									Note</button>
							</li>
						</ul>
					</nav>
				</span>
			</div>
		</div>
	</svelte:fragment>
	<div class="h-full px-1">
		{#each searchItems as item, i}
			{#if ['File', 'Folder'].includes(item.type)}
				<ItemResource {item} manage={true} on:edit={() => editResource(item)} />
			{:else if item.type == 'WebURL'}
				<ItemUrl {item} manage={true} on:edit={() => editResource(item)} />
			{:else if item.type == 'Note'}
				<ItemNote {item} manage={true} on:edit={() => openNote(item)} />
			{:else if item.type == 'Snippet'}
				<ItemSnippet {item} manage={true} on:edit={() => editResource(item)} />
			{/if}
		{:else}
			<div
				class="h-full flex flex-col justify-center items-center text-3xl font-bold text-surface-500">
				<i class="mdi mdi-text-search" />
				<span>Empty Catalog</span>
			</div>
		{/each}
	</div>
</AppShell>
<Modal width="w-full md:w-2/3" />
<Toast padding="p2 px-4" buttonAction="btn bg-secondary-500 p-2" buttonDismiss="p-2 bg-white" />
