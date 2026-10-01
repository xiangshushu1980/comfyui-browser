<script lang="ts">
  // Owner: Sean
  export let file: any = null;
  export let x = 0;
  export let y = 0;
</script>

{#if file}
  <aside
    class="pointer-events-none fixed z-[1000] w-72 overflow-hidden rounded-lg border border-red-400/80 bg-slate-950/95 text-white shadow-[0_0_22px_4px_rgba(239,68,68,.35)] backdrop-blur"
    style={`left:${x}px;top:${y}px`}
    aria-live="polite"
  >
    {#if file.fileType === 'image'}
      <img class="h-28 w-full bg-black object-contain" src={file.previewUrl || file.url} alt="" />
    {:else if file.fileType === 'video'}
      <video class="h-28 w-full bg-black object-contain" src={file.previewUrl || file.url} muted autoplay loop playsinline></video>
    {/if}
    <div class="space-y-1 p-2 text-xs">
      <strong class="block break-all text-sm">{file.name}</strong>
      <div class="flex flex-wrap gap-x-3 text-white/75">
        <span>{file.fileType || file.type || 'file'}</span>
        {#if file.formattedSize}<span>{file.formattedSize}</span>{/if}
        {#if file.formattedDatetime}<span>{file.formattedDatetime}</span>{/if}
      </div>
      {#if file.folder_path}<p class="break-all text-white/60">{file.folder_path}</p>{/if}
      <p class="pt-1 font-semibold text-red-300">Delete this item?</p>
    </div>
  </aside>
{/if}
