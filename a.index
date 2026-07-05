/* =========================================================================
 * 【LocalDream 本机生图辅助系统 - v6.8.3 零干预双击版】
 *
 * 更新日志:
 *   [FIX]  彻底删除楼层图片的长按与 pointerdown 事件拦截，不与酒馆的移动端
 *          手势引擎发生任何冲突，绝对根治按键瘫痪/死锁的恶性 Bug。
 *   [NEW]  消息楼层图片改为 **双击 (连续点击两次)** 唤出操作与预览面板！
 *   [KEEP] 悬浮球 (Fab) 保留了长按弹出扇形重试菜单的完美体验。
 *   [KEEP] 兜底扫描 / 耗时展示 / CSS潜影 等所有新特性一切正常运行。
 * ========================================================================= */

(async function () {
    const hostWindow = window.parent;

    // ==========================================
    // ⚙️ 模块 1：配置与预设存储
    // ==========================================
    const DB_NAME = "LocalDream_DB", STORE_NAME = "images";
    const DEFAULT_CONFIG = {
        resolution: 512, scheduler_base: "dpm_sde", use_karras: true,
        steps: 20, cfg: 7.0, denoise_strength: 60, seed: "",
        negative_prompt: "low quality, bad anatomy, ugly, deformed, distorted, blurry, noisy, artifacts, lowres, watermark",
        theme: "dark", fab_enabled: false, gallery_view: "grid", auto_clear: false, keep_floors: 10,
        default_tab: "config"
    };

    function getConfig() { const conf = localStorage.getItem("LD_Config"); return conf ? { ...DEFAULT_CONFIG, ...JSON.parse(conf) } : DEFAULT_CONFIG; }
    function saveConfig(updates) { const conf = { ...getConfig(), ...updates }; localStorage.setItem("LD_Config", JSON.stringify(conf)); return conf; }

    function getPresets() { const p = localStorage.getItem("LD_Presets"); return p ? JSON.parse(p) : {}; }
    function savePresets(pObj) { localStorage.setItem("LD_Presets", JSON.stringify(pObj)); }

    function hashPrompt(str) { let hash = 0; for (let i = 0; i < str.length; i++) hash = ((hash << 5) - hash) + str.charCodeAt(i) | 0; return 'ld_' + Math.abs(hash).toString(16); }

    // ==========================================
    // 🔌 模块 2：API 在线检测
    // ==========================================
    let isApiOnline = false, isProcessing = false, manualProgressCallback = null;
    setInterval(async () => {
        if (isProcessing) return;
        try {
            const controller = new AbortController(); setTimeout(() => controller.abort(), 2000);
            await fetch('http://127.0.0.1:8081/', { signal: controller.signal }); isApiOnline = true;
        } catch (e) { isApiOnline = false; }
        const dot = hostWindow.document.getElementById('ld-ui-dot'); if (dot) dot.style.background = isApiOnline ? '#22c55e' : '#64748b';
    }, 5000);

    // ==========================================
    // 🗄️ 模块 3：IndexedDB 持久化
    // ==========================================
    const BlobUrlCache = {};
    const dbPromise = new Promise((resolve, reject) => {
        const request = indexedDB.open(DB_NAME);
        request.onupgradeneeded = (e) => { const db = e.target.result; if (!db.objectStoreNames.contains(STORE_NAME)) db.createObjectStore(STORE_NAME, { keyPath: 'hash' }); };
        request.onsuccess = (e) => { const db = e.target.result; if (db.objectStoreNames.contains(STORE_NAME)) resolve(db); else { const upgradeReq = indexedDB.open(DB_NAME, db.version + 1); upgradeReq.onupgradeneeded = (e2) => { const upgradeDb = e2.target.result; if (!upgradeDb.objectStoreNames.contains(STORE_NAME)) upgradeDb.createObjectStore(STORE_NAME, { keyPath: 'hash' }); }; upgradeReq.onsuccess = (e2) => resolve(e2.target.result); upgradeReq.onerror = () => reject(upgradeReq.error); } };
        request.onerror = () => reject(request.error);
    });

    async function saveImageToDB(hash, blob, prompt, config, msgId, actualSeed, genTimeMs) {
        const db = await dbPromise;
        return new Promise((res, rej) => {
            const tx = db.transaction(STORE_NAME, 'readwrite');
            const data = { hash, blob, prompt, config, timestamp: Date.now(), message_id: msgId, seed: actualSeed, generation_time_ms: genTimeMs };
            const req = tx.objectStore(STORE_NAME).put(data);
            req.onsuccess = () => res();
            req.onerror = () => rej(req.error);
        });
    }
    async function getImageFromDB(hash) {
        const db = await dbPromise;
        return new Promise((res, rej) => {
            if (!db.objectStoreNames.contains(STORE_NAME)) return res(null);
            const req = db.transaction(STORE_NAME, 'readonly').objectStore(STORE_NAME).get(hash);
            req.onsuccess = () => res(req.result ? req.result.blob : null);
            req.onerror = () => rej(req.error);
        });
    }
    async function getRecordFromDB(hash) {
        const db = await dbPromise;
        return new Promise((res, rej) => {
            if (!db.objectStoreNames.contains(STORE_NAME)) return res(null);
            const req = db.transaction(STORE_NAME, 'readonly').objectStore(STORE_NAME).get(hash);
            req.onsuccess = () => res(req.result || null);
            req.onerror = () => rej(req.error);
        });
    }
    async function getAllImagesFromDB() {
        const db = await dbPromise;
        return new Promise((res, rej) => {
            if (!db.objectStoreNames.contains(STORE_NAME)) return res([]);
            const req = db.transaction(STORE_NAME, 'readonly').objectStore(STORE_NAME).getAll();
            req.onsuccess = () => res(req.result || []);
            req.onerror = () => rej(req.error);
        });
    }
    async function deleteImageFromDB(hash) {
        const db = await dbPromise;
        return new Promise((res, rej) => {
            const req = db.transaction(STORE_NAME, 'readwrite').objectStore(STORE_NAME).delete(hash);
            req.onsuccess = () => res();
            req.onerror = () => rej(req.error);
        });
    }
    async function getBlobUrl(hash) {
        if (BlobUrlCache[hash]) return BlobUrlCache[hash];
        const blob = await getImageFromDB(hash);
        if (blob) { const url = hostWindow.URL.createObjectURL(blob); BlobUrlCache[hash] = url; return url; }
        return null;
    }

    // ==========================================
    // 🛸 模块 4：状态追踪浮窗层
    // ==========================================
    let trackers = {};
    let zIndexCounter = 20000;

    function getOverlayLayer() {
        const chatContainer = hostWindow.document.getElementById('chat'); if (!chatContainer) return null;
        let layer = hostWindow.document.getElementById('ld-phantom-layer');
        if (!layer) {
            if (hostWindow.getComputedStyle(chatContainer).position === 'static') chatContainer.style.position = 'relative';
            layer = hostWindow.document.createElement('div'); layer.id = 'ld-phantom-layer';
            layer.style.cssText = `position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; z-index: 99999;`;
            chatContainer.appendChild(layer);
        }
        return { layer, chatContainer };
    }

    function fastFindTextRect($container, targetText) {
        if (!$container || $container.length === 0 || !targetText) return null;
        const walker = hostWindow.document.createTreeWalker($container[0], NodeFilter.SHOW_TEXT, null, false); let node;
        while ((node = walker.nextNode())) {
            const idx = node.nodeValue.indexOf(targetText);
            if (idx !== -1) { const range = hostWindow.document.createRange(); range.setStart(node, idx); range.setEnd(node, idx + targetText.length); return range.getBoundingClientRect(); }
        }
        return null;
    }

    function transformFloaterToImage(tracker, imgUrl) {
        tracker.elem.innerHTML = `<img src="${imgUrl}" draggable="false" style="width:100%; border-radius:12px; display:block; object-fit:contain; box-shadow: 0 4px 15px rgba(0,0,0,0.1);" />`;
        tracker.elem.style.padding = '0px'; tracker.elem.style.border = 'none'; tracker.elem.style.background = 'transparent'; tracker.elem.style.boxShadow = 'none'; tracker.elem.style.pointerEvents = 'none';
    }

    function createPhantomFloater(hash, prompt, msgId) {
        const { layer } = getOverlayLayer(); if (!layer) return null;
        const panel = hostWindow.document.createElement('div'); panel.id = `ld-inst-${hash}`;
        panel.style.cssText = `position: absolute !important; width: 90% !important; max-width: 360px !important; background: rgba(128, 128, 128, 0.1) !important; backdrop-filter: blur(2px) !important; border: none !important; z-index: ${zIndexCounter} !important; padding: 10px !important; border-radius: 8px !important; font-family: system-ui, sans-serif !important; display: flex !important; flex-direction: column !important; color: #94a3b8 !important; pointer-events: none !important; opacity: 0; transition: opacity 0.3s ease;`;
        panel.innerHTML = `<div style="font-weight:normal; color:#94a3b8; padding-bottom:6px; font-size:12px; display:flex; align-items:center; gap:6px; opacity:0.8;"><div style="width:8px; height:8px; background:#3b82f6; border-radius:50%; animation: pulse 1s infinite alternate;"></div><span style="font-weight:bold;">图片生成中...</span></div><div style="font-size:12px; line-height:1.4; color:#94a3b8; overflow-y:auto; max-height:80px; flex:1; position:relative; opacity:0.7;">${prompt}</div><style>@keyframes pulse { from { opacity: 0.3; transform:scale(0.8); } to { opacity: 1; transform:scale(1.2); } }</style>`;
        layer.appendChild(panel);
        return { elem: panel, prompt: prompt, msgId: msgId, isTracking: true };
    }

    function updateAllTrackingPositions() {
        const { layer } = getOverlayLayer(); if (!layer) return;
        const layerRect = layer.getBoundingClientRect();
        Object.keys(trackers).forEach(hash => {
            const state = trackers[hash]; if (!state.isTracking) return;
            const $mes = retrieveDisplayedMessage(state.msgId);
            if (!$mes || $mes.length === 0) return;
            const textRect = fastFindTextRect($mes, state.prompt);
            if (textRect && textRect.height > 0) {
                state.elem.style.opacity = '1';
                state.elem.style.top = (textRect.bottom - layerRect.top + 6) + 'px';
                const panelWidth = state.elem.offsetWidth || 300;
                const centerLeft = (layerRect.width - panelWidth) / 2;
                state.elem.style.left = Math.max(0, centerLeft) + 'px';
            } else { state.elem.style.opacity = '0'; }
        });
    }

    // ==========================================
    // 🔐 模块 5：底层渲染干预（CSS 零损耗潜影）
    // ==========================================
    function applyImgStyle($elem) {
        $elem.css({ 'max-width': '100%', 'border-radius': '12px', 'box-shadow': '0 4px 15px rgba(0,0,0,0.1)', 'border': 'none', 'margin-top': '12px', 'object-fit': 'contain', 'display': 'block', 'background': 'transparent', 'padding': '0', 'cursor': 'pointer', '-webkit-touch-callout': 'none' });
    }

    function hydrateDisplayedMessageSync(msgId) {
        const $mes = retrieveDisplayedMessage(msgId);
        if (!$mes || $mes.length === 0) return;
        $mes.find('img[data-ld-hash]').each(function () {
            const hash = $(this).attr('data-ld-hash');
            const currentSrc = $(this).attr('src');
            if (!currentSrc || !currentSrc.startsWith('blob:')) {
                if (BlobUrlCache[hash]) { $(this).attr('src', BlobUrlCache[hash]); applyImgStyle($(this)); }
                else {
                    getBlobUrl(hash).then(url => {
                        if (url && (!$(this).attr('src') || !$(this).attr('src').startsWith('blob:'))) { $(this).attr('src', url); applyImgStyle($(this)); }
                    });
                }
            } else { applyImgStyle($(this)); }
        });
    }

    function runHydrationPulse(msgId) {
        const pulse = () => {
            try {
                if (msgId === 'all') {
                    const lastId = getLastMessageId();
                    const start = Math.max(0, lastId - 40);
                    const msgs = getChatMessages(`${start}-${lastId}`);
                    if (msgs) for (let m of msgs) hydrateDisplayedMessageSync(m.message_id);
                } else { hydrateDisplayedMessageSync(msgId); }
            } catch (e) { }
        };
        pulse(); setTimeout(pulse, 100); setTimeout(pulse, 300); setTimeout(pulse, 800); setTimeout(pulse, 1500);
    }

    async function processAndHydrateMessage(msgId) {
        const msgs = getChatMessages(msgId); if (!msgs || !msgs[0]) return;
        let isChanged = false, newMessage = msgs[0].message;
        const chatVars = getVariables({ type: 'chat' });
        newMessage = newMessage.replace(/<local_img>([\s\S]*?)<\/local_img>/gi, (match, p1) => {
            const prompt = p1.trim(); const hash = hashPrompt(prompt);
            if (chatVars[`LD_STATE_${hash}`] === 'DONE') {
                isChanged = true;
                if (trackers[hash]) { trackers[hash].elem.remove(); delete trackers[hash]; }
                const encPrompt = encodeURIComponent(prompt);
                return `\n\n<img data-ld-hash="${hash}" data-ld-prompt="${encPrompt}" src="" style="display:none;" alt="Generated Image" />\n\n`;
            }
            return match;
        });
        if (isChanged) { try { await setChatMessages([{ message_id: msgId, message: newMessage }]); } catch (e) { } }
        runHydrationPulse(msgId);
    }

    // ==========================================
    // 🧠 模块 6：API 请求与生图管线
    // ==========================================
    const generateQueue = [];

    function rgbBase64ToBlob(base64Raw, w, h) {
        return new Promise((resolve) => {
            const binary = atob(base64Raw);
            const rgbBytes = new Uint8Array(binary.length);
            for (let i = 0; i < binary.length; i++) rgbBytes[i] = binary.charCodeAt(i);
            const canvas = document.createElement('canvas'); canvas.width = w; canvas.height = h;
            const ctx = canvas.getContext('2d', { willReadFrequently: true });
            const imgData = ctx.createImageData(w, h);
            for (let i = 0; i < w * h; i++) { imgData.data[i * 4] = rgbBytes[i * 3]; imgData.data[i * 4 + 1] = rgbBytes[i * 3 + 1]; imgData.data[i * 4 + 2] = rgbBytes[i * 3 + 2]; imgData.data[i * 4 + 3] = 255; }
            ctx.putImageData(imgData, 0, 0);
            canvas.toBlob(blob => resolve(blob), 'image/jpeg', 0.9);
        });
    }

    async function executeGeneration(prompt) {
        const config = getConfig();
        let final_scheduler = config.scheduler_base;
        if (config.use_karras && final_scheduler !== 'lcm') final_scheduler += '_karras';
        const payload = {
            prompt: prompt,
            negative_prompt: config.negative_prompt,
            steps: Number(config.steps),
            cfg: Number(config.cfg),
            width: Number(config.resolution),
            height: Number(config.resolution),
            scheduler: final_scheduler,
            denoise_strength: Number(config.denoise_strength) / 100.0,
            show_diffusion_process: false
        };
        if (config.seed !== "" && !isNaN(Number(config.seed))) Object.assign(payload, { seed: Number(config.seed) });
        const response = await fetch('http://127.0.0.1:8081/generate', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json', 'Accept': 'text/event-stream' },
            body: JSON.stringify(payload)
        });
        if (!response.ok) throw new Error(`API 响应异常: ${response.status}`);
        const reader = response.body.getReader();
        const decoder = new TextDecoder('utf-8');
        let buffer = '', finalImage = null, actualSeedUsed = null, genTimeMs = null;
        while (true) {
            const { done, value } = await reader.read();
            if (done) break;
            buffer += decoder.decode(value, { stream: true });
            const lines = buffer.split('\n'); buffer = lines.pop();
            for (const line of lines) {
                if (line.startsWith('data: ')) {
                    try {
                        const str = line.substring(6).trim();
                        if (!str) continue;
                        const evt = JSON.parse(str);
                        if (evt.type === 'progress' && manualProgressCallback) { manualProgressCallback(evt.step, evt.total_steps, false); }
                        else if (evt.type === 'complete') { finalImage = evt; actualSeedUsed = evt.seed; genTimeMs = evt.generation_time_ms; }
                        else if (evt.type === 'error') throw new Error(evt.message);
                    } catch (e) { }
                }
            }
        }
        if (!finalImage) throw new Error("API 未返回完整数据");
        const blob = await rgbBase64ToBlob(finalImage.image, finalImage.width, finalImage.height);
        return { blob, config, actualSeedUsed, genTimeMs };
    }

    async function processQueue() {
        if (isProcessing || generateQueue.length === 0) return;
        isProcessing = true;
        const task = generateQueue.shift();
        try {
            const result = await executeGeneration(task.prompt);
            await saveImageToDB(task.hash, result.blob, task.prompt, result.config, task.msgId, result.actualSeedUsed, result.genTimeMs);
            const cachedUrl = hostWindow.URL.createObjectURL(result.blob);
            BlobUrlCache[task.hash] = cachedUrl;
            if (task.msgId !== 'Manual') {
                insertOrAssignVariables({ [`LD_STATE_${task.hash}`]: "DONE" }, { type: 'chat' });
                if (builtin.duringGenerating() && msgIdMatchLast(task.msgId)) { if (trackers[task.hash]) transformFloaterToImage(trackers[task.hash], cachedUrl); }
                else { await processAndHydrateMessage(task.msgId); }
            }
        } catch (e) {
            if (task.msgId !== 'Manual') {
                insertOrAssignVariables({ [`LD_STATE_${task.hash}`]: "ERROR" }, { type: 'chat' });
            }
        } finally {
            if (task.msgId === 'Manual' && typeof manualProgressCallback === 'function') manualProgressCallback(100, 100, true);
            setTimeout(() => { isProcessing = false; processQueue(); }, 200);
        }
    }

    function msgIdMatchLast(id) {
        const msgs = getChatMessages(-1);
        if (!msgs || msgs.length === 0) return false;
        return msgs[0].message_id === Number(id);
    }

    function enqueueGeneration(prompt, msgId) {
        let taskHash = hashPrompt(prompt);
        if (msgId === 'Manual') { taskHash = hashPrompt(prompt + '_' + Date.now()); }
        else {
            if (generateQueue.find(t => t.hash === taskHash)) return;
            const currentVars = getVariables({ type: 'chat' });
            if (currentVars && currentVars[`LD_STATE_${taskHash}`]) return;
            insertOrAssignVariables({ [`LD_STATE_${taskHash}`]: "PENDING" }, { type: 'chat' });
        }
        generateQueue.push({ hash: taskHash, prompt, msgId });
        processQueue();
    }

    function scanMessageForPromptsAndEnqueue(msgId) {
        const msgs = getChatMessages(msgId);
        if (!msgs || msgs.length === 0) return;
        const currentMsgId = msgs[0].message_id;
        const content = msgs[0].message;
        let match;
        const scanRegex = /<local_img>([\s\S]*?)<\/local_img>/gi;
        while ((match = scanRegex.exec(content)) !== null) {
            const pt = match[1].trim();
            if (pt.length === 0) continue;
            const hash = hashPrompt(pt);
            enqueueGeneration(pt, currentMsgId);
            const chatVars = getVariables({ type: 'chat' });
            if (!trackers[hash]) {
                const floater = createPhantomFloater(hash, pt, currentMsgId);
                if (chatVars[`LD_STATE_${hash}`] === 'DONE' && BlobUrlCache[hash]) transformFloaterToImage(floater, BlobUrlCache[hash]);
                if (floater) trackers[hash] = floater;
            }
        }
        updateAllTrackingPositions();
    }

    // ==========================================
    // 🎧 模块 7：原生聊天管线监听
    // ==========================================
    eventOn(tavern_events.STREAM_TOKEN_RECEIVED, () => scanMessageForPromptsAndEnqueue(getLastMessageId()));

    eventOn(tavern_events.GENERATION_ENDED, async () => {
        const lastId = getLastMessageId();
        scanMessageForPromptsAndEnqueue(lastId);
        const msgs = getChatMessages(-1);
        if (msgs && msgs[0]) await processAndHydrateMessage(msgs[0].message_id);
        clearOldImages();
    });

    eventOn(tavern_events.CHAT_CHANGED, () => {
        Object.values(trackers).forEach(i => i.elem.remove()); trackers = {};
        closeFanMenu(); closeRetryPanel();
        runHydrationPulse('all');
    });

    eventOn(tavern_events.MESSAGE_UPDATED, (id) => { runHydrationPulse(id); });
    eventOn(tavern_events.MESSAGE_SWIPED, (id) => {
        Object.values(trackers).forEach(i => i.elem.remove()); trackers = {};
        runHydrationPulse(id);
    });

    async function clearOldImages() {
        const conf = getConfig(); if (!conf.auto_clear) return;
        const limit = getLastMessageId() - Number(conf.keep_floors); if (limit < 0) return;
        try {
            const allImgs = await getAllImagesFromDB();
            for (const img of allImgs) {
                if (img.message_id !== 'Manual' && typeof img.message_id === 'number' && img.message_id < limit) {
                    await deleteImageSafely(img.hash);
                }
            }
        } catch (e) { }
    }

    // ==========================================
    // 🛠️ 模块 8：【修改点】图片 “双击” 交互系统
    // ==========================================
    function targetHasWaitingImage(text) { return /<local_img>/i.test(text); }

    async function deleteImageSafely(hash) {
        await deleteImageFromDB(hash);
        deleteVariable(`LD_STATE_${hash}`, { type: 'chat' });
        if (BlobUrlCache[hash]) { hostWindow.URL.revokeObjectURL(BlobUrlCache[hash]); delete BlobUrlCache[hash]; }
    }

    async function executeReDrawAction(hash, msgId, forcedPrompt = null) {
        const targetMsg = getChatMessages(msgId)[0];
        if (!targetMsg) return alert("获取楼层数据失败，无法执行操作。");
        const rawRec = await getRecordFromDB(hash);
        const p = forcedPrompt || (rawRec ? rawRec.prompt : null);
        if (!p) return alert("提示词获取失败！");
        await deleteImageSafely(hash);
        const pureReg = new RegExp(`<img[^>]+data-ld-hash="${hash}"[^>]*>`, 'i');
        const newText = targetMsg.message.replace(pureReg, `<local_img>${p}</local_img>`);
        await setChatMessages([{ message_id: msgId, message: newText }]);
        runHydrationPulse(msgId);
        setTimeout(() => scanMessageForPromptsAndEnqueue(msgId), 50);
        if (uiContainer && uiContainer.style.display !== 'none') uiContainer.style.display = 'none';
    }

    function executeLargePromptModAndReDraw(hash, prompt, msgId, basePanelRemover) {
        const modWrap = hostWindow.document.createElement('div');
        modWrap.style.cssText = `position:fixed; top:0; left:0; width:100vw; height:100vh; background:rgba(0,0,0,0.85); z-index:99999999; display:flex; justify-content:center; align-items:center; padding:20px; box-sizing:border-box; animation:fadein 0.2s; font-family:system-ui;`;
        modWrap.innerHTML = `
            <div style="background:#1e293b; width:100%; max-width:600px; border-radius:20px; padding:24px; box-shadow:0 10px 50px rgba(0,0,0,0.7); display:flex; flex-direction:column; gap:16px; border:1px solid rgba(255,255,255,0.05);">
                <div style="color:#e2e8f0; font-size:18px; font-weight:bold; display:flex; align-items:center; gap:8px;">提示词调整工具</div>
                <textarea id="ld-mod-text" style="width:100%; height:220px; padding:16px; background:#0f172a; color:#e2e8f0; border:1px solid #334155; border-radius:12px; resize:vertical; font-family:inherit; box-sizing:border-box; font-size:14px; line-height:1.5; box-shadow:inset 0 2px 10px rgba(0,0,0,0.2);" placeholder="请输入需要调整的提示词..."></textarea>
                <div style="display:flex; justify-content:flex-end; gap:12px; margin-top:8px;">
                    <button id="ld-mod-cancel" style="background:transparent; color:#94a3b8; border:1px solid #334155; padding:12px 20px; border-radius:10px; cursor:pointer; font-weight:bold; font-size:14px;">取消返回</button>
                    <button id="ld-mod-confirm" style="background:#8b5cf6; color:white; border:none; padding:12px 28px; border-radius:10px; cursor:pointer; font-weight:bold; font-size:14px; box-shadow:0 4px 12px rgba(139,92,246,0.3);">确认修改并重绘</button>
                </div>
            </div>
        `;
        hostWindow.document.body.appendChild(modWrap);
        const ta = modWrap.querySelector('#ld-mod-text'); ta.value = prompt;
        modWrap.querySelector('#ld-mod-cancel').onclick = () => modWrap.remove();
        modWrap.querySelector('#ld-mod-confirm').onclick = () => {
            const np = ta.value;
            if (np && np.trim()) { modWrap.remove(); if (typeof basePanelRemover === 'function') basePanelRemover(); executeReDrawAction(hash, msgId, np.trim()); }
            else { alert("提示词内容不能为空。"); }
        };
    }

    async function showImageOperationPanel(hash, prompt, msgId, isInteractive = true) {
        const existingPanel = hostWindow.document.getElementById('ld-operation-panel');
        if (existingPanel) existingPanel.remove();

        const url = await getBlobUrl(hash); if (!url) return;
        const record = await getRecordFromDB(hash);
        const genTimeMs = record ? record.generation_time_ms : null;
        const genTimeStr = genTimeMs != null ? `⏱ ${(genTimeMs / 1000).toFixed(1)}s` : '';

        const wrap = hostWindow.document.createElement('div');
        wrap.id = 'ld-operation-panel';
        wrap.style.cssText = `position:fixed; top:0; left:0; width:100vw; height:100vh; background:rgba(15,23,42,0.9); z-index:9999999; backdrop-filter:blur(6px); display:flex; flex-direction:column; animation: fadein 0.2s; font-family:system-ui;`;
        wrap.addEventListener('pointerdown', e => e.stopPropagation());

        wrap.innerHTML = `
            <div style="flex:1; display:flex; justify-content:center; align-items:center; padding:16px; overflow:hidden;">
                <img src="${url}" style="max-width:100%; max-height:100%; object-fit:contain; border-radius:12px; box-shadow:0 12px 40px rgba(0,0,0,0.6);" />
            </div>
            <div style="background:var(--ld-card, #1e293b); padding:20px 24px; border-top-left-radius:24px; border-top-right-radius:24px; box-shadow:0 -5px 25px rgba(0,0,0,0.5); display:flex; flex-direction:column; gap:16px; border-top:1px solid rgba(255,255,255,0.05);">
                <div style="display:flex; justify-content:space-between; align-items:center; flex-wrap:wrap; gap:8px;">
                    <div style="color:#e2e8f0; font-size:15px; font-weight:bold;">图片预览与操作</div>
                    <div style="font-size:12px; color:#64748b; font-family:monospace; display:flex; gap:12px; align-items:center;">ID: ${hash.substring(0,8)}${genTimeStr ? `<span style="color:#22c55e;">| ${genTimeStr}</span>` : ''}</div>
                </div>
                <div style="font-size:13px; color:#94a3b8; background:rgba(0,0,0,0.25); padding:12px; border-radius:8px; max-height:80px; overflow-y:auto; word-break:break-all; border:1px inset rgba(255,255,255,0.02);">${prompt}</div>
                <div style="display:flex; flex-direction:column; gap:10px;">
                    <button id="ld-op-copy" style="background:#334155; color:white; border:none; padding:12px; border-radius:10px; cursor:pointer; font-weight:500;">复制提示词</button>
                    ${!isInteractive ? `<div style="text-align:center; color:#64748b; font-size:13px; line-height:38px; background:rgba(255,255,255,0.02); border-radius:10px; border:1px dashed #334155;">历史楼层 (只读模式不可修改)</div>` : `
                    <button id="ld-op-drawsing" style="background:#3b82f6; color:white; border:none; padding:12px; border-radius:10px; cursor:pointer; font-weight:500;">重新生成此图</button>
                    <button id="ld-op-mod" style="background:#8b5cf6; color:white; border:none; padding:12px; border-radius:10px; cursor:pointer; font-weight:500;">修改提示词并重绘</button>
                    `}
                </div>
                <button id="ld-op-close" style="width:100%; background:transparent; border:1px solid #475569; color:#cbd5e1; padding:12px; border-radius:10px; cursor:pointer; font-weight:bold; font-size:14px; margin-top:4px;">关闭控制面板</button>
            </div>
        `;
        hostWindow.document.body.appendChild(wrap);
        const killPanel = () => wrap.remove();
        wrap.querySelector('#ld-op-close').onclick = killPanel;
        wrap.querySelector('#ld-op-copy').onclick = () => { builtin.copyText(prompt); alert("提示词已复制到剪贴板。"); };
        if (isInteractive) {
            wrap.querySelector('#ld-op-drawsing').onclick = () => { killPanel(); executeReDrawAction(hash, msgId); };
            wrap.querySelector('#ld-op-mod').onclick = () => { executeLargePromptModAndReDraw(hash, prompt, msgId, killPanel); };
        }
    }

    // 利用标准的间隔测算来纯净实现 “双击交互” (兼容手机端各种奇葩系统流)
    let ldLastClickTime = 0;
    let ldLastClickTarget = null;
    hostWindow.document.body.addEventListener('click', async (e) => {
        if (!e.target || e.target.tagName !== 'IMG') return;
        const hash = e.target.getAttribute('data-ld-hash');
        if (!hash) return;

        const now = Date.now();
        // 允许 400 毫秒内的两次点击判定为双击
        if (ldLastClickTarget === e.target && (now - ldLastClickTime) < 400) {
            e.preventDefault();
            e.stopPropagation();
            ldLastClickTime = 0;
            ldLastClickTarget = null;

            const mesWrap = e.target.closest('.mes');
            if (!mesWrap) return;
            const msgId = parseInt(mesWrap.getAttribute('mesid'));

            const isLatest = (msgId === getLastMessageId());
            const msgObj = getChatMessages(msgId)[0];
            const isAI = msgObj && msgObj.role === 'assistant';
            const isQueueClear = generateQueue.length === 0;
            const hasUnfinished = targetHasWaitingImage(msgObj?.message || '');
            const isInteractive = isLatest && isAI && isQueueClear && !hasUnfinished;
            let encPrompt = e.target.getAttribute('data-ld-prompt');

            let prompt = null;
            try {
                prompt = encPrompt ? decodeURIComponent(encPrompt) : null;
                if (!prompt) { const r = await getRecordFromDB(hash); prompt = r ? r.prompt : '未能获取到提示词信息。'; }
                showImageOperationPanel(hash, prompt, msgId, isInteractive);
            } catch (err) {
                console.error("加载面板失败:", err);
            }
        } else {
            ldLastClickTime = now;
            ldLastClickTarget = e.target;
        }
    });

    hostWindow.document.body.addEventListener('contextmenu', (e) => {
        if (e.target && e.target.tagName === 'IMG' && e.target.getAttribute('data-ld-hash')) {
            e.preventDefault();
        }
    });

    // ==========================================
    // 🌐 模块 9：悬浮球 & UI 面板 & 扇形菜单
    // ==========================================
    let floatingBall = null, uiContainer = null, isMultiSelect = false, selectedHashes = new Set();
    function syncFabState() { const conf = getConfig(); if (conf.fab_enabled) { if (!floatingBall) createFloatingBall(); floatingBall.style.display = 'flex'; } else { if (floatingBall) floatingBall.style.display = 'none'; } }
    function applyThemeToUI(theme) {
        if (!uiContainer) return;
        if (theme === 'light') { uiContainer.style.setProperty('--ld-bg', '#f1f5f9'); uiContainer.style.setProperty('--ld-card', '#ffffff'); uiContainer.style.setProperty('--ld-border', '#cbd5e1'); uiContainer.style.setProperty('--ld-text', '#0f172a'); uiContainer.style.setProperty('--ld-text-dim', '#64748b'); uiContainer.style.setProperty('--ld-input-bg', '#f8fafc'); }
        else { uiContainer.style.setProperty('--ld-bg', '#0f172a'); uiContainer.style.setProperty('--ld-card', '#1e293b'); uiContainer.style.setProperty('--ld-border', '#334155'); uiContainer.style.setProperty('--ld-text', '#f8fafc'); uiContainer.style.setProperty('--ld-text-dim', '#94a3b8'); uiContainer.style.setProperty('--ld-input-bg', '#0f172a'); }
    }

    async function cleanResidualVars() {
        const allImgs = await getAllImagesFromDB(); const validSet = new Set(allImgs.map(i => i.hash));
        generateQueue.forEach(t => validSet.add(t.hash)); const vars = getVariables({ type: 'chat' }); let cleans = 0;
        for (const key in vars) { if (key.startsWith('LD_STATE_')) { const pureHash = key.replace('LD_STATE_', ''); if (!validSet.has(pureHash)) { deleteVariable(key, { type: 'chat' }); cleans++; } } }
        alert(`系统维护完毕。已成功清理 ${cleans} 个残留状态变量。`);
    }

    function scanLastMessageForFailedImages() {
        const msgs = getChatMessages(-1);
        if (!msgs || !msgs[0]) return [];
        const content = msgs[0].message;
        const regex = /<local_img>([\s\S]*?)<\/local_img>/gi;
        const results = []; let match;
        while ((match = regex.exec(content)) !== null) {
            const prompt = match[1].trim(); const hash = hashPrompt(prompt);
            const chatVars = getVariables({ type: 'chat' });
            const state = chatVars[`LD_STATE_${hash}`];
            results.push({ prompt, hash, state: state || 'NOT_FOUND', canRetry: !state || state === 'ERROR' || state === 'DONE' });
        }
        return results;
    }

    let retryPanel = null;
    function closeRetryPanel() { if (retryPanel) { retryPanel.remove(); retryPanel = null; } }

    function showRetryPanel(failedImages, msgId) {
        closeRetryPanel(); closeFanMenu();
        const doc = hostWindow.document;
        retryPanel = doc.createElement('div'); retryPanel.id = 'ld-retry-panel';
        retryPanel.style.cssText = `position:fixed; top:0; left:0; width:100vw; height:100vh; background:rgba(0,0,0,0.65); z-index:9999998; display:flex; justify-content:center; align-items:center; backdrop-filter:blur(3px); font-family:system-ui; animation:fadein 0.2s;`;

        let bodyHTML = '';
        if (failedImages.length === 0) {
            const msgs = getChatMessages(-1);
            const hasAnyTag = msgs && msgs[0] && /<local_img>/i.test(msgs[0].message);
            if (hasAnyTag) { bodyHTML = `<div style="text-align:center; padding:20px;"><div style="font-size:40px; margin-bottom:12px;">⏳</div><div style="color:#f8fafc; font-size:15px; font-weight:bold; margin-bottom:6px;">所有图片任务均在排队中</div><div style="color:#94a3b8; font-size:12px;">请耐心等待队列处理完成</div></div>`; }
            else { bodyHTML = `<div style="text-align:center; padding:20px;"><div style="font-size:40px; margin-bottom:12px;">✅</div><div style="color:#f8fafc; font-size:15px; font-weight:bold; margin-bottom:6px;">最后楼层所有图片均已生成完毕</div><div style="color:#94a3b8; font-size:12px;">无需重试</div></div>`; }
        } else {
            const retryableCount = failedImages.filter(i => i.canRetry).length;
            bodyHTML = `
                <div style="margin-bottom:16px; display:flex; justify-content:space-between; align-items:center;"><div style="color:#f8fafc; font-size:15px; font-weight:bold;">🔍 检测结果：${failedImages.length} 张未生成</div><span style="color:#94a3b8; font-size:11px;">可重试 ${retryableCount} 张</span></div>
                <div style="max-height:320px; overflow-y:auto; display:flex; flex-direction:column; gap:8px; margin-bottom:16px;">
                    ${failedImages.map((img, idx) => {
                const stateLabels = { 'NOT_FOUND': '⚠️ 未触发', 'ERROR': '❌ 生成失败', 'PENDING': '⏳ 排队中', 'DONE': '✅ 已完成(待渲染)' };
                const stateLabel = stateLabels[img.state] || img.state;
                const promptPreview = img.prompt.length > 60 ? img.prompt.substring(0, 60) + '...' : img.prompt;
                return `<div style="background:rgba(255,255,255,0.04); border:1px solid rgba(255,255,255,0.06); border-radius:10px; padding:12px; display:flex; justify-content:space-between; align-items:center; gap:8px;"><div style="flex:1; min-width:0;"><div style="color:#e2e8f0; font-size:12px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis;">${promptPreview}</div><div style="color:#94a3b8; font-size:11px; margin-top:3px;">状态: ${stateLabel}</div></div>${img.canRetry ? `<button data-retry-idx="${idx}" style="flex-shrink:0; background:#3b82f6; color:white; border:none; padding:8px 14px; border-radius:8px; cursor:pointer; font-weight:bold; font-size:12px; white-space:nowrap;">🔄 重试</button>` : `<span style="flex-shrink:0; color:#64748b; font-size:11px;">排队中…</span>`}</div>`;
            }).join('')}
                </div>
                ${retryableCount > 0 ? `<button id="ld-retry-all" style="width:100%; background:#8b5cf6; color:white; border:none; padding:12px; border-radius:10px; cursor:pointer; font-weight:bold; font-size:14px; margin-bottom:10px;">🔄 全部重试 (${retryableCount} 张)</button>` : ''}
            `;
        }

        retryPanel.innerHTML = `<div style="background:#1e293b; width:90%; max-width:500px; max-height:85vh; border-radius:20px; padding:24px; box-shadow:0 12px 50px rgba(0,0,0,0.7); display:flex; flex-direction:column; border:1px solid rgba(255,255,255,0.05); overflow-y:auto;">${bodyHTML}<button id="ld-retry-close" style="width:100%; background:transparent; border:1px solid #475569; color:#cbd5e1; padding:12px; border-radius:10px; cursor:pointer; font-weight:bold; font-size:14px;">关闭</button></div>`;
        doc.body.appendChild(retryPanel);
        retryPanel.querySelector('#ld-retry-close').onclick = closeRetryPanel;
        retryPanel.querySelectorAll('[data-retry-idx]').forEach(btn => {
            btn.onclick = () => { const idx = parseInt(btn.getAttribute('data-retry-idx')); if (failedImages[idx]) { enqueueGeneration(failedImages[idx].prompt, msgId); btn.disabled = true; btn.style.background = '#64748b'; btn.textContent = '已入队'; } };
        });
        const retryAllBtn = retryPanel.querySelector('#ld-retry-all');
        if (retryAllBtn) {
            retryAllBtn.onclick = () => { let count = 0; failedImages.forEach(img => { if (img.canRetry) { enqueueGeneration(img.prompt, msgId); count++; } }); retryAllBtn.disabled = true; retryAllBtn.textContent = `✅ 已入队 ${count} 张`; retryAllBtn.style.background = '#22c55e'; };
        }
    }

    let fanMenu = null, longPressFired = false;
    function closeFanMenu() { if (fanMenu) { fanMenu.style.opacity = '0'; fanMenu.style.transform = 'scale(0.6)'; fanMenu.style.transition = 'opacity 0.15s ease, transform 0.15s ease'; const menuRef = fanMenu; setTimeout(() => { if (menuRef.parentNode) menuRef.remove(); }, 200); fanMenu = null; } }

    function createFanMenu(ballRect) {
        closeFanMenu();
        const doc = hostWindow.document, FAN_RADIUS = 140, BALL_SIZE = 44, ballCenterX = ballRect.left + BALL_SIZE / 2, ballCenterY = ballRect.top + BALL_SIZE / 2, screenWidth = hostWindow.innerWidth;
        const expandRight = ballCenterX < screenWidth / 2;
        const startAngle = expandRight ? -30 : 90, endAngle = expandRight ? 90 : 210, arcSpan = endAngle - startAngle;

        fanMenu = doc.createElement('div'); fanMenu.id = 'ld-fan-menu';
        fanMenu.style.cssText = `position: fixed !important; left: ${ballCenterX}px !important; top: ${ballCenterY}px !important; width: 0 !important; height: 0 !important; z-index: 9999990 !important; pointer-events: none !important; opacity: 0; transform: scale(0.6); transition: opacity 0.2s ease, transform 0.2s ease;`;

        const bgArc = doc.createElement('div'); bgArc.id = 'ld-fan-bg'; const bgAngle = expandRight ? '-30deg' : '90deg';
        bgArc.style.cssText = `position: absolute !important; left: ${-FAN_RADIUS}px !important; top: ${-FAN_RADIUS}px !important; width: ${FAN_RADIUS * 2}px !important; height: ${FAN_RADIUS * 2}px !important; border-radius: 50% !important; background: conic-gradient(from ${bgAngle}, rgba(30, 41, 59, 0.92) 0deg, rgba(30, 41, 59, 0.92) ${arcSpan}deg, transparent ${arcSpan}deg) !important; border: 1px solid rgba(255,255,255,0.06) !important; pointer-events: none !important;`;
        fanMenu.appendChild(bgArc);

        const btnContainer = doc.createElement('div'); btnContainer.style.cssText = `position: absolute !important; left: 0 !important; top: 0 !important; pointer-events: auto !important;`;
        const buttons = [{ label: '🔍 检测重试', action: 'retry' }];
        buttons.forEach((btn, index) => {
            const btnAngle = startAngle + (arcSpan / (buttons.length + 1)) * (index + 1), rad = btnAngle * Math.PI / 180, distance = FAN_RADIUS * 0.62;
            const btnX = Math.cos(rad) * distance, btnY = Math.sin(rad) * distance;
            const btnEl = doc.createElement('button'); btnEl.textContent = btn.label;
            btnEl.style.cssText = `position: absolute !important; left: ${btnX}px !important; top: ${btnY}px !important; transform: translate(-50%, -50%) !important; background: rgba(59, 130, 246, 0.9) !important; color: white !important; border: 1px solid rgba(255,255,255,0.2) !important; padding: 10px 16px !important; border-radius: 24px !important; cursor: pointer !important; font-weight: bold !important; font-size: 13px !important; white-space: nowrap !important; font-family: system-ui, sans-serif !important; box-shadow: 0 4px 16px rgba(0,0,0,0.3) !important; pointer-events: auto !important; -webkit-tap-highlight-color: transparent !important; transition: background 0.15s !important;`;
            btnEl.addEventListener('pointerdown', (e) => { e.stopPropagation(); });
            btnEl.addEventListener('click', (e) => { e.stopPropagation(); if (btn.action === 'retry') { const lastId = getLastMessageId(); showRetryPanel(scanLastMessageForFailedImages(), lastId); } });
            btnEl.addEventListener('pointerenter', () => { btnEl.style.background = 'rgba(59, 130, 246, 1)'; }); btnEl.addEventListener('pointerleave', () => { btnEl.style.background = 'rgba(59, 130, 246, 0.9)'; });
            btnContainer.appendChild(btnEl);
        });
        fanMenu.appendChild(btnContainer); doc.body.appendChild(fanMenu);
        requestAnimationFrame(() => { fanMenu.style.opacity = '1'; fanMenu.style.transform = 'scale(1)'; });
    }

    hostWindow.document.addEventListener('pointerdown', (e) => {
        if (fanMenu && !fanMenu.contains(e.target) && e.target !== floatingBall) { closeFanMenu(); }
        if (retryPanel && !retryPanel.contains(e.target)) { closeRetryPanel(); }
    }, { capture: true });

    function createFloatingBall() {
        const STORAGE_KEY = 'ld-fab-pos', FAB_SIZE = 44;
        let isDragging = false, hasMoved = false, dragStart = { x: 0, y: 0 }, dragBase = { x: 0, y: 0 }, fabHoldTimer = null;
        function clampPos(x, y) { return { x: Math.max(10, Math.min(x, hostWindow.innerWidth - FAB_SIZE - 10)), y: Math.max(10, Math.min(y, hostWindow.innerHeight - FAB_SIZE - 10)) }; }

        floatingBall = hostWindow.document.createElement('div'); floatingBall.innerHTML = '🎨';
        const raw = hostWindow.localStorage.getItem(STORAGE_KEY), pos = raw ? clampPos(JSON.parse(raw).x, JSON.parse(raw).y) : { x: hostWindow.innerWidth - 60, y: hostWindow.innerHeight - 150 };
        floatingBall.style.cssText = `position: fixed !important; top: ${pos.y}px !important; left: ${pos.x}px !important; width: ${FAB_SIZE}px !important; height: ${FAB_SIZE}px !important; line-height: ${FAB_SIZE}px !important; border-radius: 50% !important; background: rgba(15, 23, 42, 0.88) !important; border: 1px solid rgba(255, 255, 255, 0.1) !important; backdrop-filter: blur(8px) !important; cursor: grab !important; display: flex !important; align-items: center !important; justify-content: center !important; box-shadow: 0 4px 16px rgba(0, 0, 0, 0.3) !important; color: white !important; z-index: 999998 !important; user-select: none !important; touch-action: none !important; font-size: 20px !important; -webkit-tap-highlight-color: transparent !important;`;

        floatingBall.onpointerdown = (e) => {
            if (e.button !== 0) return; e.preventDefault();
            isDragging = false; hasMoved = false; longPressFired = false;
            dragStart = { x: e.clientX, y: e.clientY }; dragBase = { x: floatingBall.getBoundingClientRect().left, y: floatingBall.getBoundingClientRect().top };
            floatingBall.style.cursor = 'grabbing';
            fabHoldTimer = setTimeout(() => { if (!hasMoved && !longPressFired) { longPressFired = true; createFanMenu(floatingBall.getBoundingClientRect()); } fabHoldTimer = null; }, 550);

            const move = (em) => {
                const dx = em.clientX - dragStart.x, dy = em.clientY - dragStart.y;
                if (Math.abs(dx) > 3 || Math.abs(dy) > 3) {
                    hasMoved = true; isDragging = true; if (fabHoldTimer) { clearTimeout(fabHoldTimer); fabHoldTimer = null; }
                    const np = clampPos(dragBase.x + dx, dragBase.y + dy); floatingBall.style.left = np.x + 'px'; floatingBall.style.top = np.y + 'px';
                }
            };
            const up = () => { hostWindow.removeEventListener('pointermove', move); hostWindow.removeEventListener('pointerup', up); floatingBall.style.cursor = 'grab'; if (fabHoldTimer) { clearTimeout(fabHoldTimer); fabHoldTimer = null; } setTimeout(() => hasMoved = false, 100); hostWindow.localStorage.setItem(STORAGE_KEY, JSON.stringify({ x: floatingBall.getBoundingClientRect().left, y: floatingBall.getBoundingClientRect().top })); };
            hostWindow.addEventListener('pointermove', move); hostWindow.addEventListener('pointerup', up);
        };
        floatingBall.onclick = () => { if (!hasMoved && !longPressFired && typeof createUI === 'function') { createUI(); } };
        hostWindow.document.body.appendChild(floatingBall);
    }

    // ==========================================
    // 🌐 模块 10：UI 面板
    // ==========================================
    function createUI() {
        const cConf = getConfig();
        if (uiContainer) {
            uiContainer.style.display = 'flex'; applyThemeToUI(cConf.theme);
            const initTab = cConf.default_tab === 'gallery' ? 'ld-tab-gallery' : 'ld-tab-config';
            uiContainer.querySelectorAll('.ld-tab-btn').forEach(b => { if (b.dataset.target === initTab) b.click(); });
            return;
        }
        const doc = hostWindow.document; uiContainer = doc.createElement('div');
        uiContainer.style.cssText = `position:fixed; top:0; left:0; width:100vw; height:100vh; background:rgba(0,0,0,0.5); z-index:999999; display:flex; justify-content:center; align-items:center; backdrop-filter:blur(3px); font-family: system-ui, sans-serif; font-size: 14px;`;
        uiContainer.innerHTML = `<div style="background:var(--ld-card); border:1px solid var(--ld-border); color:var(--ld-text); border-radius:12px; width:90%; max-width:650px; height:85vh; display:flex; flex-direction:column; box-shadow:0 12px 40px rgba(0,0,0,0.4); transition: 0.3s; overflow:hidden;"><div style="background:var(--ld-bg); padding:16px; display:flex; justify-content:space-between; align-items:center; border-bottom:1px solid var(--ld-border);"><h2 style="margin:0; font-size:16px; display:flex; align-items:center; gap:8px;"><div id="ld-ui-dot" style="width:10px; height:10px; border-radius:50%; background:${isApiOnline ? '#22c55e' : '#64748b'};"></div> LocalDream 面板</h2><button id="ld-ui-close" style="background:transparent; border:none; color:var(--ld-text-dim); cursor:pointer; font-size:24px; line-height:1;">×</button></div><div style="display:flex; border-bottom:1px solid var(--ld-border); background:var(--ld-card);"><button class="ld-tab-btn" data-target="ld-tab-config" style="flex:1; padding:12px; background:transparent; border:none; color:var(--ld-text); font-weight:bold; border-bottom:2px solid transparent;">生成参数设置</button><button class="ld-tab-btn" data-target="ld-tab-gallery" style="flex:1; padding:12px; background:transparent; border:none; color:var(--ld-text-dim); font-weight:bold; border-bottom:2px solid transparent;">画廊图册管理</button></div><div style="flex:1; overflow-y:auto; padding:16px; display:flex; flex-direction:column; gap:16px; background:var(--ld-card);"><div id="ld-tab-config" style="display:none; flex-direction:column; gap:16px;"><div style="background:var(--ld-bg); border:1px solid var(--ld-border); border-radius:8px; padding:12px;"><label style="font-weight:bold; margin-bottom:8px; display:block; color:var(--ld-text);">🔖 生图参数预设</label><div style="display:flex; flex-direction:column; gap:8px;"><select id="ld-preset-sel" style="width:100%; padding:8px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border); border-radius:6px;"></select><div style="display:flex; gap:8px; justify-content:flex-end;"><button id="ld-preset-save" style="background:#3b82f6; color:white; border:none; padding:6px 14px; border-radius:6px; cursor:pointer; font-size:12px; font-weight:bold;">保存当前预设</button><button id="ld-preset-del" style="background:#ef4444; color:white; border:none; padding:6px 14px; border-radius:6px; cursor:pointer; font-size:12px; font-weight:bold;">删除选中预设</button></div></div></div><details style="background:var(--ld-bg); border:1px solid var(--ld-border); border-radius:8px; padding:12px;"><summary style="font-weight:bold; cursor:pointer; color:var(--ld-text); list-style:none;">🟢 手动生图测试 (点击展开)</summary><div style="margin-top:10px;"><textarea id="ld-manual-p" rows="2" placeholder="输入测试的提示词内容..." style="width:100%; box-sizing:border-box; padding:8px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border); border-radius:6px; resize:vertical;"></textarea><div style="margin-top:10px; display:flex; align-items:center;"><div style="flex:1; margin-right:12px; height:6px; background:var(--ld-border); border-radius:3px; overflow:hidden;"><div id="ld-manual-bar" style="width:0%; height:100%; background:#3b82f6; transition: 0.2s;"></div></div><button id="ld-manual-btn" style="background:transparent; border:1px solid #3b82f6; color:#3b82f6; padding:6px 14px; border-radius:4px; font-size:12px; cursor:pointer; font-weight:bold;">生成图片</button></div></div></details><div><label style="display:block; margin-bottom:4px;">分辨率 (Resolution)</label><select id="ld-i-res" style="width:100%; padding:8px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border); border-radius:6px;"><option value="512">512 × 512</option><option value="768">768 × 768</option><option value="1024">1024 × 1024</option></select></div><div><label style="display:block; margin-bottom:4px;">调度器 (Scheduler)</label><div style="display:flex; gap:12px; align-items:center;"><select id="ld-i-sche" style="flex:1; padding:8px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border); border-radius:6px;"><option value="dpm_sde">DPM++ 2M SDE</option><option value="dpm">DPM++ 2M</option><option value="euler_a">Euler A</option><option value="euler">Euler</option><option value="lcm">LCM</option></select><label style="display:flex; align-items:center; gap:6px; cursor:pointer;"><input type="checkbox" id="ld-i-karras"> 使用 Karras</label></div></div><details style="background:var(--ld-bg); border:1px solid var(--ld-border); border-radius:8px; padding:10px;"><summary style="font-weight:bold; cursor:pointer; color:var(--ld-text); list-style:none;">⚙️ 高级参数配置 (点击展开)</summary><div style="margin-top:14px; display:flex; flex-direction:column; gap:12px;"><div><label style="display:flex; justify-content:space-between;"><span>生成步数 (Steps)</span><strong id="ld-v-steps">20</strong></label><input type="range" id="ld-i-steps" min="0" max="50" style="width:100%;"></div><div><label style="display:flex; justify-content:space-between;"><span>相关性 (CFG Scale)</span><strong id="ld-v-cfg">7.0</strong></label><input type="range" id="ld-i-cfg" min="1" max="20" step="0.5" style="width:100%;"></div><div><label style="display:flex; justify-content:space-between;"><span>重绘强度 (Denoising)</span><strong id="ld-v-denoise">60</strong></label><input type="range" id="ld-i-denoise" min="0" max="100" style="width:100%;"></div></div></details><div><label style="display:block; margin-bottom:4px;">随机种子 (Seed)</label><input type="number" id="ld-i-seed" placeholder="留空则使用随机种子" style="width:100%; padding:8px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border); border-radius:6px; box-sizing:border-box;"></div><div><label style="display:block; margin-bottom:4px;">默认反向提示词 (Negative Prompt)</label><textarea id="ld-i-neg" rows="3" style="width:100%; padding:8px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border); border-radius:6px; resize:vertical; box-sizing:border-box;"></textarea></div></div><div id="ld-tab-gallery" style="display:none; flex-direction:column; gap:16px;"><div style="background:var(--ld-bg); border:1px solid var(--ld-border); border-radius:8px; padding:12px;"><label style="font-weight:bold; margin-bottom:10px; display:block; color:var(--ld-text); font-size:13px;">画廊批量管理</label><div style="display:flex; flex-wrap:wrap; gap:6px;"><button id="ld-btn-multiselect" style="flex:1; min-width:60px; padding:4px 8px; border-radius:4px; border:1px solid #64748b; background:transparent; color:#64748b; cursor:pointer; font-size:12px;">批量选择</button><button id="ld-btn-delselect" style="flex:1; min-width:60px; padding:4px 8px; border-radius:4px; border:1px solid #ef4444; background:transparent; color:#ef4444; cursor:pointer; font-size:12px; display:none;">删除选中</button><button id="ld-btn-delall" style="flex:1; min-width:60px; padding:4px 8px; border-radius:4px; border:1px solid #ef4444; background:transparent; color:#ef4444; cursor:pointer; font-size:12px;">清空所有图库</button><button id="ld-btn-cleanvars" style="flex:1; min-width:80px; padding:4px 8px; border-radius:4px; border:1px solid #f59e0b; background:transparent; color:#f59e0b; cursor:pointer; font-size:12px;">清理残余变量</button></div></div><details style="background:var(--ld-bg); border:1px solid var(--ld-border); border-radius:8px; padding:12px;"><summary style="font-weight:bold; cursor:pointer; color:var(--ld-text); font-size:13px;">⚙️ 系统维护与设置</summary><div style="margin-top:12px; display:flex; flex-direction:column; gap:10px;"><div style="display:flex; justify-content:space-between; align-items:center;"><label>默认打开面板</label><select id="ld-s-deftab" style="padding:4px; border-radius:4px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border);"><option value="config">生成参数配置</option><option value="gallery">画廊管理记录</option></select></div><hr style="border:none; border-top:1px dashed var(--ld-border);"><div style="display:flex; justify-content:space-between; align-items:center;"><label>UI 主题风格</label><select id="ld-s-theme" style="padding:4px; border-radius:4px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border);"><option value="dark">深色模式</option><option value="light">浅色模式</option></select></div><div style="display:flex; justify-content:space-between; align-items:center;"><label>启用桌面悬浮快捷球</label><input type="checkbox" id="ld-s-fab" style="width:18px;height:18px;"></div><hr style="border:none; border-top:1px dashed var(--ld-border);"><div style="display:flex; justify-content:space-between; align-items:center;"><label>开启历史长图自动清理</label><input type="checkbox" id="ld-s-autoclear" style="width:18px;height:18px;"></div><div style="display:flex; justify-content:space-between; align-items:center;"><label>保留最新楼层范围</label><input type="number" id="ld-s-keep" min="1" max="100" style="width:80px; padding:4px; border-radius:4px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border);"></div><hr style="border:none; border-top:1px dashed var(--ld-border);"><div style="display:flex; justify-content:space-between; align-items:center;"><label>画廊浏览样式</label><select id="ld-s-view" style="padding:4px; border-radius:4px; background:var(--ld-input-bg); color:var(--ld-text); border:1px solid var(--ld-border);"><option value="grid">网格布局</option><option value="col2">双列布局</option><option value="list">单列宽视</option></select></div><button id="ld-btn-force-render" style="margin-top:6px; background:transparent; color:#8b5cf6; border:1px solid #8b5cf6; padding:6px; border-radius:4px; cursor:pointer; font-size:12px;">执行界面强制图片渲染</button></div></details><div id="ld-gallery-grid" style="display:grid; grid-template-columns:repeat(auto-fill, minmax(130px, 1fr)); gap:12px; padding-bottom:30px;"></div></div></div></div>`;
        doc.body.appendChild(uiContainer); applyThemeToUI(getConfig().theme);

        uiContainer.querySelector('#ld-ui-close').onclick = () => { uiContainer.style.display = 'none'; };
        uiContainer.querySelectorAll('.ld-tab-btn').forEach(t => t.onclick = (e) => { uiContainer.querySelectorAll('.ld-tab-btn').forEach(btn => { btn.style.color = 'var(--ld-text-dim)'; btn.style.borderBottomColor = 'transparent'; }); e.target.style.color = 'var(--ld-text)'; e.target.style.borderBottomColor = '#3b82f6'; uiContainer.querySelector('#ld-tab-config').style.display = 'none'; uiContainer.querySelector('#ld-tab-gallery').style.display = 'none'; uiContainer.querySelector('#' + e.target.dataset.target).style.display = 'flex'; if (e.target.dataset.target === 'ld-tab-gallery') renderGallery(); });
        const initTab = cConf.default_tab === 'gallery' ? 'ld-tab-gallery' : 'ld-tab-config'; uiContainer.querySelector(`[data-target="${initTab}"]`).click();

        const pBtn = uiContainer.querySelector('#ld-manual-btn'), pInput = uiContainer.querySelector('#ld-manual-p'), pBar = uiContainer.querySelector('#ld-manual-bar');
        pBtn.onclick = () => { const val = pInput.value.trim(); if (val) { pBtn.disabled = true; pBtn.innerText = '生成中'; pBar.style.width = '0%'; manualProgressCallback = (step, total, isDone) => { if (isDone) { pBar.style.width = '100%'; setTimeout(() => { pBtn.disabled = false; pBtn.innerText = '生成图片'; pBar.style.width = '0%'; }, 2000); return; } pBar.style.width = Math.min((step / total) * 100, 100) + '%'; }; enqueueGeneration(val, 'Manual'); } };

        const conf = getConfig(), D = { res: uiContainer.querySelector('#ld-i-res'), sche: uiContainer.querySelector('#ld-i-sche'), kar: uiContainer.querySelector('#ld-i-karras'), step: uiContainer.querySelector('#ld-i-steps'), cfg: uiContainer.querySelector('#ld-i-cfg'), den: uiContainer.querySelector('#ld-i-denoise'), seed: uiContainer.querySelector('#ld-i-seed'), neg: uiContainer.querySelector('#ld-i-neg'), vStep: uiContainer.querySelector('#ld-v-steps'), vCfg: uiContainer.querySelector('#ld-v-cfg'), vDen: uiContainer.querySelector('#ld-v-denoise'), theme: uiContainer.querySelector('#ld-s-theme'), fab: uiContainer.querySelector('#ld-s-fab'), ac: uiContainer.querySelector('#ld-s-autoclear'), keep: uiContainer.querySelector('#ld-s-keep'), view: uiContainer.querySelector('#ld-s-view'), dtab: uiContainer.querySelector('#ld-s-deftab') };
        D.res.value = conf.resolution; D.sche.value = conf.scheduler_base; D.kar.checked = conf.use_karras; D.step.value = conf.steps; D.vStep.innerText = conf.steps; D.cfg.value = conf.cfg; D.vCfg.innerText = conf.cfg; D.den.value = conf.denoise_strength; D.vDen.innerText = conf.denoise_strength; D.seed.value = conf.seed || ""; D.neg.value = conf.negative_prompt; D.theme.value = conf.theme; D.fab.checked = conf.fab_enabled; D.ac.checked = conf.auto_clear; D.keep.value = conf.keep_floors; D.view.value = conf.gallery_view; D.dtab.value = conf.default_tab;

        const syncConfig = () => { D.vStep.innerText = D.step.value; D.vCfg.innerText = parseFloat(D.cfg.value).toFixed(1); D.vDen.innerText = D.den.value; const NC = saveConfig({ resolution: Number(D.res.value), scheduler_base: D.sche.value, use_karras: D.kar.checked, steps: Number(D.step.value), cfg: Number(D.cfg.value), denoise_strength: Number(D.den.value), seed: D.seed.value.trim(), negative_prompt: D.neg.value, theme: D.theme.value, fab_enabled: D.fab.checked, auto_clear: D.ac.checked, keep_floors: Number(D.keep.value), gallery_view: D.view.value, default_tab: D.dtab.value }); applyThemeToUI(NC.theme); syncFabState(); renderGallery(false); };
        Object.values(D).forEach(el => { if (el && (el.tagName === 'INPUT' || el.tagName === 'SELECT' || el.tagName === 'TEXTAREA')) { el.addEventListener('input', syncConfig); el.addEventListener('change', syncConfig); } });

        const pSel = uiContainer.querySelector('#ld-preset-sel');
        const renderPresetSel = () => { const presets = getPresets(); pSel.innerHTML = '<option value="">-- 请选择生成参数预设选项 --</option>'; Object.keys(presets).forEach(name => { const opt = doc.createElement('option'); opt.value = name; opt.innerText = name; pSel.appendChild(opt); }); };
        renderPresetSel();
        uiContainer.querySelector('#ld-preset-save').onclick = () => { const name = hostWindow.prompt("请输入参数预设的保存名称:", ""); if (!name || !name.trim()) return; const presets = getPresets(); presets[name.trim()] = { resolution: Number(D.res.value), scheduler_base: D.sche.value, use_karras: D.kar.checked, steps: Number(D.step.value), cfg: Number(D.cfg.value), denoise_strength: Number(D.den.value), negative_prompt: D.neg.value }; savePresets(presets); renderPresetSel(); pSel.value = name.trim(); };
        uiContainer.querySelector('#ld-preset-del').onclick = () => { const name = pSel.value; if (!name) return alert("当前未选中预设，无法执行删除。"); if (confirm(`是否确定彻底删除预设方案 [${name}] ？`)) { const presets = getPresets(); delete presets[name]; savePresets(presets); renderPresetSel(); } };
        pSel.onchange = () => { const name = pSel.value; if (!name) return; const presets = getPresets(); const p = presets[name]; if (p) { if (p.resolution) D.res.value = p.resolution; if (p.scheduler_base) D.sche.value = p.scheduler_base; if (p.use_karras !== undefined) D.kar.checked = p.use_karras; if (p.steps !== undefined) { D.step.value = p.steps; D.vStep.innerText = p.steps; } if (p.cfg !== undefined) { D.cfg.value = p.cfg; D.vCfg.innerText = parseFloat(p.cfg).toFixed(1); } if (p.denoise_strength !== undefined) { D.den.value = p.denoise_strength; D.vDen.innerText = p.denoise_strength; } if (p.negative_prompt !== undefined) D.neg.value = p.negative_prompt; syncConfig(); } };

        uiContainer.querySelector('#ld-btn-cleanvars').onclick = () => cleanResidualVars();
        uiContainer.querySelector('#ld-btn-force-render').onclick = async () => { runHydrationPulse('all'); alert("已强制触发图片渲染流程，请退出面板观察图层。"); };
        uiContainer.querySelector('#ld-btn-delall').onclick = async () => { if (!confirm('警告：此操作不可挽回。确定要清空画廊并且删除底源中所有存储记录吗？')) return; isMultiSelect = false; selectedHashes.clear(); const allImgs = await getAllImagesFromDB(); for (const img of allImgs) { await deleteImageSafely(img.hash); } renderGallery(); };

        const multiBtn = uiContainer.querySelector('#ld-btn-multiselect'), delSelBtn = uiContainer.querySelector('#ld-btn-delselect'), delAllBtn = uiContainer.querySelector('#ld-btn-delall');
        multiBtn.onclick = () => { isMultiSelect = !isMultiSelect; selectedHashes.clear(); if (isMultiSelect) { multiBtn.innerText = '取消多选'; multiBtn.style.background = '#64748b'; multiBtn.style.color = 'white'; delSelBtn.style.display = 'block'; delAllBtn.style.display = 'none'; } else { multiBtn.innerText = '批量选择'; multiBtn.style.background = 'transparent'; multiBtn.style.color = '#64748b'; delSelBtn.style.display = 'none'; delAllBtn.style.display = 'block'; } renderGallery(false); };
        delSelBtn.onclick = async () => { if (selectedHashes.size === 0) return; if (!confirm(`警告：已选中 ${selectedHashes.size} 张图像，确定要抹除记录吗？`)) return; for (const h of selectedHashes) { await deleteImageSafely(h); } selectedHashes.clear(); renderGallery(); };
    }

    async function renderGallery(reloadFetch = true) {
        if (!uiContainer) return; const grid = uiContainer.querySelector('#ld-gallery-grid'), conf = getConfig(), delSelBtn = uiContainer.querySelector('#ld-btn-delselect');
        if (conf.gallery_view === 'col2') grid.style.gridTemplateColumns = '1fr 1fr'; else if (conf.gallery_view === 'list') grid.style.gridTemplateColumns = '1fr'; else grid.style.gridTemplateColumns = 'repeat(auto-fill, minmax(130px, 1fr))';
        if (reloadFetch) { grid.innerHTML = '<div style="grid-column:1/-1; text-align:center; padding:20px; font-size:12px;">正在加载并索引图中...</div>'; try { grid.dataset.images = JSON.stringify(await getAllImagesFromDB()); } catch (e) { grid.dataset.images = '[]'; } }
        const images = JSON.parse(grid.dataset.images || '[]');
        if (images.length === 0) { grid.innerHTML = '<div style="grid-column:1/-1; text-align:center; color:var(--ld-text-dim); font-size:12px;">目前尚未产生任何保存的局部图画记录。</div>'; return; }
        grid.innerHTML = '';
        images.sort((a, b) => b.timestamp - a.timestamp).forEach(record => {
            const url = BlobUrlCache[record.hash] || (record.blob ? hostWindow.URL.createObjectURL(record.blob) : '');
            const card = hostWindow.document.createElement('div'), isSel = selectedHashes.has(record.hash);
            card.style.cssText = `position:relative; border-radius:6px; overflow:hidden; border:1px solid ${isMultiSelect ? (isSel ? '#3b82f6' : 'var(--ld-border)') : 'var(--ld-border)'}; background:var(--ld-bg); display:flex; flex-direction:column; cursor:pointer; transition:0.2s;`;
            card.innerHTML = `<div style="padding:2px; max-height:400px; display:flex; align-items:center; justify-content:center; background:#000; position:relative;"><img src="${url}" style="max-width:100%; max-height:100%; object-fit:contain; border-radius:4px; display:block; opacity:${isSel ? '0.5' : '1'}; transition:0.2s;" />${isMultiSelect ? `<div style="position:absolute; top:4px; left:4px; width:16px; height:16px; border-radius:3px; border:1px solid white; background:${isSel ? '#3b82f6' : 'rgba(0,0,0,0.5)'}; color:white; display:flex; align-items:center; justify-content:center; font-size:10px;">${isSel ? '✓' : ''}</div>` : ''}</div>`;
            card.onclick = () => { if (isMultiSelect) { if (selectedHashes.has(record.hash)) selectedHashes.delete(record.hash); else selectedHashes.add(record.hash); delSelBtn.innerText = `删选中 (${selectedHashes.size})`; renderGallery(false); } else { const isInteractive = (record.message_id !== 'Manual') && msgIdMatchLast(record.message_id); showImageOperationPanel(record.hash, record.prompt, record.message_id || 'Manual', isInteractive); } };
            grid.appendChild(card);
        });
        delSelBtn.innerText = `删选中 (${selectedHashes.size})`;
    }

    // ==========================================
    // 🚀 初始化
    // ==========================================
    appendInexistentScriptButtons([{ name: '🎨 生图', visible: true }]);
    const btnEvent = getButtonEvent('🎨 生图');
    if (btnEvent) eventOn(btnEvent, () => { createUI(); });
    if (hostWindow.document.readyState === 'loading') { hostWindow.document.addEventListener('DOMContentLoaded', syncFabState); } else { syncFabState(); }

})();
