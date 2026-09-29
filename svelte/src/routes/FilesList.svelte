<script lang="ts">
  import { onDestroy, onMount } from 'svelte';
  import { onLoadWorkflow, onScroll, fetchFiles, WHITE_EXTS } from './utils';
  import { t } from 'svelte-i18n';
  import type { FOLDER_TYPES } from './utils';
  import MediaShow from './MediaShow.svelte';
  import type Toast from './Toast.svelte';
  import filenameColorConfig from './filename-color-config.json';

  export let comfyUrl: string;
  export let folderType: FOLDER_TYPES;
  export let toast: Toast;
  export let folderPath: string;
  $: if (folderPath != undefined) {
    refresh();
  }

  let comfyApp: any;
  let files: Array<any> = [];
  let filesRequestId = 0;
  let loaded: boolean = true;
  let showCursor = 20;
  let searchQuery = '';
  let candidateQuery = '';
  let searchCandidates: Array<any> = [];
  let selectedSearchCandidate = '';
  let showSearchCandidates = false;
  let searchInput: HTMLInputElement;
  let startDate = '';
  let endDate = '';
  let datePreset: 'all' | 'today' | '3days' | 'week' = 'all';
  let filteredFiles: Array<any> = [];
  let scrollTop = 0;
  let folderHistory = [''];
  let folderHistoryIndex = 0;
  let canGoBack = false;
  let canGoForward = false;
  let selectedFile: any = null;
  let hoveredFile: any = null;
  let browserRoots: Array<any> = [{ id: 'outputs', name: 'Output', path: '' }];
  let activeRootId = 'outputs';
  let activeRoot: any = browserRoots[0];
  let currentPathSegments: string[] = [];
  let breadcrumbOrientation: 'horizontal' | 'vertical' = 'horizontal';
  let viewMode: 'full' | 'side' = 'full';
  let newRootPath = '';
  let rootError = '';
  let rootSettingsOpen = false;
  let pickerPath = '';
  let pickerParent: string | null = null;
  let pickerDirectories: Array<any> = [];
  let pickerError = '';
  let pickerNotice = '';
  let pickerRequestId = 0;
  let pickerLoading = false;
  let standaloneBrowser = false;
  let standalonePreviewText = '';

  $: searchCandidates = candidateQuery.trim()
    ? files.filter((file) => file.name.toLowerCase().includes(candidateQuery.trim().toLowerCase())).slice(0, 8)
    : [];
  $: activeRoot = folderType === 'outputs'
    ? (browserRoots.find((root) => root.id === activeRootId) || browserRoots[0])
    : null;
  $: currentPathSegments = (folderPath || '').split('/').filter(Boolean);
  $: filteredFiles = files.filter((file) => {
    if (!file.name.toLowerCase().includes(searchQuery.trim().toLowerCase())) return false;
    if (!startDate && !endDate) return true;
    const createdAt = Number(file.created_at);
    if (!Number.isFinite(createdAt)) return false;
    const date = dateInputValue(new Date(createdAt * 1000));
    return (!startDate || date >= startDate) && (!endDate || date <= endDate);
  });

  function wordColor(word: string) {
    const normalized = word.toLowerCase();
    if (filenameColorConfig.wordColors[normalized]) return filenameColorConfig.wordColors[normalized];
    for (const category of Object.values(filenameColorConfig.categories)) {
      if (category.terms.includes(normalized)) return category.color;
    }
    if (normalized.length === 1 && filenameColorConfig.stageLetters[normalized]) return filenameColorConfig.stageLetters[normalized];
    let hash = 0;
    for (const char of normalized) hash = (hash * 31 + char.codePointAt(0)!) >>> 0;
    return filenameColorConfig.fallbackPalette[hash % filenameColorConfig.fallbackPalette.length];
  }

  function filenameSegments(name: string, isFile: boolean) {
    const extensionIndex = isFile ? name.lastIndexOf('.') : -1;
    const hasExtension = extensionIndex > 0;
    const stem = (hasExtension ? name.slice(0, extensionIndex) : name).replace(/_/g, ' ');
    const segments: Array<{ text: string; color: string }> = (stem.match(/\p{L}+|\p{N}+|[^\p{L}\p{N}]+/gu) || [stem]).map((text) => ({
      text: /^\s+$/u.test(text) ? '\u00a0' : text,
      color: /^\p{N}+$/u.test(text)
        ? filenameColorConfig.numberColor
        : /^\p{L}+$/u.test(text)
          ? wordColor(text)
          : 'rgba(255,255,255,.55)',
    }));
    return segments;
  }

  function filenameColumns(name: string, isFile: boolean) {
    const segments = filenameSegments(name, isFile);
    const midpoint = Math.ceil(segments.length / 2);
    let splitAt = segments.findIndex((part, index) => index >= midpoint && part.text === '\u00a0');
    if (splitAt < 0) splitAt = midpoint;
    return [segments.slice(0, splitAt), segments.slice(splitAt).filter((part) => part.text !== '\u00a0')];
  }

  function dateInputValue(date: Date) {
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    const day = String(date.getDate()).padStart(2, '0');
    return `${year}-${month}-${day}`;
  }

  function setDatePreset(preset: 'all' | 'today' | '3days' | 'week') {
    if (datePreset === preset) preset = 'all';
    datePreset = preset;
    showCursor = 20;
    if (preset === 'all') {
      startDate = '';
      endDate = '';
      return;
    }
    const today = new Date();
    const start = new Date(today.getFullYear(), today.getMonth(), today.getDate());
    if (preset === '3days') start.setDate(start.getDate() - 2);
    if (preset === 'week') start.setDate(start.getDate() - 6);
    startDate = dateInputValue(start);
    endDate = dateInputValue(today);
  }

  function onSearchInput() {
    candidateQuery = searchQuery;
    selectedSearchCandidate = '';
    showSearchCandidates = true;
    showCursor = 20;
  }

  function selectSearchCandidate(name: string) {
    selectedSearchCandidate = name;
    searchQuery = name;
    showSearchCandidates = false;
    showCursor = 20;
  }

  function onSearchKeydown(event: KeyboardEvent) {
    if ((event.ctrlKey || event.metaKey) && event.key.toLowerCase() === 'f') {
      event.preventDefault();
      searchInput?.focus();
      searchInput?.select();
      return;
    }
    if ((event.key === 'ArrowDown' || event.key === 'ArrowUp') && searchCandidates.length > 0) {
      event.preventDefault();
      showSearchCandidates = true;
      const currentIndex = searchCandidates.findIndex((candidate) => candidate.name === selectedSearchCandidate);
      const nextIndex = event.key === 'ArrowDown'
        ? (currentIndex + 1) % searchCandidates.length
        : (currentIndex <= 0 ? searchCandidates.length - 1 : currentIndex - 1);
      selectSearchCandidate(searchCandidates[nextIndex].name);
      showSearchCandidates = true;
      return;
    }
    if (event.key === 'Enter') {
      event.preventDefault();
      if (searchCandidates.length > 0 && !selectedSearchCandidate) selectSearchCandidate(searchCandidates[0].name);
      else showSearchCandidates = false;
    } else if (event.key === 'Escape') {
      event.preventDefault();
      searchQuery = '';
      candidateQuery = '';
      selectedSearchCandidate = '';
      showSearchCandidates = false;
      showCursor = 20;
    }
  }

  function onSearchShortcut(event: KeyboardEvent) {
    if ((event.ctrlKey || event.metaKey) && event.key.toLowerCase() === 'f') onSearchKeydown(event);
  }

  function toggleHostViewMode() {
    if (window.top === window) {
      setViewMode(viewMode === 'side' ? 'full' : 'side');
      return;
    }
    window.top?.postMessage({ source: 'comfyui-browser', type: 'toggle-view-mode' }, '*');
  }

  function sizeColor(bytes: number) {
    if (bytes >= 1024 ** 3) return '#fca5a5';
    if (bytes >= 500 * 1024 ** 2) return '#fdba74';
    if (bytes >= 100 * 1024 ** 2) return '#fcd34d';
    if (bytes >= 10 * 1024 ** 2) return '#bef264';
    if (bytes >= 1024 ** 2) return '#86efac';
    return '#a5f3fc';
  }

  function dateColor(createdAt: number) {
    const created = new Date(createdAt * 1000);
    const ageMs = Math.max(0, Date.now() - created.getTime());
    if (ageMs < 60 * 60 * 1000) return '#ffffff';
    if (ageMs < 3 * 60 * 60 * 1000) return '#f4f4f5';
    const now = new Date();
    const isToday = created.getFullYear() === now.getFullYear()
      && created.getMonth() === now.getMonth()
      && created.getDate() === now.getDate();
    if (isToday) return '#e4e4e7';
    return '#d4d4d8';
  }

  $: tt = function(key: string) {
    return $t('filesList.' + key);
  }

  export async function refresh() {
    const requestId = ++filesRequestId;
    const requestedFolder = folderPath;
    const requestedRoot = activeRootId;
    loaded = false;
    try {
      const nextFiles = await fetchFiles(folderType, comfyUrl, requestedFolder, requestedRoot);
      if (requestId === filesRequestId) files = nextFiles;
    } finally {
      if (requestId === filesRequestId) loaded = true;
    }
  }

  onMount(async () => {
    standaloneBrowser = window.top === window;
    const savedOrientation = localStorage.getItem('comfyui-browser-breadcrumb-orientation');
    if (savedOrientation === 'horizontal' || savedOrientation === 'vertical') breadcrumbOrientation = savedOrientation;
    //@ts-ignore
    comfyApp = window.top.app;

    //@ts-ignore
    window.top.addEventListener("comfyuiBrowserShow", () => {
      refresh();
    });

    window.addEventListener('message', onHostMessage);
    viewMode = hostViewMode() || viewMode;
    if (folderType === 'outputs') {
      try {
        const response = await fetch(comfyUrl + '/browser/roots');
        if (response.ok) {
          const result = await response.json();
          browserRoots = result.roots || browserRoots;
        }
      } catch {
        rootError = 'Could not load directories';
      }
      viewMode = hostViewMode() || viewMode;
      const storedRoot = localStorage.getItem(rootStorageKey()) || localStorage.getItem(`comfyui-browser-root-${viewMode}`);
      if (browserRoots.some((root) => root.id === storedRoot)) activeRootId = storedRoot;
      const storedFolder = localStorage.getItem(folderStorageKey()) ?? localStorage.getItem(`comfyui-browser-folder-${viewMode}`) ?? '';
      folderPath = storedFolder;
      localStorage.setItem(rootStorageKey(), activeRootId);
      localStorage.setItem(folderStorageKey(), folderPath);
    }

    resetFolderHistory();

    window.addEventListener('scroll', (e) => {
      showCursor = onScroll(showCursor, files.length);
      //@ts-ignore
      scrollTop = (e.target.scrollingElement as HTMLElement).scrollTop;
    });
    window.addEventListener('keydown', onFolderShortcut);
    window.addEventListener('keydown', onFileShortcut);
    window.addEventListener('keydown', onSearchShortcut);
    window.addEventListener('auxclick', onMouseNavigation);
  });

  onDestroy(() => {
    if (typeof window !== 'undefined') {
      window.removeEventListener('keydown', onFolderShortcut);
      window.removeEventListener('keydown', onFileShortcut);
      window.removeEventListener('keydown', onSearchShortcut);
      window.removeEventListener('auxclick', onMouseNavigation);
      window.removeEventListener('message', onHostMessage);
      postPreview(null);
    }
  });

  function rootStorageKey() {
    return 'comfyui-browser-root';
  }

  function folderStorageKey() {
    return 'comfyui-browser-folder';
  }

  function hostViewMode(): 'full' | 'side' | null {
    try {
      const mode = (window.top as any).__comfyuiBrowserViewMode;
      return mode === 'full' || mode === 'side' ? mode : null;
    } catch {
      return null;
    }
  }

  function syncViewMode(): 'full' | 'side' {
    const mode = hostViewMode();
    if (mode && mode !== viewMode) setViewMode(mode);
    return mode || viewMode;
  }

  function onHostMessage(event: MessageEvent) {
    if (event.source !== window.top || event.data?.source !== 'comfyui-browser-host') return;
    if (event.data.type === 'view-mode' && (event.data.mode === 'full' || event.data.mode === 'side')) {
      setViewMode(event.data.mode);
    }
    if (event.data.type === 'dismiss-root-settings') rootSettingsOpen = false;
    if (event.data.type === 'preview-closed') selectedFile = null;
    if (event.data.type === 'preview-navigate') navigatePreview(Number(event.data.direction));
  }

  function navigatePreview(direction: number) {
    if (!selectedFile || !direction) return;
    const previewable = files.filter((file) => file.type === 'file' && file.name.toLowerCase().includes(searchQuery.trim().toLowerCase())).slice(0, showCursor);
    const index = previewable.findIndex((file) => file.name === selectedFile.name && file.folder_path === selectedFile.folder_path && file.root_id === activeRootId);
    const next = previewable[index + direction];
    if (next) void onSelectFile(next);
  }

  function setViewMode(mode: 'full' | 'side') {
    viewMode = mode;
  }

  async function onRootChange(event: Event) {
    syncViewMode();
    activeRootId = (event.currentTarget as HTMLSelectElement).value;
    localStorage.setItem(rootStorageKey(), activeRootId);
    folderPath = '';
    localStorage.setItem(folderStorageKey(), folderPath);
    resetFolderHistory();
    refresh();
    const activeRoot = browserRoots.find((root) => root.id === activeRootId);
    await loadPickerDirectory(activeRoot?.path);
  }

  async function loadPickerDirectory(path?: string) {
    const requestId = ++pickerRequestId;
    pickerLoading = true;
    pickerError = '';
    pickerNotice = '';
    try {
      const params = new URLSearchParams();
      if (path) params.set('path', path);
      const response = await fetch(`${comfyUrl}/browser/roots/directories?${params.toString()}`);
      const result = await response.json();
      if (!response.ok) throw new Error(result.message || 'Could not read directory');
      if (requestId !== pickerRequestId) return;
      pickerPath = result.path;
      pickerParent = result.parent;
      pickerDirectories = result.directories || [];
    } catch (error) {
      if (requestId === pickerRequestId) pickerError = String(error);
    } finally {
      if (requestId === pickerRequestId) pickerLoading = false;
    }
  }

  async function selectPickerDirectory(path: string) {
    await loadPickerDirectory(path);
    await copyDirectoryPath(path);
  }

  async function copyDirectoryPath(path: string) {
    if (!path) return;
    try {
      await navigator.clipboard.writeText(path);
      pickerNotice = `Copied: ${path}`;
    } catch {
      pickerError = 'Could not copy path';
    }
  }

  async function toggleRootSettings() {
    rootSettingsOpen = !rootSettingsOpen;
    if (rootSettingsOpen) {
      const activeRoot = browserRoots.find((root) => root.id === activeRootId);
      await loadPickerDirectory(activeRoot?.path);
    }
  }

  async function addBrowserRoot() {
    syncViewMode();
    rootError = '';
    const response = await fetch(comfyUrl + '/browser/roots', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ path: newRootPath }),
    });
    if (!response.ok) {
      const result = await response.json().catch(() => ({}));
      rootError = result.message || 'Could not add directory';
      return;
    }
    const result = await response.json();
    const rootsResponse = await fetch(comfyUrl + '/browser/roots');
    const rootsResult = await rootsResponse.json();
    browserRoots = rootsResult.roots || browserRoots;
    activeRootId = result.root.id;
    localStorage.setItem(rootStorageKey(), activeRootId);
    newRootPath = '';
    folderPath = '';
    localStorage.setItem(folderStorageKey(), folderPath);
    resetFolderHistory();
    refresh();
    await loadPickerDirectory(result.root.path);
  }

  async function removeBrowserRoot() {
    syncViewMode();
    if (activeRootId === 'outputs') return;
    const response = await fetch(comfyUrl + `/browser/roots/${encodeURIComponent(activeRootId)}`, { method: 'DELETE' });
    if (!response.ok) return;
    browserRoots = browserRoots.filter((root) => root.id !== activeRootId);
    activeRootId = 'outputs';
    localStorage.setItem(rootStorageKey(), activeRootId);
    folderPath = '';
    localStorage.setItem(folderStorageKey(), folderPath);
    resetFolderHistory();
    refresh();
    const outputRoot = browserRoots.find((root) => root.id === 'outputs');
    await loadPickerDirectory(outputRoot?.path);
  }

  function updateFolderHistoryButtons() {
    canGoBack = folderHistoryIndex > 0;
    canGoForward = folderHistoryIndex < folderHistory.length - 1;
  }

  function resetFolderHistory(path = '') {
    folderHistory = [path];
    folderHistoryIndex = 0;
    updateFolderHistoryButtons();
  }

  function navigateToFolder(path: string) {
    if (path === folderPath) return;
    syncViewMode();
    selectedFile = null;
    postPreview(null);
    folderHistory = folderHistory.slice(0, folderHistoryIndex + 1);
    folderHistory.push(path);
    folderHistoryIndex = folderHistory.length - 1;
    folderPath = path;
    if (folderType === 'outputs') localStorage.setItem(folderStorageKey(), folderPath);
    updateFolderHistoryButtons();
  }

  function onClickPath(index: number) {
    const nextPath = index < 0 ? '' : currentPathSegments.slice(0, index + 1).join('/');
    navigateToFolder(nextPath);
  }

  function toggleBreadcrumbOrientation() {
    breadcrumbOrientation = breadcrumbOrientation === 'horizontal' ? 'vertical' : 'horizontal';
    localStorage.setItem('comfyui-browser-breadcrumb-orientation', breadcrumbOrientation);
  }

  function goBack() {
    if (!canGoBack) return;
    syncViewMode();
    folderHistoryIndex -= 1;
    folderPath = folderHistory[folderHistoryIndex];
    if (folderType === 'outputs') localStorage.setItem(folderStorageKey(), folderPath);
    updateFolderHistoryButtons();
  }

  function goForward() {
    if (!canGoForward) return;
    syncViewMode();
    folderHistoryIndex += 1;
    folderPath = folderHistory[folderHistoryIndex];
    if (folderType === 'outputs') localStorage.setItem(folderStorageKey(), folderPath);
    updateFolderHistoryButtons();
  }

  function onFolderShortcut(event: KeyboardEvent) {
    const target = event.target as HTMLElement | null;
    if (target?.closest('input, textarea, select, [contenteditable="true"]')) return;
    if (!event.altKey || event.ctrlKey || event.metaKey || event.shiftKey) return;
    if (event.key !== 'ArrowLeft' && event.key !== 'ArrowRight') return;

    event.preventDefault();
    event.stopPropagation();
    event.key === 'ArrowLeft' ? goBack() : goForward();
  }

  function onFileShortcut(event: KeyboardEvent) {
    const target = event.target as HTMLElement | null;
    if (!hoveredFile || target?.closest('input, textarea, select, [contenteditable="true"]')) return;
    if (event.ctrlKey || event.metaKey || event.altKey || event.shiftKey) return;
    const key = event.key.toLowerCase();
    if (!['s', 'l', 'd'].includes(key)) return;
    if (key === 'l' && (!comfyApp || hoveredFile.type === 'dir')) return;

    event.preventDefault();
    event.stopPropagation();
    if (key === 's') void onToggleFavorite(hoveredFile);
    if (key === 'l') void onLoadWorkflow(hoveredFile, comfyApp, toast);
    if (key === 'd') void onDelete(hoveredFile);
  }

  function onMouseNavigation(event: MouseEvent) {
    if (event.button !== 3 && event.button !== 4) return;
    event.preventDefault();
    event.stopPropagation();
    event.button === 3 ? goBack() : goForward();
  }

  async function onSelectFile(file: any) {
    if (selectedFile && selectedFile.name === file.name && selectedFile.folder_path === file.folder_path && selectedFile.root_id === activeRootId) {
      selectedFile = null;
      postPreview(null);
      return;
    }
    selectedFile = file;
    if (standaloneBrowser) {
      standalonePreviewText = '';
      if (file.fileType === 'text' || file.fileType === 'html' || file.fileType === 'json') {
        try {
          const response = await fetch(file.url);
          standalonePreviewText = response.ok ? await response.text() : `Could not load preview: HTTP ${response.status}`;
        } catch (error) {
          standalonePreviewText = `Could not load preview: ${error}`;
        }
      }
    } else {
      postPreview({
        name: file.name,
        fileType: file.fileType,
        url: file.url,
        previewUrl: file.previewUrl || file.url,
        formattedSize: file.formattedSize,
        rootId: activeRootId,
      });
    }
  }

  function postPreview(file: any) {
    if (window.top !== window) {
      window.top?.postMessage({ source: 'comfyui-browser', type: 'preview', file, mode: viewMode }, '*');
    }
  }

  async function onToggleFavorite(file: any) {
    const nextValue = !file.is_favorite;
    const res = await fetch(comfyUrl + '/browser/favorites', {
      method: 'POST',
      body: JSON.stringify({
        filename: file.name,
        folder_path: file.folder_path,
        folder_type: folderType,
        root_id: activeRootId,
        favorite: nextValue,
      }),
    });
    if (res.ok) {
      file.is_favorite = nextValue;
      files = [...files];
    }
    toast.show(res.ok, nextValue ? 'Added to Saves' : 'Removed from Saves', 'Failed to update Saves');
  }

  async function scrollToTop() {
    window.scrollTo(0, 0);
  }


  async function onDelete(file: any) {
    const ret = confirm(tt('You want to delete this file?') + ' ' + file.name);
    if (!ret) {
      return;
    }

    const res = await fetch(comfyUrl + '/browser/files', {
      method: 'DELETE',
      body: JSON.stringify({
        folder_type: folderType,
        root_id: activeRootId,
        filename: file.name,
        folder_path: file.folder_path,
      }),
    });

    refresh();
    toast.show(
      res.ok,
      tt('Deleted the file') + file.name,
      tt('Failed to delete the file'),
    );
  }

  async function onClickDir(dir: any) {
    navigateToFolder(dir.path);
  }

  function onClickCard(file: any, event: MouseEvent) {
    const target = event.target as HTMLElement | null;
    if (target?.closest('button')) return;
    if (file.type === 'dir') void onClickDir(file);
    else void onSelectFile(file);
  }

  function onCardKeydown(file: any, event: KeyboardEvent) {
    if (event.key !== 'Enter' && event.key !== ' ') return;
    event.preventDefault();
    onClickCard(file, event as unknown as MouseEvent);
  }

</script>

<div class="relative z-30 ml-3 mt-2 flex min-w-0 flex-nowrap items-center gap-1 text-sm">
  <div class="flex shrink-0 items-center gap-1">
    <div class="flex shrink-0 items-center gap-0.5">
      <button class="btn btn-ghost btn-sm px-1" aria-label="Back" title="Back (Alt+←)" disabled={!canGoBack} on:click={goBack}>
        <svg viewBox="0 0 24 24" class="w-4 h-4" fill="none" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M15 18l-6-6 6-6" /></svg>
      </button>
      <button class="btn btn-ghost btn-sm px-1" aria-label="Forward" title="Forward (Alt+→)" disabled={!canGoForward} on:click={goForward}>
        <svg viewBox="0 0 24 24" class="w-4 h-4" fill="none" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M9 18l6-6-6-6" /></svg>
      </button>
    </div>
  </div>
  <div class="min-w-0 flex-1" aria-hidden="true"></div>

  <div class="relative flex w-48 shrink-0 items-center">
    <input
      type="text"
      placeholder={tt('searchInput.placeholder')}
      bind:this={searchInput}
      bind:value={searchQuery}
      on:input={onSearchInput}
      on:keydown={onSearchKeydown}
      on:focus={() => { if (searchQuery.trim()) showSearchCandidates = true; }}
      class="input input-bordered input-sm min-w-0 flex-1 border-slate-600 text-sm"
    />
    {#if showSearchCandidates && searchCandidates.length > 0}
      <div class="absolute left-0 right-0 top-full mt-1 max-h-64 overflow-auto rounded-md border border-base-content/20 bg-base-200 shadow-xl">
        {#each searchCandidates as candidate}
          <button class="block w-full truncate px-3 py-2 text-left text-sm hover:bg-base-300 {selectedSearchCandidate === candidate.name ? 'bg-base-300' : ''}" title={candidate.name} on:click={() => selectSearchCandidate(candidate.name)}>
            {candidate.name}
          </button>
        {/each}
      </div>
    {/if}
  </div>
  <div class="flex shrink-0 items-center gap-0.5" aria-label="Date filter">
    <button class="btn btn-xs px-2 {datePreset === 'today' ? 'btn-active' : 'btn-ghost'}" aria-pressed={datePreset === 'today'} on:click={() => setDatePreset('today')}>今日</button>
    <button class="btn btn-xs px-2 {datePreset === '3days' ? 'btn-active' : 'btn-ghost'}" aria-pressed={datePreset === '3days'} on:click={() => setDatePreset('3days')}>3 日</button>
    <button class="btn btn-xs px-2 {datePreset === 'week' ? 'btn-active' : 'btn-ghost'}" aria-pressed={datePreset === 'week'} on:click={() => setDatePreset('week')}>一周</button>
  </div>
  <button class="btn btn-ghost btn-sm btn-square shrink-0" aria-label={viewMode === 'side' ? 'Switch to full view' : 'Switch to side view'} title={viewMode === 'side' ? 'Full view' : 'Side view'} on:click={toggleHostViewMode}>
    {#if viewMode === 'side'}
      <svg class="h-5 w-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path stroke-linecap="round" stroke-linejoin="round" d="M8 3H5a2 2 0 0 0-2 2v3m13-5h3a2 2 0 0 1 2 2v3M3 16v3a2 2 0 0 0 2 2h3m13-5v3a2 2 0 0 1-2 2h-3" /></svg>
    {:else}
      <svg class="h-5 w-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><rect x="3" y="4" width="18" height="16" rx="2" /><path stroke-linecap="round" d="M9 4v16" /></svg>
    {/if}
  </button>
  <button class="btn btn-ghost btn-sm btn-square shrink-0" aria-label="Open in new tab" title="Open in new tab" on:click={() => window.open(window.location.href, '_blank', 'noopener')}>
    <svg class="h-5 w-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path stroke-linecap="round" stroke-linejoin="round" d="M13.5 6H19v5.5M19 6l-8 8" /><path stroke-linecap="round" stroke-linejoin="round" d="M19 13v4.25A1.75 1.75 0 0 1 17.25 19h-10.5A1.75 1.75 0 0 1 5 17.25V6.75A1.75 1.75 0 0 1 6.75 5H11" /></svg>
  </button>

{#if folderType === 'outputs'}
  <div class="fixed left-14 top-14 z-[45] max-h-[45vh] max-w-[min(85vw,34rem)] rounded-md border border-white/15 bg-black/80 p-2 pr-8 text-xs leading-tight text-white shadow-xl backdrop-blur-sm">
    <nav class="flex max-h-[calc(45vh-1rem)] items-center gap-1 overflow-auto {breadcrumbOrientation === 'vertical' ? 'flex-col items-start' : 'flex-row whitespace-nowrap'}" aria-label="Current directory breadcrumb">
      <button class="max-w-full break-all py-0.5 text-left font-semibold hover:underline" title={activeRoot?.name || 'Root'} on:click={() => onClickPath(-1)}>{activeRoot?.name || 'Root'}</button>
      {#each currentPathSegments as segment, index}
        <span class="shrink-0 opacity-60" aria-hidden="true">{breadcrumbOrientation === 'vertical' ? '↓' : '›'}</span>
        <button class="max-w-full break-all py-0.5 text-left hover:underline" title={segment} on:click={() => onClickPath(index)}>{segment}</button>
      {/each}
    </nav>
    <button class="btn btn-ghost btn-xs btn-square absolute right-0.5 top-0.5 h-6 min-h-0 w-6" aria-label={breadcrumbOrientation === 'horizontal' ? 'Switch breadcrumb to vertical' : 'Switch breadcrumb to horizontal'} title={breadcrumbOrientation === 'horizontal' ? 'Vertical breadcrumb' : 'Horizontal breadcrumb'} on:click={toggleBreadcrumbOrientation}>
      {#if breadcrumbOrientation === 'horizontal'}
        <svg class="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path stroke-linecap="round" d="M5 7h14M5 12h14M5 17h14" /></svg>
      {:else}
        <svg class="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path stroke-linecap="round" d="M7 5v14M12 5v14M17 5v14" /></svg>
      {/if}
    </button>
  </div>
{/if}

{#if folderType === 'outputs' && rootSettingsOpen}
  <button class="fixed inset-0 z-40 cursor-default bg-transparent" aria-label="Close directory settings" on:click={() => { rootSettingsOpen = false; }}></button>
{/if}
{#if folderType === 'outputs'}
  <div class="relative z-50 shrink-0">
    <button class="btn btn-circle btn-sm shadow-lg" aria-label="Directory settings" title="Directory settings" on:click={toggleRootSettings}>
      {#if rootSettingsOpen}
        <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path stroke-linecap="round" stroke-linejoin="round" d="M6 6l12 12M18 6L6 18" /></svg>
      {:else}
        <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path stroke-linecap="round" stroke-linejoin="round" d="M3 7.5A1.5 1.5 0 014.5 6h5l2 2H19.5A1.5 1.5 0 0121 9.5v8a1.5 1.5 0 01-1.5 1.5h-15A1.5 1.5 0 013 17.5v-10z" /></svg>
      {/if}
    </button>
    {#if rootSettingsOpen}
      <section class="absolute right-0 top-full z-50 mt-2 flex h-[min(72vh,38rem)] w-[min(92vw,28rem)] flex-col overflow-hidden rounded-lg border border-base-content/20 bg-base-200/95 p-3 shadow-2xl backdrop-blur" aria-label="Directory settings">
        <div class="flex items-center gap-2">
          <label class="text-xs opacity-70" for="browser-root-select">Directory</label>
          <select id="browser-root-select" class="select select-bordered select-sm min-w-0 flex-1" value={activeRootId} on:change={onRootChange}>
            {#each browserRoots as root}<option value={root.id}>{root.name}</option>{/each}
          </select>
          {#if activeRootId !== 'outputs'}
            <button class="btn btn-ghost btn-xs text-error" on:click={removeBrowserRoot} title="Remove this directory">✕</button>
          {/if}
        </div>
        <div class="mt-3 flex min-h-0 flex-1 flex-col rounded border border-base-content/15 p-2">
          <div class="mb-2 flex items-center gap-2">
            <button class="btn btn-ghost btn-xs" disabled={!pickerParent || pickerLoading} on:click={() => pickerParent && loadPickerDirectory(pickerParent)}>Up</button>
            <button class="min-w-0 flex-1 truncate text-left text-xs hover:underline" title={`Click to copy: ${pickerPath}`} on:click={() => copyDirectoryPath(pickerPath)}>{pickerPath || 'Loading…'}</button>
            <button class="btn btn-primary btn-xs" disabled={!pickerPath || pickerLoading || browserRoots.some((root) => root.path === pickerPath)} on:click={async () => { newRootPath = pickerPath; await addBrowserRoot(); }}>Add this folder</button>
          </div>
          {#if pickerError}<p class="mb-2 text-xs text-error">{pickerError}</p>{/if}
          {#if pickerNotice}<p class="mb-2 truncate text-xs text-success" title={pickerNotice}>{pickerNotice}</p>{/if}
          <div class="min-h-0 flex-1 overflow-auto">
            {#if pickerLoading}
              <p class="p-2 text-xs opacity-60">Loading folders…</p>
            {:else if pickerDirectories.length === 0}
              <p class="p-2 text-xs opacity-60">No subfolders</p>
            {:else}
              {#each pickerDirectories as directory}
                <div class="flex items-center gap-1 rounded hover:bg-base-300">
                  <button class="flex min-w-0 flex-1 items-center gap-2 px-2 py-1.5 text-left text-sm" title={`Open ${directory.path} and copy its path`} on:click={() => selectPickerDirectory(directory.path)}>
                    <span class="truncate">{directory.name}</span>
                  </button>
                  <button class="btn btn-ghost btn-xs" aria-label={`Copy path for ${directory.name}`} title="Copy full path" on:click={() => copyDirectoryPath(directory.path)}>
                    <svg class="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><rect x="8" y="8" width="12" height="12" rx="2"/><path stroke-linecap="round" stroke-linejoin="round" d="M16 8V5.75A1.75 1.75 0 0 0 14.25 4h-8.5A1.75 1.75 0 0 0 4 5.75v8.5A1.75 1.75 0 0 0 5.75 16H8"/></svg>
                  </button>
                  <button class="btn btn-ghost btn-xs" aria-label={`Open ${directory.name}`} title="Open folder" on:click={() => loadPickerDirectory(directory.path)}>
                    <svg class="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="m9 5 7 7-7 7" /></svg>
                  </button>
                </div>
              {/each}
            {/if}
          </div>
        </div>
        {#if rootError}<p class="mt-2 text-xs text-error">{rootError}</p>{/if}
      </section>
    {/if}
  </div>
{/if}
</div>

<div class="grid w-full gap-2" style="grid-template-columns: repeat(auto-fill, minmax(min(100%, 220px), 1fr));">
  {#each filteredFiles.slice(0, showCursor) as file}
    {#if WHITE_EXTS.includes(file.fileType)}
      <div class="group relative flex h-full flex-col overflow-hidden rounded-lg bg-black" role="button" tabindex="0" on:click={(event) => onClickCard(file, event)} on:keydown={(event) => onCardKeydown(file, event)} on:pointerenter={() => hoveredFile = file} on:pointerleave={() => { if (hoveredFile === file) hoveredFile = null; }}>
        <MediaShow {file} styleClass="w-full min-h-24 flex-auto" {onClickDir} {onSelectFile} />

        <button
          class="btn btn-circle btn-ghost btn-xs absolute left-1 top-1 z-10 h-7 min-h-0 w-7 p-0 shadow backdrop-blur-sm hover:bg-black/50 {file.is_favorite ? 'bg-black/50 text-yellow-300' : 'bg-black/35 text-white'}"
          aria-pressed={!!file.is_favorite}
          aria-label={file.is_favorite ? 'Remove from Saves' : 'Add to Saves'}
          title={`${file.is_favorite ? 'Remove from Saves' : 'Add to Saves'} (S)`}
          on:click={async () => await onToggleFavorite(file)}
        >
          <svg class="h-[18px] w-[18px]" viewBox="0 0 24 24" fill={file.is_favorite ? 'currentColor' : 'none'} stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3 2.8 5.7 6.2.9-4.5 4.4 1.1 6.2-5.6-3-5.6 3 1.1-6.2L3 9.6l6.2-.9L12 3z"/></svg>
        </button>
        {#if file.type === 'dir'}
          <div class="pointer-events-none absolute left-1 right-1 top-1/2 z-[1] max-h-[calc(100%_-_4.5rem)] -translate-y-1/2 overflow-hidden text-center opacity-80">
            <div class="mx-auto grid max-h-[calc(100%_-_4.5rem)] w-full grid-cols-2 gap-3 font-bold leading-tight [text-shadow:0_1px_3px_rgba(0,0,0,.95)] text-[28px]" title={file.name}>
              {#each filenameColumns(file.name, false) as column, index}
                <p class="break-words {index === 0 ? 'text-left' : 'text-right'}">{#each column as part}<span class="block" style={`color:${part.color}`}>{part.text}</span>{/each}</p>
              {/each}
            </div>
          </div>
        {:else}
          <div class="pointer-events-none absolute left-1 right-1 top-1/2 z-[1] max-h-[calc(100%_-_4.5rem)] -translate-y-1/2 overflow-hidden opacity-80">
            <div class="mx-auto grid max-h-[calc(100%_-_4.5rem)] w-full grid-cols-2 gap-3 font-bold leading-tight [text-shadow:0_1px_3px_rgba(0,0,0,.95)] text-[28px]" title={file.name}>
              {#each filenameColumns(file.name, true) as column, index}
                <p class="break-words {index === 0 ? 'text-left' : 'text-right'}">{#each column as part}<span class="block" style={`color:${part.color}`}>{part.text}</span>{/each}</p>
              {/each}
            </div>
          </div>
        {/if}
        <div class="pointer-events-none absolute bottom-1 left-1 max-w-[calc(50%-0.35rem)] truncate rounded-md border border-white/15 bg-black/80 px-2 py-1 text-xs leading-4 shadow backdrop-blur-[3px]" style={`color:${dateColor(file.created_at)}`} title={`${file.formattedDatetime} · ${file.formattedSize}`}>
          <span>{file.formattedDatetime.split(' ')[0].slice(5)}</span>
          <span class="ml-1">{file.formattedDatetime.split(' ')[1].slice(0, 5)}</span>
        </div>
        <div class="pointer-events-none absolute bottom-1 right-1 max-w-[calc(50%-0.35rem)] truncate rounded-md border border-white/15 bg-black/80 px-2 py-1 text-xs font-semibold leading-4 shadow backdrop-blur-[3px]" style={`color:${sizeColor(file.bytes || 0)}`} title={file.formattedSize}>
          {file.formattedSize}
        </div>
      </div>
    {/if}
  {/each}
</div>


<div class="flex justify-center">
  {#if filteredFiles.length > showCursor}
    <button on:click={() => showCursor += 10} class="btn btn-neutral btn-outline">Load more</button>
  {:else}
    <p class="text-neutral-content">No more content.</p>
  {/if}
</div>

{#if standaloneBrowser && selectedFile}
  <button class="fixed inset-0 z-40 bg-black/20 cursor-default" aria-label="Close preview" title="Click outside to close" on:click={() => selectedFile = null}></button>
  <section class="fixed z-50 left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 w-[88vw] h-[86vh] rounded-lg border border-base-content/30 bg-base-200/90 shadow-2xl backdrop-blur-sm p-3 flex flex-col" role="dialog" aria-modal="true" aria-label={selectedFile.name}>
    <div class="flex items-center gap-2 mb-2">
      <strong class="truncate flex-1">{selectedFile.name}</strong>
      <span class="text-xs opacity-70">{selectedFile.formattedSize}</span>
      <button class="btn btn-ghost btn-sm" aria-label="Close preview" on:click={() => selectedFile = null}>✕</button>
    </div>
    <div class="min-h-0 flex-1 flex items-center justify-center overflow-auto">
      {#if selectedFile.fileType === 'image'}
        <img class="max-w-full max-h-full object-contain" src={selectedFile.previewUrl} alt={selectedFile.name} />
      {:else if selectedFile.fileType === 'video'}
        <video class="max-w-full max-h-full" src={selectedFile.previewUrl} controls playsinline><track kind="captions" /></video>
      {:else if selectedFile.fileType === 'audio'}
        <audio class="w-full max-w-2xl" src={selectedFile.url} controls />
      {:else}
        <pre class="w-full h-full overflow-auto whitespace-pre-wrap break-words rounded bg-base-100/80 p-3 text-xs">{standalonePreviewText}</pre>
      {/if}
    </div>
  </section>
{/if}

{#if files.length === 0}
  <div class="w-full h-full flex items-center justify-center">
    <span class="font-bold text-4xl">
      {#if loaded}
        {$t('common.emptyList')}
      {:else}
        {$t('common.loading')}
      {/if}
    </span>
  </div>
{/if}

{#if scrollTop >= 300}
<button class="fixed right-10 bottom-10 w-10 h-10 flex justify-center items-center	rounded-full bg-slate-50" on:click={scrollToTop}>
  <svg class="w-5 h-5 icon" viewBox="0 0 1024 1024" xmlns="http://www.w3.org/2000/svg">
    <path fill="#333333" d="M572.235 205.282v600.365a30.118 30.118 0 11-60.235 0V205.282L292.382 438.633a28.913 28.913 0 01-42.646 0 33.43 33.43 0 010-45.236l271.058-288.045a28.913 28.913 0 0142.647 0L834.5 393.397a33.43 33.43 0 010 45.176 28.913 28.913 0 01-42.647 0l-219.618-233.23z"/>
  </svg>
</button>
{/if}
