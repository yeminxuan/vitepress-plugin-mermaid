<template>
  <div class="mermaid-container" :class="props.class" :data-mermaid-id="props.id" ref="diagramContent">
    <div v-html="svg" class="mermaid-content" :style="contentStyle"></div>
    <button
        class="mermaid-fullscreen-btn"
        @click="toggleFullscreen"
        title="View in fullscreen"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"><path fill="currentColor" d="M3 21v-5h2v3h3v2zm13 0v-2h3v-3h2v5zM3 8V3h5v2H5v3zm16 0V5h-3V3h5v5z"/></svg>
    </button>
    <div v-if="isFullscreen" class="mermaid-zoom-controls">
      <button @click="zoomIn" title="Zoom in">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"><path fill="currentColor" d="M8.5 10.5h-1q-.425 0-.712-.288T6.5 9.5t.288-.712T7.5 8.5h1v-1q0-.425.288-.712T9.5 6.5t.713.288t.287.712v1h1q.425 0 .713.288t.287.712t-.288.713t-.712.287h-1v1q0 .425-.288.713T9.5 12.5t-.712-.288T8.5 11.5zm1 5.5q-2.725 0-4.612-1.888T3 9.5t1.888-4.612T9.5 3t4.613 1.888T16 9.5q0 1.1-.35 2.075T14.7 13.3l5.6 5.6q.275.275.275.7t-.275.7t-.7.275t-.7-.275l-5.6-5.6q-.75.6-1.725.95T9.5 16m0-2q1.875 0 3.188-1.312T14 9.5t-1.312-3.187T9.5 5T6.313 6.313T5 9.5t1.313 3.188T9.5 14"/></svg>
      </button>
      <button @click="zoomOut" title="Zoom out">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"><path fill="currentColor" d="m19.6 21l-6.3-6.3q-.75.6-1.725.95T9.5 16q-2.725 0-4.612-1.888T3 9.5t1.888-4.612T9.5 3t4.613 1.888T16 9.5q0 1.1-.35 2.075T14.7 13.3l6.3 6.3zM9.5 14q1.875 0 3.188-1.312T14 9.5t-1.312-3.187T9.5 5T6.313 6.313T5 9.5t1.313 3.188T9.5 14M7 10.5v-2h5v2z"/></svg>
      </button>
      <button @click="resetZoom" title="Reset zoom">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"><path fill="currentColor" d="m3.9 21.5l-1.4-1.4L5.6 17H3v-2h6v6H7v-2.6zm16.2 0L17 18.4V21h-2v-6h6v2h-2.6l3.1 3.1zM3 9V7h2.6L2.5 3.9l1.4-1.4L7 5.6V3h2v6zm12 0V3h2v2.6l3.1-3.1l1.4 1.4L18.4 7H21v2z"/></svg>
      </button>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref, toRaw, computed } from "vue";
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
const zoomLevel = ref(1);
const position = ref({ x: 0, y: 0 });
const diagramContent = ref(null);
const isDragging = ref(false);
const dragStart = ref({ x: 0, y: 0 });

const contentStyle = computed(() => ({
  transform: `scale(${zoomLevel.value}) translate(${position.value.x}px, ${position.value.y}px)`,
  transformOrigin: 'center center',
  cursor: isDragging.value ? 'grabbing' : (isFullscreen.value ? 'grab' : 'default')
}));

const zoomIn = () => {
  zoomLevel.value = Math.min(zoomLevel.value + 0.1, 3); // Max zoom 3x
};

const zoomOut = () => {
  zoomLevel.value = Math.max(zoomLevel.value - 0.1, 0.5); // Min zoom 0.5x
};

const resetZoom = () => {
  zoomLevel.value = 1;
  position.value = { x: 0, y: 0 };
};

const startDrag = (event) => {
  if (!isFullscreen.value) return;

  isDragging.value = true;
  dragStart.value = {
    x: event.clientX - position.value.x,
    y: event.clientY - position.value.y
  };

  document.addEventListener('mousemove', onDrag);
  document.addEventListener('mouseup', stopDrag);
};

const onDrag = (event) => {
  if (!isDragging.value) return;

  position.value = {
    x: event.clientX - dragStart.value.x,
    y: event.clientY - dragStart.value.y
  };
};

const stopDrag = () => {
  isDragging.value = false;
  document.removeEventListener('mousemove', onDrag);
  document.removeEventListener('mouseup', stopDrag);
};

const handleKeyDown = (event) => {
  if (event.key === 'Escape' && isFullscreen.value) {
    toggleFullscreen();
  }
};

const handleWheel = (event) => {
  if (!isFullscreen.value) return;

  // Prevent default scrolling behavior.
  event.preventDefault();

  if (event.deltaY < 0) {
    zoomLevel.value = Math.min(zoomLevel.value + 0.1, 3); // Max zoom 3x.
  } else {
    zoomLevel.value = Math.max(zoomLevel.value - 0.1, 0.5); // Min zoom 0.5x.
  }
};

const toggleFullscreen = () => {
  isFullscreen.value = !isFullscreen.value;

  const container = document.querySelector(`.mermaid-container[data-mermaid-id="${props.id}"]`);
  if (isFullscreen.value) {

    // Add fullscreen class to the body to handle overflow.
    document.body.classList.add('mermaid-fullscreen-mode');

    // Add fullscreen class to this specific container.
    container.classList.add('fullscreen');

    // Reset zoom and position when entering fullscreen.
    resetZoom();

    // Append dom at body element.
    document.body.appendChild(container); 

    // Add event listener for ESC key.
    container.addEventListener('keydown', handleKeyDown);

    // Add event listeners for drag start and mouse wheel zoom.
    if (diagramContent.value) {
      diagramContent.value.addEventListener('mousedown', startDrag);
      diagramContent.value.addEventListener('wheel', handleWheel, { passive: false });
    }
  } else {
    // Remove fullscreen class from the body.
    document.body.classList.remove('mermaid-fullscreen-mode');

    // Remove fullscreen class from this specific container.
    container.classList.remove('fullscreen');

    // Reset zoom and position when exiting fullscreen.
    resetZoom();

    // Remove dom from body element.
    document.body.removeChild(container); 

    // Remove event listener for ESC key.
    container.removeEventListener('keydown', handleKeyDown);

    // Remove event listeners for drag and wheel.
    if (diagramContent.value) {
      diagramContent.value.removeEventListener('mousedown', startDrag);
      diagramContent.value.removeEventListener('wheel', handleWheel);
    }
  }
};

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

<style>
.mermaid-container {
  position: relative;
  width: 100%;
}

.mermaid-content {
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

body.mermaid-fullscreen-mode {
  overflow: hidden;
}

.mermaid-container {
  position: relative;
  width: 100%;
}

.mermaid-container.fullscreen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background-color: rgba(255, 255, 255, 0.95);
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  box-sizing: border-box;
}

html.dark .mermaid-container.fullscreen {
  background-color: rgba(30, 30, 30, 0.95);
}

.mermaid-container.fullscreen .mermaid-content {
  max-width: 90%;
  max-height: 90%;
  overflow: visible;
  transition: transform 0.1s ease-out;
}

.mermaid-container.fullscreen .mermaid-fullscreen-btn {
  opacity: 1;
  top: 16px;
  right: 16px;
  bottom: auto;
}

.mermaid-zoom-controls {
  position: absolute;
  bottom: 16px;
  right: 16px;
  display: flex;
  gap: 8px;
  z-index: 10;
}

.mermaid-zoom-controls button {
  background-color: var(--vp-code-copy-code-bg);
  border: 1px solid var(--vp-code-copy-code-border-color);
  color: var(--vp-code-copy-code-active-text);
  border-radius: 4px;
  padding: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.2s ease-in-out;
}

.mermaid-zoom-controls button:hover {
  background-color: var(--vp-code-copy-code-hover-bg);
  border-color: var(--vp-code-copy-code-hover-border-color);
}
</style>
