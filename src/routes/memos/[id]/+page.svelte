<script>
  import { onMount } from "svelte";
  import { page } from "$app/stores";
  import { supabase } from "$lib/supabase";

  // URLからメモのIDを取得する
  const memoId = $derived($page.params.id);

  // メモの状態管理
  let title = $state("");
  let loading = $state(true);
  let saveStatus = $state("");

  // エディタのDOM要素への参照
  let editorEl = $state(null);

  let initialContent = $state("");

  // 自動保存用タイマー
  let saveTimer = null;

  // カラーパレットの定義
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

  // フォントサイズの状態管理（デフォルト18px）
  let fontSize = $state(18);

  // テキスト選択範囲を保存する変数
  let savedRange = null;
  let activeSpan = null; // フォントサイズ適用中のspan

  // ページ表示時にメモを取得する
  onMount(async () => {
    // エディタ内の選択範囲が変わるたびに保存する
    document.addEventListener("selectionchange", () => {
      const sel = window.getSelection();
      if (!sel || sel.rangeCount === 0) return;
      const range = sel.getRangeAt(0);
      if (editorEl && editorEl.contains(range.commonAncestorContainer)) {
        // 選択範囲のテキストが変わったらactiveSpanをリセットする
        const selectedText = range.toString();
        if (savedRange && savedRange.toString() !== selectedText) {
          activeSpan = null;
        }
        savedRange = range.cloneRange();
      }
    });

    const {
      data: { session },
    } = await supabase.auth.getSession();
    if (!session) {
      window.location.href = "/login";
      return;
    }

    const { data, error } = await supabase.from("memos").select("title, content").eq("id", memoId).single();

    if (error) {
      console.error("メモの取得に失敗しました", error);
    } else {
      title = data.title;
      initialContent = data.content;
    }

    loading = false;
  });

  // editorElとinitialContentが揃ったらHTMLをセットする
  $effect(() => {
    if (editorEl && initialContent) {
      editorEl.innerHTML = initialContent;
    }
  });

  // 太字を適用する
  function applyBold() {
    if (!savedRange) return;
    const sel = window.getSelection();
    sel.removeAllRanges();
    sel.addRange(savedRange);
    document.execCommand("bold");
    scheduleAutoSave();
  }

  // 文字色を適用する
  function applyColor(colorValue) {
    if (!savedRange) return;
    const sel = window.getSelection();
    sel.removeAllRanges();
    sel.addRange(savedRange);
    document.execCommand("foreColor", false, colorValue);
    scheduleAutoSave();
  }

  // フォントサイズを適用する
  function applyFontSize(newSize) {
    fontSize = Math.min(72, Math.max(8, newSize));

    // activeSpanがあれば直接fontSizeを書き換える（連続押し対応）
    if (activeSpan && editorEl.contains(activeSpan)) {
      activeSpan.style.fontSize = fontSize + "px";
      scheduleAutoSave();
      return;
    }

    // 選択範囲がなければ何もしない
    if (!savedRange || savedRange.collapsed) return;

    // 選択範囲をspanで包む
    const sel = window.getSelection();
    sel.removeAllRanges();
    sel.addRange(savedRange);
    const range = sel.getRangeAt(0);
    const span = document.createElement("span");
    span.style.fontSize = fontSize + "px";
    try {
      range.surroundContents(span);
      activeSpan = span; // spanへの参照を保持する
    } catch (e) {
      // 複数ノードにまたがる選択の場合はスキップ
    }

    scheduleAutoSave();
  }

  // 入力が止まったら1.5秒後に自動保存する
  function scheduleAutoSave() {
    clearTimeout(saveTimer);
    saveTimer = setTimeout(() => {
      saveMemo();
    }, 1500);
  }

  // Supabaseにメモを保存する
  async function saveMemo() {
    saveStatus = "保存中...";
    const { error } = await supabase
      .from("memos")
      .update({
        title,
        content: editorEl ? editorEl.innerHTML : "",
        updated_at: new Date().toISOString(),
      })
      .eq("id", memoId);

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

<div class="container">
  <header>
    <a href="/memos" class="back-btn">← 一覧へ</a>
    <span class="save-status">{saveStatus}</span>
  </header>

  {#if loading}
    <p class="message">読み込み中...</p>
  {:else}
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
  {/if}
</div>

<style>
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
    justify-content: space-between;
    align-items: center;
    margin-bottom: 16px;
  }

  .back-btn {
    color: #4a90e2;
    text-decoration: none;
    font-size: 14px;
  }

  .back-btn:hover {
    text-decoration: underline;
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

  /* カラーボタン */
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

  /* フォントサイズコントロール */
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

  /* エディタ本文 */
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

  .message {
    color: #999;
    margin-top: 48px;
  }
</style>
