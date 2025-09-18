<template>
  <div class="mermaid-container" :class="props.class" :data-mermaid-id="props.id">
    <div v-html="svg" class="mermaid-content"></div>
      <button
        class="mermaid-fullscreen-btn"
        @click="toggleFullscreen"
        title="View in fullscreen"
      >
      <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"><path fill="currentColor" d="M3 21v-5h2v3h3v2zm13 0v-2h3v-3h2v5zM3 8V3h5v2H5v3zm16 0V5h-3V3h5v5z"/></svg>
    </button>
  </div>
  <Teleport to="body" :disabled="!isFullscreen">
    <DragMermaid :svg="svg" :id="props.id" :class="props.class" v-if="isFullscreen" @exitFullScreen="exitFullScreen"></DragMermaid>
  </Teleport>
</template>

<script setup>
import { onMounted, onUnmounted, ref, toRaw, computed } from "vue";
import DragMermaid  from './DragMermaid.vue'
import { render, init } from "./mermaid";

//get mermaid settings
import { useData } from "vitepress";

const pluginSettings = ref({
  securityLevel: "loose",
  startOnLoad: false,
  externalDiagrams: [],
});
const { page } = useData();
const { frontmatter } = toRaw(page.value);
const mermaidPageTheme = frontmatter.mermaidTheme || "";

const props = defineProps({
  graph: {
    type: String,
    required: true,
  },
  id: {
    type: String,
    required: true,
  },
  class:{
    type: String,
    required: false,
    default: "mermaid",
  }
});

const svg = ref(null);
let mut = null;
const isFullscreen = ref(false);

const toggleFullscreen = () => {
  isFullscreen.value = !isFullscreen.value;
};

const exitFullScreen = () => {
  isFullscreen.value = false
}
onMounted(async () => {
  await init(pluginSettings.value.externalDiagrams);
  let settings = await import("virtual:mermaid-config");
  if (settings?.default) pluginSettings.value = settings.default;

  mut = new MutationObserver(async () => await renderChart());
  mut.observe(document.documentElement, { attributes: true });
  await renderChart();

  //refresh images on first render
  const hasImages =
    /<img([\w\W]+?)>/.exec(decodeURIComponent(props.graph))?.length > 0;
  if (hasImages)
    setTimeout(() => {
      let imgElements = document.getElementsByTagName("img");
      let imgs = Array.from(imgElements);
      if (imgs.length) {
        Promise.all(
          imgs
            .filter((img) => !img.complete)
            .map(
              (img) =>
                new Promise((resolve) => {
                  img.onload = img.onerror = resolve;
                })
            )
        ).then(async () => {
          await renderChart();
        });
      }
    }, 100);
});

onUnmounted(() => mut.disconnect());

const renderChart = async () => {
  const hasDarkClass = document.documentElement.classList.contains("dark");
  let mermaidConfig = {
    ...pluginSettings.value,
  };

  if (mermaidPageTheme) mermaidConfig.theme = mermaidPageTheme;
  if (hasDarkClass) mermaidConfig.theme = "dark";

  let svgCode = await render(
    props.id,
    decodeURIComponent(props.graph),
    mermaidConfig
  );
  // This is a hack to force v-html to re-render, otherwise the diagram disappears
  // when **switching themes** or **reloading the page**.
  // The cause is that the diagram is deleted during rendering (out of Vue's knowledge).
  // Because svgCode does NOT change, v-html does not re-render.
  // This is not required for all diagrams, but it is required for c4c, mindmap and zenuml.
  const salt = Math.random().toString(36).substring(7);
  svg.value = `${svgCode} <span style="display: none">${salt}</span>`;
};
</script>
<style scoped>
.mermaid-container {
  position: fixed;
  width: 100%;
  z-index: 9999;
  left: 0;
}
.mermaid-container {
  position: relative;
  width: 100%;
}
.mermaid-fullscreen-btn {
  position: absolute;
  bottom: 8px;
  right: 8px;
  background-color: var(--vp-code-copy-code-bg);
  border: 1px solid var(--vp-code-copy-code-border-color);
  color: var(--vp-code-copy-code-active-text);
  border-radius: 4px;
  padding: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
}
.mermaid-fullscreen-btn:hover {
  background-color: var(--vp-code-copy-code-hover-bg);
  border-color: var(--vp-code-copy-code-hover-border-color);
}
</style>