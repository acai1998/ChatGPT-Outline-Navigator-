# ChatGPT Outline Extension

下面是一套可直接落地的 Chrome / Edge 浏览器插件实现，功能与我们刚刚调好的 Tampermonkey 原型一致：

- 默认仅显示右侧刻度
- hover 展开大纲
- 图钉固定展开/取消固定
- 左侧只展示 10 条大纲
- 右侧只展示 10 条刻度，均匀分布
- 当前视口实时高亮最近的大纲项
- 点击大纲或刻度跳转到对应消息
- DeepSeek 风格浅色悬浮卡片

---

## 目录结构

```text
chatgpt-outline-extension/
├─ manifest.json
├─ content.js
├─ styles.css
└─ icons/
   ├─ icon16.png
   ├─ icon48.png
   └─ icon128.png
```

---

## manifest.json

```json
{
  "manifest_version": 3,
  "name": "ChatGPT Outline Navigator",
  "version": "0.1.0",
  "description": "ChatGPT 对话大纲导航插件，支持刻度导航、hover 展开、图钉固定与快速跳转。",
  "content_scripts": [
    {
      "matches": [
        "https://chatgpt.com/*",
        "https://chat.openai.com/*"
      ],
      "js": ["content.js"],
      "css": ["styles.css"],
      "run_at": "document_idle"
    }
  ],
  "host_permissions": [
    "https://chatgpt.com/*",
    "https://chat.openai.com/*"
  ],
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

---

## styles.css

```css
#tm-chatgpt-outline-root {
  position: fixed;
  top: 50%;
  right: 10px;
  transform: translateY(-50%);
  z-index: 99999;
  display: flex;
  align-items: center;
  pointer-events: auto;
  font-family: Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

#tm-chatgpt-outline-root.hidden-by-screen {
  display: none !important;
}

#tm-chatgpt-outline-root .tm-outline-shell {
  position: relative;
  display: flex;
  align-items: center;
  flex-direction: row;
}

#tm-chatgpt-outline-root .tm-outline-panel {
  width: 0;
  opacity: 0;
  transform: translateX(8px);
  transform-origin: right center;
  overflow: hidden;
  transition: width 0.18s ease, opacity 0.16s ease, transform 0.18s ease, margin-right 0.18s ease;
  margin-right: 0;
  pointer-events: none;
}

#tm-chatgpt-outline-root.expanded .tm-outline-panel {
  width: 240px;
  opacity: 1;
  transform: translateX(0);
  margin-right: 8px;
  pointer-events: auto;
}

#tm-chatgpt-outline-root .tm-outline-panel-inner {
  max-height: 68vh;
  background: rgba(250, 250, 250, 0.92);
  border: 1px solid rgba(0, 0, 0, 0.05);
  border-radius: 20px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.06), 0 2px 8px rgba(0, 0, 0, 0.03);
  backdrop-filter: blur(10px);
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

#tm-chatgpt-outline-root .tm-outline-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 10px 6px 12px;
}

#tm-chatgpt-outline-root .tm-outline-title {
  font-size: 12px;
  font-weight: 600;
  color: #9a9a9a;
  letter-spacing: 0.01em;
}

#tm-chatgpt-outline-root .tm-outline-header-right {
  display: flex;
  align-items: center;
  gap: 4px;
}

#tm-chatgpt-outline-root .tm-outline-btn {
  border: none;
  background: transparent;
  color: #a6a6a6;
  cursor: pointer;
  font-size: 12px;
  line-height: 1;
  padding: 4px 6px;
  border-radius: 8px;
  transition: background 0.15s ease, color 0.15s ease;
}

#tm-chatgpt-outline-root .tm-outline-btn:hover {
  background: rgba(0, 0, 0, 0.04);
  color: #666;
}

#tm-chatgpt-outline-root .tm-outline-btn.pinned {
  color: #4c6fff;
  background: rgba(76, 111, 255, 0.08);
}

#tm-chatgpt-outline-root .tm-outline-body {
  overflow-y: auto;
  overflow-x: hidden;
  padding: 2px 4px 6px 6px;
  scrollbar-width: thin;
  scrollbar-color: rgba(180, 180, 180, 0.45) transparent;
}

#tm-chatgpt-outline-root .tm-outline-body::-webkit-scrollbar {
  width: 6px;
}

#tm-chatgpt-outline-root .tm-outline-body::-webkit-scrollbar-track {
  background: transparent;
}

#tm-chatgpt-outline-root .tm-outline-body::-webkit-scrollbar-thumb {
  background: rgba(180, 180, 180, 0.45);
  border-radius: 999px;
}

#tm-chatgpt-outline-root .tm-outline-item {
  position: relative;
  display: flex;
  align-items: center;
  width: 100%;
  text-align: left;
  border: none;
  background: transparent;
  border-radius: 10px;
  padding: 7px 22px 7px 10px;
  cursor: pointer;
  color: #8a8a8a;
  transition: color 0.14s ease, background 0.14s ease;
  margin-bottom: 0;
}

#tm-chatgpt-outline-root .tm-outline-item:hover {
  background: rgba(0, 0, 0, 0.03);
  color: #666;
}

#tm-chatgpt-outline-root .tm-outline-item.active {
  color: #4c6fff;
  background: transparent;
  font-weight: 500;
}

#tm-chatgpt-outline-root .tm-outline-item::after {
  content: "";
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  width: 8px;
  height: 2px;
  border-radius: 999px;
  background: transparent;
  transition: background 0.14s ease, width 0.14s ease;
}

#tm-chatgpt-outline-root .tm-outline-item.active::after {
  width: 12px;
  background: #4c6fff;
}

#tm-chatgpt-outline-root .tm-outline-item-title {
  display: block;
  width: 100%;
  min-width: 0;
  font-size: 13px;
  line-height: 1.2;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

#tm-chatgpt-outline-root .tm-outline-empty {
  padding: 12px 12px;
  font-size: 12px;
  color: #9a9a9a;
  line-height: 1.5;
}

#tm-chatgpt-outline-root .tm-outline-rail {
  position: relative;
  width: 18px;
  min-width: 18px;
  height: 320px;
  max-height: 320px;
  display: block;
  overflow: visible;
}

#tm-chatgpt-outline-root .tm-outline-rail::before {
  content: "";
  position: absolute;
  top: 4px;
  bottom: 4px;
  left: 50%;
  transform: translateX(-50%);
  width: 1px;
  border-radius: 999px;
  background: rgba(190, 190, 190, 0.28);
  pointer-events: none;
}

#tm-chatgpt-outline-root .tm-outline-tick {
  position: absolute;
  left: 50%;
  transform: translateX(-10%);
  width: 8px;
  height: 2px;
  border-radius: 999px;
  background: rgba(160, 160, 160, 0.72);
  transition: width 0.14s ease, height 0.14s ease, background 0.14s ease, transform 0.14s ease;
  cursor: pointer;
}

#tm-chatgpt-outline-root .tm-outline-tick:hover {
  width: 12px;
  background: rgba(120, 120, 120, 0.9);
  transform: translateX(-10%);
}

#tm-chatgpt-outline-root .tm-outline-tick.active {
  width: 12px;
  height: 3px;
  background: #4c6fff;
  transform: translateX(-10%);
}

.tm-chatgpt-outline-target-highlight {
  outline: 2px solid rgba(76, 111, 255, 0.6) !important;
  outline-offset: 6px !important;
  border-radius: 12px !important;
  transition: outline 0.2s ease;
}
```

---

## content.js

```javascript
(() => {
  'use strict';

  const CONFIG = {
    rootId: 'tm-chatgpt-outline-root',
    maxOutlineItems: 10,
    tickMapHeight: 320,
    hoverExpandDelay: 70,
    hoverCollapseDelay: 220,
    refreshDebounceMs: 280,
    outlineTitleMaxLen: 16,
    highlightClass: 'tm-chatgpt-outline-target-highlight',
    onlyUserMessages: true,
    smallScreenWidth: 1100,
    autoHideOnSmallScreen: true,
    clickLockMs: 650,
    activeScrollDelayMs: 180,
    debug: false,
  };

  const state = {
    allItems: [],
    outlineItems: [],
    activeId: null,
    observer: null,
    intersectionObserver: null,
    initialized: false,
    isExpanded: false,
    isPinned: false,
    expandTimer: null,
    collapseTimer: null,
    lastUrl: '',
    clickLockUntil: 0,
    activeScrollTimer: null,
    lastRenderedTickKey: '',
  };

  const logger = {
    info: (...args) => CONFIG.debug && console.log('[Outline]', ...args),
    error: (...args) => console.error('[Outline]', ...args),
  };

  function debounce(fn, wait) {
    let timer = null;
    return function (...args) {
      clearTimeout(timer);
      timer = setTimeout(() => fn.apply(this, args), wait);
    };
  }

  function throttle(fn, wait) {
    let last = 0;
    let timer = null;
    return function (...args) {
      const now = Date.now();
      const remaining = wait - (now - last);
      if (remaining <= 0) {
        clearTimeout(timer);
        timer = null;
        last = now;
        fn.apply(this, args);
      } else if (!timer) {
        timer = setTimeout(() => {
          last = Date.now();
          timer = null;
          fn.apply(this, args);
        }, remaining);
      }
    };
  }

  function cleanText(text) {
    return (text || '').replace(/\s+/g, ' ').trim();
  }

  function truncateText(text, maxLen = CONFIG.outlineTitleMaxLen) {
    if (!text) return '（空消息）';
    return text.length <= maxLen ? text : `${text.slice(0, maxLen)}...`;
  }

  function safeId(prefix = 'msg') {
    return `${prefix}-${Math.random().toString(36).slice(2, 10)}`;
  }

  function isVisible(el) {
    if (!el || !(el instanceof HTMLElement)) return false;
    const rect = el.getBoundingClientRect();
    const style = window.getComputedStyle(el);
    return rect.width > 0 && rect.height > 0 && style.display !== 'none' && style.visibility !== 'hidden';
  }

  function escapeHtml(str) {
    return String(str)
      .replaceAll('&', '&amp;')
      .replaceAll('<', '&lt;')
      .replaceAll('>', '&gt;')
      .replaceAll('"', '&quot;')
      .replaceAll("'", '&#39;');
  }

  function normalizeUrlPath() {
    return `${location.pathname}${location.search}`;
  }

  function isSmallScreen() {
    return window.innerWidth < CONFIG.smallScreenWidth;
  }

  function nowLockedByClick() {
    return Date.now() < state.clickLockUntil;
  }

  function createRoot() {
    let root = document.getElementById(CONFIG.rootId);
    if (root) return root;

    root = document.createElement('div');
    root.id = CONFIG.rootId;
    root.innerHTML = `
      <div class="tm-outline-shell">
        <div class="tm-outline-panel">
          <div class="tm-outline-panel-inner">
            <div class="tm-outline-header">
              <div class="tm-outline-title">对话大纲</div>
              <div class="tm-outline-header-right">
                <button class="tm-outline-btn tm-pin-btn" title="固定展开">📌</button>
              </div>
            </div>
            <div class="tm-outline-body">
              <div class="tm-outline-empty">正在扫描当前对话...</div>
            </div>
          </div>
        </div>
        <div class="tm-outline-rail"></div>
      </div>
    `;

    document.body.appendChild(root);

    root.addEventListener('mouseenter', () => {
      if (!state.isPinned) expandPanel();
    });

    root.addEventListener('mouseleave', () => {
      if (!state.isPinned) collapsePanel();
    });

    const pinBtn = root.querySelector('.tm-pin-btn');
    pinBtn.addEventListener('click', (e) => {
      e.stopPropagation();
      togglePin();
    });

    handleResponsiveMode();
    window.addEventListener(
      'resize',
      debounce(() => {
        handleResponsiveMode();
        refreshOutline('resize');
      }, 140)
    );

    updatePinButton();
    return root;
  }

  function expandPanel() {
    clearTimeout(state.collapseTimer);
    clearTimeout(state.expandTimer);
    state.expandTimer = setTimeout(() => {
      state.isExpanded = true;
      createRoot().classList.add('expanded');
    }, CONFIG.hoverExpandDelay);
  }

  function collapsePanel() {
    clearTimeout(state.expandTimer);
    clearTimeout(state.collapseTimer);
    state.collapseTimer = setTimeout(() => {
      if (state.isPinned) return;
      state.isExpanded = false;
      createRoot().classList.remove('expanded');
    }, CONFIG.hoverCollapseDelay);
  }

  function togglePin() {
    state.isPinned = !state.isPinned;
    updatePinButton();

    const root = createRoot();
    if (state.isPinned) {
      state.isExpanded = true;
      root.classList.add('expanded');
    } else {
      state.isExpanded = false;
      root.classList.remove('expanded');
    }
  }

  function updatePinButton() {
    const root = createRoot();
    const pinBtn = root.querySelector('.tm-pin-btn');
    if (!pinBtn) return;
    pinBtn.classList.toggle('pinned', state.isPinned);
    pinBtn.title = state.isPinned ? '取消固定展开' : '固定展开';
  }

  function handleResponsiveMode() {
    const root = createRoot();
    const hidden = CONFIG.autoHideOnSmallScreen && isSmallScreen();
    root.classList.toggle('hidden-by-screen', hidden);
  }

  function collectCandidateMessageNodes() {
    const result = [];
    const seen = new Set();
    const selectors = [
      '[data-message-author-role]',
      'article',
      'main article',
      'main [data-message-author-role]',
    ];

    for (const selector of selectors) {
      document.querySelectorAll(selector).forEach((node) => {
        if (!(node instanceof HTMLElement)) return;
        if (!isVisible(node)) return;
        const text = cleanText(node.innerText || '');
        if (!text || text.length < 2) return;
        if (seen.has(node)) return;
        seen.add(node);
        result.push(node);
      });
    }

    return result;
  }

  function guessRole(node, fallbackRole = 'user') {
    const own = node.getAttribute?.('data-message-author-role');
    if (own === 'user' || own === 'assistant') return own;

    const parent = node.closest('[data-message-author-role]');
    const parentRole = parent?.getAttribute?.('data-message-author-role');
    if (parentRole === 'user' || parentRole === 'assistant') return parentRole;

    return fallbackRole;
  }

  function buildAllItems() {
    const nodes = collectCandidateMessageNodes();
    const items = [];
    let fallbackRole = 'user';

    for (const node of nodes) {
      const text = cleanText(node.innerText || '');
      if (!text) continue;

      const role = guessRole(node, fallbackRole);
      fallbackRole = role === 'user' ? 'assistant' : 'user';

      if (CONFIG.onlyUserMessages && role !== 'user') continue;

      if (!node.dataset.tmOutlineId) {
        node.dataset.tmOutlineId = safeId(role);
      }

      items.push({
        id: node.dataset.tmOutlineId,
        role,
        title: truncateText(text),
        rawText: text,
        element: node,
      });
    }

    return items;
  }

  function buildFixedOutlineItems(allItems) {
    if (allItems.length <= CONFIG.maxOutlineItems) return [...allItems];

    const result = [];
    const lastIndex = allItems.length - 1;

    for (let i = 0; i < CONFIG.maxOutlineItems; i++) {
      const idx = Math.round((i * lastIndex) / (CONFIG.maxOutlineItems - 1));
      result.push(allItems[idx]);
    }

    return result.filter((item, index, arr) => index === arr.findIndex((x) => x.id === item.id));
  }

  function buildTickItems(outlineItems) {
    const count = outlineItems.length;
    if (!count) return [];

    if (count === 1) {
      return [{ ...outlineItems[0], mapY: Math.round(CONFIG.tickMapHeight / 2) }];
    }

    const topPadding = 8;
    const bottomPadding = 8;
    const usableHeight = CONFIG.tickMapHeight - topPadding - bottomPadding;
    const step = usableHeight / (count - 1);

    return outlineItems.map((item, index) => ({
      ...item,
      mapY: Math.round(topPadding + step * index),
    }));
  }

  function mapActiveIdToNearestOutlineId(activeId, outlineItems, allItems) {
    if (!activeId || !outlineItems.length) return null;
    if (outlineItems.some((item) => item.id === activeId)) return activeId;

    const activeIndex = allItems.findIndex((item) => item.id === activeId);
    if (activeIndex < 0) return outlineItems[0]?.id || null;

    let bestId = outlineItems[0].id;
    let bestDistance = Infinity;

    outlineItems.forEach((item) => {
      const idx = allItems.findIndex((x) => x.id === item.id);
      if (idx < 0) return;
      const distance = Math.abs(idx - activeIndex);
      if (distance < bestDistance) {
        bestDistance = distance;
        bestId = item.id;
      }
    });

    return bestId;
  }

  function renderTicks(force = false) {
    const root = createRoot();
    const rail = root.querySelector('.tm-outline-rail');
    if (!rail) return;

    const tickItems = buildTickItems(state.outlineItems);
    const highlightedId = mapActiveIdToNearestOutlineId(state.activeId, state.outlineItems, state.allItems);

    const tickKey = JSON.stringify({
      ids: tickItems.map((x) => [x.id, x.mapY]),
      active: highlightedId,
    });

    if (!force && tickKey === state.lastRenderedTickKey) return;
    state.lastRenderedTickKey = tickKey;

    rail.innerHTML = '';

    if (!tickItems.length) {
      const empty = document.createElement('div');
      empty.className = 'tm-outline-tick';
      empty.style.top = `${Math.round(CONFIG.tickMapHeight / 2)}px`;
      rail.appendChild(empty);
      return;
    }

    tickItems.forEach((item) => {
      const tick = document.createElement('div');
      tick.className = 'tm-outline-tick';
      if (item.id === highlightedId) tick.classList.add('active');
      tick.title = item.title;
      tick.style.top = `${item.mapY}px`;
      tick.addEventListener('click', (e) => {
        e.stopPropagation();
        jumpToMessage(item.id);
      });
      rail.appendChild(tick);
    });
  }

  function renderPanelItems() {
    const root = createRoot();
    const body = root.querySelector('.tm-outline-body');
    const title = root.querySelector('.tm-outline-title');
    if (!body) return;

    if (title) {
      title.textContent = state.outlineItems.length ? `对话大纲 · ${state.outlineItems.length}` : '对话大纲';
    }

    if (!state.outlineItems.length) {
      body.innerHTML = `<div class="tm-outline-empty">没有可导航的消息</div>`;
      return;
    }

    const highlightedId = mapActiveIdToNearestOutlineId(state.activeId, state.outlineItems, state.allItems);

    body.innerHTML = '';

    state.outlineItems.forEach((item) => {
      const btn = document.createElement('button');
      btn.className = 'tm-outline-item';
      if (item.id === highlightedId) btn.classList.add('active');

      btn.innerHTML = `<div class="tm-outline-item-title">${escapeHtml(item.title)}</div>`;

      btn.addEventListener('click', (e) => {
        e.stopPropagation();
        jumpToMessage(item.id);
      });

      body.appendChild(btn);
    });

    scheduleEnsureActiveVisible();
  }

  function scheduleEnsureActiveVisible() {
    clearTimeout(state.activeScrollTimer);
    state.activeScrollTimer = setTimeout(() => {
      ensureActiveItemVisible();
    }, CONFIG.activeScrollDelayMs);
  }

  function ensureActiveItemVisible() {
    const root = createRoot();
    const body = root.querySelector('.tm-outline-body');
    const active = root.querySelector('.tm-outline-item.active');
    if (!body || !active) return;

    const bodyRect = body.getBoundingClientRect();
    const activeRect = active.getBoundingClientRect();

    const overTop = activeRect.top < bodyRect.top + 12;
    const overBottom = activeRect.bottom > bodyRect.bottom - 12;

    if (overTop || overBottom) {
      active.scrollIntoView({
        block: 'nearest',
        behavior: 'smooth',
      });
    }
  }

  function flashTarget(el) {
    if (!el) return;
    el.classList.add(CONFIG.highlightClass);
    setTimeout(() => el.classList.remove(CONFIG.highlightClass), 1400);
  }

  function jumpToMessage(id) {
    const item = state.allItems.find((x) => x.id === id);
    if (!item?.element) return;

    state.clickLockUntil = Date.now() + CONFIG.clickLockMs;
    state.activeId = id;

    renderTicks(true);
    renderPanelItems();

    item.element.scrollIntoView({
      behavior: 'smooth',
      block: 'center',
      inline: 'nearest',
    });

    flashTarget(item.element);
  }

  function pickBestVisibleId(entriesVisibleMap) {
    let bestId = null;
    let bestScore = -Infinity;

    entriesVisibleMap.forEach((meta, id) => {
      const ratioScore = meta.ratio * 1000;
      const centerPenalty = Math.abs(meta.centerOffset);
      const score = ratioScore - centerPenalty;

      if (score > bestScore) {
        bestScore = score;
        bestId = id;
      }
    });

    return bestId;
  }

  function setupIntersectionObserver() {
    if (state.intersectionObserver) {
      state.intersectionObserver.disconnect();
    }

    const visibleMap = new Map();

    state.intersectionObserver = new IntersectionObserver(
      (entries) => {
        if (nowLockedByClick()) return;

        const viewportCenter = window.innerHeight / 2;

        entries.forEach((entry) => {
          const id = entry.target.dataset.tmOutlineId;
          if (!id) return;

          if (entry.isIntersecting) {
            const rect = entry.boundingClientRect;
            const elementCenter = rect.top + rect.height / 2;
            const centerOffset = elementCenter - viewportCenter;

            visibleMap.set(id, {
              ratio: entry.intersectionRatio,
              centerOffset,
            });
          } else {
            visibleMap.delete(id);
          }
        });

        const bestId = pickBestVisibleId(visibleMap);

        if (bestId && bestId !== state.activeId) {
          state.activeId = bestId;
          renderTicks();
          renderPanelItems();
        }
      },
      {
        root: null,
        rootMargin: '-10% 0px -18% 0px',
        threshold: [0.08, 0.15, 0.25, 0.4, 0.6, 0.8],
      }
    );

    state.allItems.forEach((item) => {
      if (item.element) state.intersectionObserver.observe(item.element);
    });
  }

  function refreshOutline(reason = 'unknown') {
    logger.info('refresh:', reason);

    try {
      const oldActiveId = state.activeId;
      state.allItems = buildAllItems();
      state.outlineItems = buildFixedOutlineItems(state.allItems);

      if (!state.allItems.length) {
        state.activeId = null;
      } else if (oldActiveId && state.allItems.some((x) => x.id === oldActiveId)) {
        state.activeId = oldActiveId;
      } else {
        state.activeId = state.allItems[0].id;
      }

      state.lastRenderedTickKey = '';
      renderTicks(true);
      renderPanelItems();
      setupIntersectionObserver();
      updatePinButton();
    } catch (err) {
      logger.error('refresh failed:', err);
    }
  }

  const scheduleRefresh = debounce((reason) => refreshOutline(reason), CONFIG.refreshDebounceMs);

  function setupMutationObserver() {
    if (state.observer) {
      state.observer.disconnect();
    }

    state.observer = new MutationObserver((mutations) => {
      if (nowLockedByClick()) return;

      for (const mutation of mutations) {
        if (mutation.type === 'childList') {
          if (mutation.addedNodes.length || mutation.removedNodes.length) {
            scheduleRefresh('dom mutation');
            return;
          }
        }
      }
    });

    state.observer.observe(document.body, {
      childList: true,
      subtree: true,
    });
  }

  function setupUrlWatcher() {
    state.lastUrl = normalizeUrlPath();

    setInterval(() => {
      const current = normalizeUrlPath();
      if (current !== state.lastUrl) {
        state.lastUrl = current;
        setTimeout(() => refreshOutline('url changed'), 450);
      }
    }, 800);
  }

  function setupScrollListener() {
    window.addEventListener(
      'scroll',
      throttle(() => {
        if (nowLockedByClick()) return;
        renderTicks(true);
      }, 180),
      { passive: true }
    );
  }

  function init() {
    if (state.initialized) return;
    state.initialized = true;

    createRoot();
    setupMutationObserver();
    setupUrlWatcher();
    setupScrollListener();
    refreshOutline('init');

    setTimeout(() => refreshOutline('post-init 1'), 900);
    setTimeout(() => refreshOutline('post-init 2'), 1800);
    setTimeout(() => refreshOutline('post-init 3'), 3000);
  }

  function waitForReady(retries = 40) {
    if (document.querySelector('main') || retries <= 0) {
      init();
      return;
    }
    setTimeout(() => waitForReady(retries - 1), 500);
  }

  waitForReady();
})();
```

---

## 安装方式

在 Chrome 或 Edge 打开扩展管理页，开启开发者模式，点击“加载已解压的扩展程序”，选择这个目录即可。

---

## 设置页文件

### options.html

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ChatGPT Outline Navigator 设置</title>
    <link rel="stylesheet" href="options.css" />
  </head>
  <body>
    <main class="page">
      <section class="card">
        <header class="card-header">
          <div>
            <h1>ChatGPT Outline Navigator</h1>
            <p>调整大纲条数、显示模式、位置与面板宽度。</p>
          </div>
          <button id="resetBtn" class="btn btn-secondary">恢复默认</button>
        </header>

        <form id="settingsForm" class="form-grid">
          <label class="field">
            <span class="label">大纲条数</span>
            <input id="maxOutlineItems" type="number" min="5" max="20" step="1" />
            <small>默认 10，建议范围 8 - 15。</small>
          </label>

          <label class="field field-checkbox">
            <span>
              <span class="label">仅显示用户消息</span>
              <small>开启后，只把用户提问作为大纲锚点。</small>
            </span>
            <input id="onlyUserMessages" type="checkbox" />
          </label>

          <label class="field">
            <span class="label">右侧偏移</span>
            <input id="panelRight" type="number" min="0" max="48" step="1" />
            <small>控制组件距离右边缘的像素值。</small>
          </label>

          <label class="field">
            <span class="label">面板宽度</span>
            <input id="expandedWidth" type="number" min="200" max="360" step="10" />
            <small>展开后的大纲面板宽度。</small>
          </label>

          <label class="field">
            <span class="label">刻度地图高度</span>
            <input id="tickMapHeight" type="number" min="220" max="500" step="10" />
            <small>右侧刻度区总高度。</small>
          </label>

          <label class="field field-checkbox">
            <span>
              <span class="label">小屏自动隐藏</span>
              <small>窗口过窄时自动隐藏，避免遮挡内容。</small>
            </span>
            <input id="autoHideOnSmallScreen" type="checkbox" />
          </label>
        </form>

        <footer class="card-footer">
          <span id="statusText" class="status">尚未保存</span>
          <div class="actions">
            <button id="saveBtn" class="btn btn-primary">保存设置</button>
          </div>
        </footer>
      </section>
    </main>

    <script src="options.js"></script>
  </body>
</html>
```

### options.css

```css
:root {
  color-scheme: light;
  --bg: #f5f7fb;
  --card: rgba(255, 255, 255, 0.92);
  --border: rgba(15, 23, 42, 0.08);
  --text: #0f172a;
  --muted: #64748b;
  --primary: #4c6fff;
  --primary-soft: rgba(76, 111, 255, 0.12);
  --shadow: 0 18px 40px rgba(15, 23, 42, 0.08), 0 4px 12px rgba(15, 23, 42, 0.05);
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  background: linear-gradient(180deg, #f8fafc 0%, var(--bg) 100%);
  color: var(--text);
}

.page {
  min-height: 100vh;
  padding: 32px 20px;
}

.card {
  max-width: 860px;
  margin: 0 auto;
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 24px;
  box-shadow: var(--shadow);
  padding: 24px;
  backdrop-filter: blur(10px);
}

.card-header,
.card-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

.card-header h1 {
  margin: 0 0 6px;
  font-size: 24px;
}

.card-header p {
  margin: 0;
  color: var(--muted);
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
  margin: 24px 0;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 16px;
  border: 1px solid var(--border);
  border-radius: 18px;
  background: rgba(255, 255, 255, 0.7);
}

.field-checkbox {
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
}

.label {
  font-weight: 600;
}

small,
.status {
  color: var(--muted);
}

input[type="number"] {
  width: 100%;
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 10px 12px;
  font-size: 14px;
  outline: none;
}

input[type="number"]:focus {
  border-color: rgba(76, 111, 255, 0.35);
  box-shadow: 0 0 0 4px var(--primary-soft);
}

input[type="checkbox"] {
  width: 18px;
  height: 18px;
}

.btn {
  border: none;
  border-radius: 12px;
  padding: 10px 14px;
  font-size: 14px;
  cursor: pointer;
  transition: transform 0.16s ease, box-shadow 0.16s ease, background 0.16s ease;
}

.btn:hover {
  transform: translateY(-1px);
}

.btn-primary {
  background: var(--primary);
  color: #fff;
  box-shadow: 0 10px 24px rgba(76, 111, 255, 0.22);
}

.btn-secondary {
  background: #eef2ff;
  color: #334155;
}

.actions {
  display: flex;
  gap: 10px;
}

@media (max-width: 720px) {
  .form-grid {
    grid-template-columns: 1fr;
  }

  .card-header,
  .card-footer {
    flex-direction: column;
    align-items: flex-start;
  }
}
```

### options.js

```javascript
const DEFAULT_SETTINGS = {
  maxOutlineItems: 10,
  onlyUserMessages: true,
  panelRight: 10,
  expandedWidth: 240,
  tickMapHeight: 320,
  autoHideOnSmallScreen: true,
};

const STORAGE_KEY = 'chatgpt_outline_settings';

const form = document.getElementById('settingsForm');
const saveBtn = document.getElementById('saveBtn');
const resetBtn = document.getElementById('resetBtn');
const statusText = document.getElementById('statusText');

const fields = {
  maxOutlineItems: document.getElementById('maxOutlineItems'),
  onlyUserMessages: document.getElementById('onlyUserMessages'),
  panelRight: document.getElementById('panelRight'),
  expandedWidth: document.getElementById('expandedWidth'),
  tickMapHeight: document.getElementById('tickMapHeight'),
  autoHideOnSmallScreen: document.getElementById('autoHideOnSmallScreen'),
};

function setStatus(text) {
  statusText.textContent = text;
}

function clampNumber(value, min, max, fallback) {
  const num = Number(value);
  if (Number.isNaN(num)) return fallback;
  return Math.min(max, Math.max(min, num));
}

function readForm() {
  return {
    maxOutlineItems: clampNumber(fields.maxOutlineItems.value, 5, 20, DEFAULT_SETTINGS.maxOutlineItems),
    onlyUserMessages: fields.onlyUserMessages.checked,
    panelRight: clampNumber(fields.panelRight.value, 0, 48, DEFAULT_SETTINGS.panelRight),
    expandedWidth: clampNumber(fields.expandedWidth.value, 200, 360, DEFAULT_SETTINGS.expandedWidth),
    tickMapHeight: clampNumber(fields.tickMapHeight.value, 220, 500, DEFAULT_SETTINGS.tickMapHeight),
    autoHideOnSmallScreen: fields.autoHideOnSmallScreen.checked,
  };
}

function writeForm(settings) {
  fields.maxOutlineItems.value = settings.maxOutlineItems;
  fields.onlyUserMessages.checked = settings.onlyUserMessages;
  fields.panelRight.value = settings.panelRight;
  fields.expandedWidth.value = settings.expandedWidth;
  fields.tickMapHeight.value = settings.tickMapHeight;
  fields.autoHideOnSmallScreen.checked = settings.autoHideOnSmallScreen;
}

async function loadSettings() {
  const result = await chrome.storage.sync.get(STORAGE_KEY);
  const settings = { ...DEFAULT_SETTINGS, ...(result[STORAGE_KEY] || {}) };
  writeForm(settings);
  setStatus('已加载当前设置');
}

async function saveSettings() {
  const settings = readForm();
  await chrome.storage.sync.set({ [STORAGE_KEY]: settings });
  setStatus('保存成功，刷新 ChatGPT 页面后生效');
}

async function resetSettings() {
  writeForm(DEFAULT_SETTINGS);
  await chrome.storage.sync.set({ [STORAGE_KEY]: DEFAULT_SETTINGS });
  setStatus('已恢复默认设置');
}

saveBtn.addEventListener('click', async (event) => {
  event.preventDefault();
  await saveSettings();
});

resetBtn.addEventListener('click', async (event) => {
  event.preventDefault();
  await resetSettings();
});

form.addEventListener('input', () => {
  setStatus('设置已修改，记得保存');
});

loadSettings();
```

---

## manifest.json 补充项

把 `manifest.json` 改成下面这版，加入设置页和存储权限：

```json
{
  "manifest_version": 3,
  "name": "ChatGPT Outline Navigator",
  "version": "0.2.0",
  "description": "ChatGPT 对话大纲导航插件，支持刻度导航、hover 展开、图钉固定与快速跳转。",
  "permissions": ["storage"],
  "options_page": "options.html",
  "content_scripts": [
    {
      "matches": [
        "https://chatgpt.com/*",
        "https://chat.openai.com/*"
      ],
      "js": ["content.js"],
      "css": ["styles.css"],
      "run_at": "document_idle"
    }
  ],
  "host_permissions": [
    "https://chatgpt.com/*",
    "https://chat.openai.com/*"
  ],
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

---

## content.js 里读取设置

在 `content.js` 最前面补一个默认设置和读取逻辑，并把常量 `CONFIG` 改成可更新对象。

### 替换 `CONFIG` 定义为

```javascript
const DEFAULT_CONFIG = {
  rootId: 'tm-chatgpt-outline-root',
  panelRight: 10,
  expandedWidth: 240,
  collapsedWidth: 18,
  maxOutlineItems: 10,
  tickMapHeight: 320,
  hoverExpandDelay: 70,
  hoverCollapseDelay: 220,
  refreshDebounceMs: 280,
  outlineTitleMaxLen: 16,
  highlightClass: 'tm-chatgpt-outline-target-highlight',
  debug: false,
  onlyUserMessages: true,
  smallScreenWidth: 1100,
  autoHideOnSmallScreen: true,
  panelMaxHeight: '68vh',
  clickLockMs: 650,
  activeScrollDelayMs: 180,
};

const CONFIG = { ...DEFAULT_CONFIG };
const STORAGE_KEY = 'chatgpt_outline_settings';
```

### 再补一个方法

```javascript
async function loadUserSettings() {
  try {
    const result = await chrome.storage.sync.get(STORAGE_KEY);
    const settings = result[STORAGE_KEY] || {};
    Object.assign(CONFIG, settings);
  } catch (error) {
    console.error('[Outline] load settings failed:', error);
  }
}
```

### 然后把初始化入口改成

```javascript
async function init() {
  if (state.initialized) return;
  state.initialized = true;

  await loadUserSettings();
  createRoot();
  setupMutationObserver();
  setupUrlWatcher();
  setupScrollListener();
  refreshOutline('init');

  setTimeout(() => refreshOutline('post-init 1'), 900);
  setTimeout(() => refreshOutline('post-init 2'), 1800);
  setTimeout(() => refreshOutline('post-init 3'), 3000);
}

function waitForReady(retries = 40) {
  if (document.querySelector('main') || retries <= 0) {
    init();
    return;
  }
  setTimeout(() => waitForReady(retries - 1), 500);
}
```

---

## 图标资源

### 目录

```text
icons/
├─ icon16.png
├─ icon48.png
├─ icon128.png
└─ icon.svg
```

### icon.svg

先放一个矢量源文件，之后导出成 png：

```xml
<svg width="128" height="128" viewBox="0 0 128 128" fill="none" xmlns="http://www.w3.org/2000/svg">
  <rect x="14" y="10" width="78" height="108" rx="20" fill="#F8FAFC" stroke="#E2E8F0" stroke-width="2"/>
  <rect x="28" y="30" width="38" height="6" rx="3" fill="#94A3B8"/>
  <rect x="28" y="48" width="48" height="6" rx="3" fill="#94A3B8"/>
  <rect x="28" y="66" width="42" height="6" rx="3" fill="#94A3B8"/>
  <rect x="28" y="84" width="52" height="6" rx="3" fill="#4C6FFF"/>
  <rect x="99" y="22" width="2" height="84" rx="1" fill="#CBD5E1"/>
  <rect x="98" y="32" width="12" height="4" rx="2" fill="#94A3B8"/>
  <rect x="98" y="52" width="12" height="4" rx="2" fill="#94A3B8"/>
  <rect x="98" y="72" width="12" height="4" rx="2" fill="#4C6FFF"/>
  <rect x="98" y="92" width="12" height="4" rx="2" fill="#94A3B8"/>
</svg>
```

### 生成 png 的最简单办法

1. 用 Figma、Sketch、Photoshop 或浏览器打开 `icon.svg`
2. 分别导出成：
   - `icon16.png`
   - `icon48.png`
   - `icon128.png`
3. 放进 `icons/` 目录

---

## 最终目录结构

```text
chatgpt-outline-extension/
├─ manifest.json
├─ content.js
├─ styles.css
├─ options.html
├─ options.css
├─ options.js
└─ icons/
   ├─ icon16.png
   ├─ icon48.png
   ├─ icon128.png
   └─ icon.svg
```

---

## 下一步最值得补的两项

现在已经够作为一个完整插件跑起来了。下一步最值得做的是：

1. 把标题提取从“截前 16 个字符”升级成“去噪摘要”
2. 增加开关项：是否默认固定展开、是否显示刻度轨道线

