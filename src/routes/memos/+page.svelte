<script>
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabase";

  // -----------------------------------------------
  // localStorageのキー定数
  // -----------------------------------------------
  const STORAGE_KEY = "memo-open-tabs";
  const ACTIVE_TAB_KEY = "memo-active-tab"; // 最後にアクティブだったタブのID

  // -----------------------------------------------
  // タブ管理
  // -----------------------------------------------
  // 開いているタブのリスト { id, title }
  let openTabs = $state([]);

  // アクティブなタブのID
  let activeTabId = $state("");

  // -----------------------------------------------
  // メモキャッシュ
  // { [id]: { title: string, content: string } }
  // -----------------------------------------------
  let memoCache = $state({});

  // -----------------------------------------------
  // エディタの状態管理
  // -----------------------------------------------
  // 現在表示中のタイトル
  let title = $state("");

  // 保存状態の表示
  let saveStatus = $state("");

  // エディタのDOM要素への参照
  let editorEl = $state(null);

  // 自動保存用タイマー
  let saveTimer = null;

  // -----------------------------------------------
  // カラーパレットの定義
  // -----------------------------------------------
  const colors = [
    { label: "黒", value: "#000000" },
    { label: "赤", value: "#e53935" },
    { label: "オレンジ", value: "#fb8c00" },
    { label: "黄色", value: "#fdd835" },
    { label: "緑", value: "#43a047" },
    { label: "青", value: "#1e88e5" },
    { label: "ピンク", value: "#e91e63" },
    { label: "紫", value: "#8e24aa" },
    { label: "グレー", value: "#757575" },
  ];

  // -----------------------------------------------
  // フォントサイズの状態管理（デフォルト18px）
  // -----------------------------------------------
  let fontSize = $state(18);

  // テキスト選択範囲を保存する変数
  let savedRange = null;
  let activeSpan = null;

  // -----------------------------------------------
  // localStorageにタブリストを保存する
  // -----------------------------------------------
  function saveTabsToStorage(tabs) {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(tabs));
  }

  // -----------------------------------------------
  // localStorageからタブリストを読み込む
  // -----------------------------------------------
  function loadTabsFromStorage() {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return [];
    try {
      return JSON.parse(raw);
    } catch {
      return [];
    }
  }

  // -----------------------------------------------
  // Supabaseから1件のメモを取得してキャッシュに保存する
  // -----------------------------------------------
  async function fetchAndCacheMemo(id) {
    // すでにキャッシュ済みならスキップ
    if (memoCache[id]) return;

    const { data, error } = await supabase.from("memos").select("title, content").eq("id", id).single();

    if (error) {
      console.error(`メモの取得に失敗しました id=${id}`, error);
      return;
    }

    // キャッシュに保存する
    memoCache = {
      ...memoCache,
      [id]: { title: data.title ?? "", content: data.content ?? "" },
    };

    // タブのタイトルも最新に更新する
    openTabs = openTabs.map((t) => (t.id === id ? { ...t, title: data.title ?? "無題" } : t));
  }

  // -----------------------------------------------
  // Supabaseに新規メモを作成してそのIDを返す
  // -----------------------------------------------
  async function createNewMemo() {
    const {
      data: { session },
    } = await supabase.auth.getSession();

    const { data, error } = await supabase.from("memos").insert({ title: "無題", content: "", user_id: session.user.id }).select("id").single();

    if (error) {
      console.error("新規メモの作成に失敗しました", error);
      return null;
    }
    return data.id;
  }

  // -----------------------------------------------
  // エディタにアクティブタブのメモを表示する
  // -----------------------------------------------
  function loadEditorFromCache(id) {
    const cached = memoCache[id];
    if (!cached) return;

    // タイトルをセット
    title = cached.title;

    // エディタ本文をセット
    if (editorEl) {
      editorEl.innerHTML = cached.content;
    }
  }

  // -----------------------------------------------
  // タブをクリックしたときの処理
  // -----------------------------------------------
  function switchTab(id) {
    // 切り替え前に現在の内容をキャッシュに保存する
    if (activeTabId && editorEl) {
      memoCache = {
        ...memoCache,
        [activeTabId]: {
          ...memoCache[activeTabId],
          title,
          content: editorEl.innerHTML,
        },
      };
    }

    // アクティブタブを切り替える
    activeTabId = id;

    // 最後にアクティブだったタブのIDをlocalStorageに保存する
    localStorage.setItem(ACTIVE_TAB_KEY, id);

    // キャッシュからエディタに表示する
    loadEditorFromCache(id);
  }

  // -----------------------------------------------
  // ＋ボタン：新規メモを作成してタブに追加する
  // -----------------------------------------------
  async function addTab() {
    const newId = await createNewMemo();
    if (!newId) return;

    // キャッシュに空のメモを登録する
    memoCache = {
      ...memoCache,
      [newId]: { title: "無題", content: "" },
    };

    // タブリストに追加する
    openTabs = [...openTabs, { id: newId, title: "無題" }];
    saveTabsToStorage(openTabs);

    // 新しいタブに切り替える
    switchTab(newId);
  }

  // -----------------------------------------------
  // ×ボタン：タブを閉じる
  // -----------------------------------------------
  async function closeTab(id) {
    const index = openTabs.findIndex((t) => t.id === id);

    // タブリストから削除する
    openTabs = openTabs.filter((t) => t.id !== id);
    saveTabsToStorage(openTabs);

    // キャッシュからも削除する
    const { [id]: _, ...rest } = memoCache;
    memoCache = rest;

    // 閉じたのがアクティブタブだった場合は別のタブに切り替える
    if (id === activeTabId) {
      if (openTabs.length === 0) {
        // タブが0件になったら新規メモを作成する
        await addTab();
      } else {
        // 閉じたタブの1つ前（なければ先頭）に切り替える
        const nextTab = openTabs[Math.max(0, index - 1)];
        switchTab(nextTab.id);
      }
    }
  }

  // -----------------------------------------------
  // ページ表示時の初期化処理
  // -----------------------------------------------
  onMount(async () => {
    // エディタ内の選択範囲が変わるたびに保存する
    document.addEventListener("selectionchange", () => {
      const sel = window.getSelection();
      if (!sel || sel.rangeCount === 0) return;
      const range = sel.getRangeAt(0);
      if (editorEl && editorEl.contains(range.commonAncestorContainer)) {
        const selectedText = range.toString();
        if (savedRange && savedRange.toString() !== selectedText) {
          activeSpan = null;
        }
        savedRange = range.cloneRange();
      }
    });

    // 認証チェック
    const {
      data: { session },
    } = await supabase.auth.getSession();
    if (!session) {
      window.location.href = "/login";
      return;
    }

    // localStorageからタブリストを復元する
    let savedTabs = loadTabsFromStorage();

    // タブが0件なら新規メモを作成する
    if (savedTabs.length === 0) {
      const newId = await createNewMemo();
      if (!newId) return;
      savedTabs = [{ id: newId, title: "無題" }];
    }

    openTabs = savedTabs;
    saveTabsToStorage(openTabs);

    // 最後にアクティブだったタブのIDをlocalStorageから復元する
    const savedActiveId = localStorage.getItem(ACTIVE_TAB_KEY);

    // localStorageの値は文字列なので数値に変換して比較する
    const savedActiveIdNum = savedActiveId ? Number(savedActiveId) : null;
    const initialId = savedActiveIdNum && openTabs.some((t) => t.id === savedActiveIdNum) ? savedActiveIdNum : openTabs[0].id;

    activeTabId = initialId;

    // アクティブタブのメモを先に取得してエディタに表示する
    await fetchAndCacheMemo(activeTabId);
    loadEditorFromCache(activeTabId);

    // バックグラウンドで残りのタブのメモを順次取得する
    for (const tab of openTabs.filter((t) => t.id !== activeTabId)) {
      fetchAndCacheMemo(tab.id);
    }
  });

  // -----------------------------------------------
  // 太字を適用する
  // -----------------------------------------------
  function applyBold() {
    if (!savedRange) return;
    const sel = window.getSelection();
    sel.removeAllRanges();
    sel.addRange(savedRange);
    document.execCommand("bold");
    scheduleAutoSave();
  }

  // -----------------------------------------------
  // 文字色を適用する
  // -----------------------------------------------
  function applyColor(colorValue) {
    if (!savedRange) return;
    const sel = window.getSelection();
    sel.removeAllRanges();
    sel.addRange(savedRange);
    document.execCommand("foreColor", false, colorValue);
    scheduleAutoSave();
  }

  // -----------------------------------------------
  // フォントサイズを適用する
  // -----------------------------------------------
  function applyFontSize(newSize) {
    fontSize = Math.min(72, Math.max(8, newSize));

    if (activeSpan && editorEl.contains(activeSpan)) {
      activeSpan.style.fontSize = fontSize + "px";
      scheduleAutoSave();
      return;
    }

    if (!savedRange || savedRange.collapsed) return;

    const sel = window.getSelection();
    sel.removeAllRanges();
    sel.addRange(savedRange);
    const range = sel.getRangeAt(0);
    const span = document.createElement("span");
    span.style.fontSize = fontSize + "px";
    try {
      range.surroundContents(span);
      activeSpan = span;
    } catch (e) {
      // 複数ノードにまたがる選択の場合はスキップ
    }

    scheduleAutoSave();
  }

  // -----------------------------------------------
  // 入力が止まったら1.5秒後に自動保存する
  // -----------------------------------------------
  function scheduleAutoSave() {
    clearTimeout(saveTimer);
    saveTimer = setTimeout(() => {
      saveMemo();
    }, 1500);
  }

  // -----------------------------------------------
  // Supabaseにメモを保存する
  // -----------------------------------------------
  async function saveMemo() {
    if (!activeTabId) return;

    saveStatus = "保存中...";

    // 現在の内容をキャッシュにも反映する
    memoCache = {
      ...memoCache,
      [activeTabId]: {
        title,
        content: editorEl ? editorEl.innerHTML : "",
      },
    };

    // タブのタイトルも更新する
    openTabs = openTabs.map((t) => (t.id === activeTabId ? { ...t, title: title || "無題" } : t));
    saveTabsToStorage(openTabs);

    const { error } = await supabase
      .from("memos")
      .update({
        title,
        content: editorEl ? editorEl.innerHTML : "",
        updated_at: new Date().toISOString(),
      })
      .eq("id", activeTabId);

    if (error) {
      console.error("保存に失敗しました", error);
      saveStatus = "保存失敗";
    } else {
      saveStatus = "保存しました";
      setTimeout(() => {
        saveStatus = "";
      }, 2000);
    }
  }
</script>

<!-- タブバー -->
<div class="tab-bar">
  {#each openTabs as tab}
    <button class="tab" class:active={tab.id === activeTabId} onclick={() => switchTab(tab.id)}>
      <span class="tab-title">{tab.title || "無題"}</span>
      <span
        class="tab-close"
        role="button"
        tabindex="0"
        aria-label="タブを閉じる"
        onclick={(e) => {
          e.stopPropagation();
          closeTab(tab.id);
        }}
        onkeydown={(e) => {
          if (e.key === "Enter") {
            e.stopPropagation();
            closeTab(tab.id);
          }
        }}>×</span
      >
    </button>
  {/each}

  <!-- ＋ボタン -->
  <button class="tab-add" onclick={addTab}>＋</button>
</div>

<!-- エディタエリア -->
<div class="container">
  <header>
    <span class="save-status">{saveStatus}</span>
  </header>

  <!-- タイトル入力 -->
  <input class="title-input" type="text" bind:value={title} placeholder="タイトルを入力" oninput={scheduleAutoSave} />

  <!-- ツールバー -->
  <div class="toolbar">
    <!-- 太字ボタン -->
    <button class="tool-btn bold-btn" onmousedown={(e) => e.preventDefault()} onclick={applyBold}>B</button>

    <div class="divider"></div>

    <!-- カラーパレット -->
    {#each colors as color}
      <button class="color-btn" style="background: {color.value}" title={color.label} onmousedown={(e) => e.preventDefault()} onclick={() => applyColor(color.value)}></button>
    {/each}

    <div class="divider"></div>

    <!-- フォントサイズ -->
    <div class="font-size-control">
      <button class="size-btn" onmousedown={(e) => e.preventDefault()} onclick={() => applyFontSize(fontSize - 2)}>▼</button>
      <span class="size-display">{fontSize}px</span>
      <button class="size-btn" onmousedown={(e) => e.preventDefault()} onclick={() => applyFontSize(fontSize + 2)}>▲</button>
    </div>
  </div>

  <!-- エディタ本文 -->
  <div class="editor" role="textbox" aria-multiline="true" tabindex="0" bind:this={editorEl} contenteditable="true" oninput={scheduleAutoSave}></div>
</div>

<style>
  /* -----------------------------------------------
     タブバー
  ----------------------------------------------- */
  .tab-bar {
    display: flex;
    align-items: flex-end;
    gap: 2px;
    padding: 8px 16px 0;
    background: #f5f5f5;
    border-bottom: 2px solid #ddd;
    overflow-x: auto;
  }

  .tab {
    position: relative;
    display: flex;
    align-items: center;
    padding: 6px 14px;
    background: #e0e0e0;
    border: 1px solid #ccc;
    border-bottom: none;
    border-radius: 6px 6px 0 0;
    cursor: pointer;
    font-size: 13px;
    color: #555;
    white-space: nowrap;
    max-width: 160px;
    min-width: 0;
    overflow: hidden;
    transition: background 0.15s;
  }

  .tab.active {
    background: white;
    color: #111;
    font-weight: bold;
    border-color: #ddd;
    position: relative;
    bottom: -2px;
  }

  .tab:hover:not(.active) {
    background: #ececec;
  }

  .tab-title {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    width: 100%;
  }

  .tab-close {
    position: absolute;
    right: 4px;
    top: 50%;
    transform: translateY(-50%);
    font-size: 12px;
    color: #666;
    line-height: 1;
    padding: 2px 4px;
    border-radius: 3px;
    background: #e0e0e0;
    opacity: 0;
    transition: opacity 0.15s;
    border: none;
    cursor: pointer;
  }

  .tab:hover .tab-close {
    opacity: 1;
  }

  .tab.active .tab-close {
    background: white;
  }

  .tab-close:hover {
    background: #ccc;
    color: #333;
  }

  .tab-add {
    padding: 6px 10px;
    background: transparent;
    border: none;
    cursor: pointer;
    font-size: 18px;
    color: #888;
    border-radius: 4px;
    line-height: 1;
    margin-bottom: 2px;
  }

  .tab-add:hover {
    background: #e0e0e0;
    color: #333;
  }

  /* -----------------------------------------------
     メインコンテンツ
  ----------------------------------------------- */
  .container {
    max-width: 800px;
    margin: 0 auto;
    padding: 24px 16px;
    display: flex;
    flex-direction: column;
    min-height: 100vh;
  }

  header {
    display: flex;
    justify-content: flex-end;
    align-items: center;
    margin-bottom: 16px;
  }

  .save-status {
    font-size: 13px;
    color: #999;
  }

  .title-input {
    width: 100%;
    font-size: 24px;
    font-weight: bold;
    border: none;
    outline: none;
    background: transparent;
    margin-bottom: 16px;
    padding: 8px 0;
    border-bottom: 1px solid #eee;
  }

  /* ツールバー */
  .toolbar {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 8px 12px;
    background: white;
    border: 1px solid #eee;
    border-radius: 8px;
    margin-bottom: 12px;
    flex-wrap: wrap;
  }

  .tool-btn {
    padding: 4px 10px;
    border: 1px solid #ddd;
    border-radius: 4px;
    background: white;
    cursor: pointer;
    font-size: 14px;
  }

  .tool-btn:hover {
    background: #f0f0f0;
  }

  .bold-btn {
    font-weight: bold;
  }

  .color-btn {
    width: 22px;
    height: 22px;
    border-radius: 50%;
    border: 2px solid rgba(0, 0, 0, 0.15);
    cursor: pointer;
    padding: 0;
  }

  .color-btn:hover {
    transform: scale(1.2);
    border-color: rgba(0, 0, 0, 0.4);
  }

  .font-size-control {
    display: flex;
    align-items: center;
    gap: 4px;
  }

  .size-btn {
    padding: 2px 6px;
    border: 1px solid #ddd;
    border-radius: 4px;
    background: white;
    cursor: pointer;
    font-size: 11px;
    line-height: 1;
  }

  .size-btn:hover {
    background: #f0f0f0;
  }

  .size-display {
    font-size: 13px;
    min-width: 36px;
    text-align: center;
    color: #555;
  }

  .divider {
    width: 1px;
    height: 20px;
    background: #ddd;
    margin: 0 2px;
  }

  .editor {
    flex: 1;
    min-height: 400px;
    outline: none;
    line-height: 1.5;
    padding: 8px 0;
    white-space: pre-wrap;
    word-break: break-word;
    font-size: 18px;
  }

  .editor:empty::before {
    content: "本文を入力...";
    color: #bbb;
    pointer-events: none;
  }
</style>
