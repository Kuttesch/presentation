<script lang="ts">
  import { onMount } from "svelte";

  let slides: string[] = [];
  let index = 0;

  // visibility state
  let controlsVisible = false;
  let cursorVisible = false;
  let hideTimeout: number | null = null;

  function next() {
    if (index < slides.length - 1) index++;
  }

  function prev() {
    if (index > 0) index--;
  }

  function showTemporarily() {
    controlsVisible = true;
    cursorVisible = true;

    if (hideTimeout) clearTimeout(hideTimeout);
    hideTimeout = window.setTimeout(() => {
      controlsVisible = false;
      cursorVisible = false;
    }, 1500);
  }

  onMount(async () => {
    const res = await fetch("/presentation.html");
    const html = await res.text();
    const doc = new DOMParser().parseFromString(html, "text/html");

    // Inject all <style> from HTML
    doc.querySelectorAll("style").forEach((s) => {
      const el = document.createElement("style");
      el.textContent = s.textContent;
      document.head.appendChild(el);
    });

    // Inject font + FA <link>
    doc.querySelectorAll("link").forEach((l) => {
      const clone = document.createElement("link");
      [...l.attributes].forEach((a) => clone.setAttribute(a.name, a.value));
      document.head.appendChild(clone);
    });

    // Grab slides
    slides = [...doc.querySelectorAll(".slide-container")].map((n) =>
      n.outerHTML
    );

    // Keyboard nav
    window.addEventListener("keydown", (e) => {
      // showTemporarily();
      if (e.key === "ArrowRight") next();
      if (e.key === "ArrowLeft") prev();
      if (e.key === " ") {
        e.preventDefault();
        next();
      }
    });

    // Mouse movement triggers visibility
    window.addEventListener("mousemove", showTemporarily);
    window.addEventListener("mouseenter", showTemporarily);

    showTemporarily();
  });

  // sync body cursor state
  $: document.body.classList.toggle("cursor-visible", cursorVisible);
</script>

<!-- Viewer -->
<div
  class="w-screen h-screen overflow-y-auto overflow-x-hidden flex flex-col items-center bg-[#21262d]"
  class:cursor-visible={cursorVisible}
  on:click={next}
  on:mousemove={showTemporarily}
  role="presentation"
>
  {#if slides.length}
    {@html slides[index]}
  {/if}
</div>

<!-- Controls -->
<div
  class="fixed bottom-6 left-1/2 -translate-x-1/2 flex items-center gap-6 z-50
         pt-2 pb-2 px-10 rounded-2xl border-2 border-[#30363d] bg-[#0d1117]
         transition-opacity duration-300"
  style="opacity: {controlsVisible ? 1 : 0}; pointer-events: {controlsVisible ? 'auto' : 'none'}"
  on:mouseenter={() => (controlsVisible = true)}
  on:mouseleave={showTemporarily}
  role="presentation"
>
  <!-- Prev -->
  <button
    class="text-xl px-4 py-2 rounded-md border-2 border-[#30363d]
           bg-[#0d1117] text-[#c9d1d9]
           disabled:opacity-30 hover:border-[#7D838A]"
    on:click|stopPropagation={prev}
    disabled={index === 0}
  >
    ←
  </button>

  <!-- Counter -->
  <div class="text-[#c9d1d9] text-lg font-mono select-none">
    {index + 1} / {slides.length}
  </div>

  <!-- Next -->
  <button
    class="text-xl px-4 py-2 rounded-md border-2 border-[#30363d]
           bg-[#0d1117] text-[#c9d1d9]
           disabled:opacity-30 hover:border-[#7D838A]"
    on:click|stopPropagation={next}
    disabled={index === slides.length - 1}
  >
    →
  </button>
</div>

<style>
  :global(html),
  :global(body) {
    margin: 0 !important;
    padding: 0 !important;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
  }

  /* hide cursor globally unless flagged visible */
  :global(html),
  :global(body),
  .viewer {
    cursor: none;
  }

  :global(body.cursor-visible),
  .viewer.cursor-visible {
    cursor: default;
  }

  /* preserve spacing from original HTML theme */
  :global(.slide-container) {
    margin-top: 3vw;
    margin-bottom: 3vw;
  }
</style>
