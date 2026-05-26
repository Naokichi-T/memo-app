<script>
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabase";

  // -----------------------------------------------
  // localStorageのキー定数
  // -----------------------------------------------
  const ACTIVE_TAB_KEY = "memo-active-tab"; // 最後にアクティブだったタブのID

  // -----------------------------------------------
  // タブ管理
  // -----------------------------------------------
  // 開いているタブのリスト { id, title, sort_order }
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
  // タブのドラッグ＆ドロップ管理
  // -----------------------------------------------
  // ドラッグ中のタブのindex
  let dragIndex = $state(null);
  // ドロップ先のindex
  let dropIndex = $state(null);

  // -----------------------------------------------
  // Supabaseから全メモを取得してタブに並べる
  // -----------------------------------------------
  async function loadAllMemos() {
    const { data, error } = await supabase.from("memos").select("id, title, sort_order").order("sort_order", { ascending: true });

    if (error) {
      console.error("メモ一覧の取得に失敗しました", error);
      return [];
    }
    return data;
  }

  // -----------------------------------------------
  // Supabaseから1件のメモ本文を取得してキャッシュに保存する
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
  }

  // -----------------------------------------------
  // Supabaseに新規メモを作成してそのIDを返す
  // -----------------------------------------------
  async function createNewMemo() {
    const {
      data: { session },
    } = await supabase.auth.getSession();

    // 現在の最大sort_orderを取得して+1する
    const maxOrder = openTabs.length > 0 ? Math.max(...openTabs.map((t) => t.sort_order ?? 0)) : 0;

    const { data, error } = await supabase
      .from("memos")
      .insert({
        title: "無題",
        content: "",
        user_id: session.user.id,
        sort_order: maxOrder + 1,
      })
      .select("id, title, sort_order")
      .single();

    if (error) {
      console.error("新規メモの作成に失敗しました", error);
      return null;
    }
    return data;
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
    const newMemo = await createNewMemo();
    if (!newMemo) return;

    // キャッシュに空のメモを登録する
    memoCache = {
      ...memoCache,
      [newMemo.id]: { title: "無題", content: "" },
    };

    // タブリストに追加する
    openTabs = [...openTabs, { id: newMemo.id, title: "無題", sort_order: newMemo.sort_order }];

    // 新しいタブに切り替える
    switchTab(newMemo.id);
  }

  // -----------------------------------------------
  // ×ボタン：メモを削除してタブを閉じる
  // -----------------------------------------------
  async function closeTab(id) {
    // 確認ダイアログを表示する
    if (!confirm("このメモを削除しますか？")) return;

    const index = openTabs.findIndex((t) => t.id === id);

    // Supabaseからメモを削除する
    const { error } = await supabase.from("memos").delete().eq("id", id);
    if (error) {
      console.error("メモの削除に失敗しました", error);
      return;
    }

    // タブリストから削除する
    openTabs = openTabs.filter((t) => t.id !== id);

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
  // タブのドラッグ開始時の処理
  // -----------------------------------------------
  function handleTabDragStart(e, index) {
    dragIndex = index;
    e.dataTransfer.effectAllowed = "move";
  }

  // -----------------------------------------------
  // タブのドラッグ中・別タブの上を通過したときの処理
  // -----------------------------------------------
  function handleTabDragOver(e, index) {
    e.preventDefault();
    e.dataTransfer.dropEffect = "move";
    dropIndex = index;
  }

  // -----------------------------------------------
  // タブのドラッグがタブバーから外れたときの処理
  // -----------------------------------------------
  function handleTabDragLeave() {
    dropIndex = null;
  }

  // -----------------------------------------------
  // タブのドロップ時の処理（順番を入れ替えてSupabaseに保存する）
  // -----------------------------------------------
  async function handleTabDrop(e, index) {
    e.preventDefault();

    if (dragIndex === null || dragIndex === index) {
      dragIndex = null;
      dropIndex = null;
      return;
    }

    // タブの順番を入れ替える
    const newTabs = [...openTabs];
    const [moved] = newTabs.splice(dragIndex, 1);
    newTabs.splice(index, 0, moved);

    // sort_orderを1から振り直す
    const updated = newTabs.map((t, i) => ({ ...t, sort_order: i + 1 }));
    openTabs = updated;

    dragIndex = null;
    dropIndex = null;

    // Supabaseにsort_orderを一括保存する
    await saveSortOrder(updated);
  }

  // -----------------------------------------------
  // タブのドラッグ終了時の処理（ドロップ外でも呼ばれる）
  // -----------------------------------------------
  function handleTabDragEnd() {
    dragIndex = null;
    dropIndex = null;
  }

  // -----------------------------------------------
  // sort_orderをSupabaseに一括保存する
  // -----------------------------------------------
  async function saveSortOrder(tabs) {
    const updates = tabs.map((t) => supabase.from("memos").update({ sort_order: t.sort_order }).eq("id", t.id));
    await Promise.all(updates);
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

    // Supabaseから全メモを取得してタブに並べる
    const allMemos = await loadAllMemos();

    // メモが0件なら新規メモを作成する
    if (allMemos.length === 0) {
      await addTab();
      return;
    }

    openTabs = allMemos.map((m) => ({
      id: m.id,
      title: m.title ?? "無題",
      sort_order: m.sort_order,
    }));

    // 最後にアクティブだったタブのIDをlocalStorageから復元する
    const savedActiveId = localStorage.getItem(ACTIVE_TAB_KEY);
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
  {#each openTabs as tab, i}
    <!-- ドロップ先インジケーター（タブの左側に縦線を表示） -->
    {#if dropIndex === i && dragIndex !== i}
      <div class="tab-drop-line"></div>
    {/if}

    <button
      class="tab"
      class:active={tab.id === activeTabId}
      class:dragging={dragIndex === i}
      draggable="true"
      onclick={() => switchTab(tab.id)}
      ondragstart={(e) => handleTabDragStart(e, i)}
      ondragover={(e) => handleTabDragOver(e, i)}
      ondragleave={handleTabDragLeave}
      ondrop={(e) => handleTabDrop(e, i)}
      ondragend={handleTabDragEnd}
    >
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

  <!-- 末尾のドロップ先インジケーター -->
  {#if dropIndex === openTabs.length && dragIndex !== openTabs.length - 1}
    <div class="tab-drop-line"></div>
  {/if}

  <!-- ＋ボタン -->
  <button class="tab-add" onclick={addTab}>＋</button>
</div>

<!-- ツールバー（containerの外・タブバーの直下に固定） -->
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

<!-- エディタエリア -->
<div class="container">
  <header>
    <span class="save-status">{saveStatus}</span>
  </header>

  <!-- タイトル入力 -->
  <input class="title-input" type="text" bind:value={title} placeholder="タイトルを入力" oninput={scheduleAutoSave} />

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

  /* ドラッグ中のタブを半透明にする */
  .tab.dragging {
    opacity: 0.4;
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

  /* ドロップ先を示す縦線 */
  .tab-drop-line {
    width: 2px;
    height: 28px;
    background: #4a90e2;
    border-radius: 2px;
    flex-shrink: 0;
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

  /* ツールバー：タブバーの直下に固定する */
  .toolbar {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 8px 12px;
    background: white;
    border-bottom: 1px solid #eee;
    /* border-radius は削除（固定時は角丸なしの方が自然） */
    /* margin-bottom は削除（containerのpaddingで調整） */
    flex-wrap: wrap;
    /* 画面上部に固定する（タブバーの高さ分だけ下にずらす） */
    position: sticky;
    top: 0px; /* タブバーのおおよその高さ */
    z-index: 99;
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
