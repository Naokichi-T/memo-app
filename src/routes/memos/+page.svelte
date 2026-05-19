<script>
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabase";

  // メモ一覧の状態管理
  let memos = $state([]);
  let loading = $state(true);

  // ドラッグ中のメモのインデックス
  let dragIndex = $state(null);
  // ドロップ先のインデックス
  let dropIndex = $state(null);

  // ページ表示時にメモ一覧を取得する
  onMount(async () => {
    await loadMemos();
  });

  // Supabaseからメモ一覧を取得する
  async function loadMemos() {
    loading = true;

    const {
      data: { session },
    } = await supabase.auth.getSession();
    if (!session) {
      window.location.href = "/login";
      return;
    }

    const { data, error } = await supabase.from("memos").select("id, title, sort_order").order("sort_order", { ascending: true });

    if (error) {
      console.error("メモの取得に失敗しました", error);
    } else {
      memos = data;
    }

    loading = false;
  }

  // 新規メモを作成してエディタへ移動する
  async function createMemo() {
    const {
      data: { session },
    } = await supabase.auth.getSession();

    const maxOrder = memos.length > 0 ? Math.max(...memos.map((m) => m.sort_order)) : 0;

    const { data, error } = await supabase
      .from("memos")
      .insert({
        user_id: session.user.id,
        title: "無題のメモ",
        content: "",
        sort_order: maxOrder + 1,
      })
      .select()
      .single();

    if (error) {
      console.error("メモの作成に失敗しました", error);
    } else {
      window.location.href = `/memos/${data.id}`;
    }
  }

  // メモを削除する
  async function deleteMemo(event, id) {
    event.stopPropagation();
    if (!confirm("このメモを削除しますか？")) return;

    const { error } = await supabase.from("memos").delete().eq("id", id);

    if (error) {
      console.error("メモの削除に失敗しました", error);
    } else {
      memos = memos.filter((m) => m.id !== id);
    }
  }

  // ログアウトする
  async function logout() {
    await supabase.auth.signOut();
    window.location.href = "/login";
  }

  // ---- ドラッグ＆ドロップ ----

  // ドラッグ開始時の処理
  function handleDragStart(index) {
    dragIndex = index;
  }

  // ドラッグ中に別のメモの上に重なったときの処理
  function handleDragOver(event, index) {
    // デフォルトのドロップ禁止を解除する
    event.preventDefault();
    dropIndex = index;
  }

  // ドラッグがメモ一覧から外れたときの処理
  function handleDragLeave() {
    dropIndex = null;
  }

  // ドロップ時の処理
  async function handleDrop(event, index) {
    event.preventDefault();

    if (dragIndex === null || dragIndex === index) {
      dragIndex = null;
      dropIndex = null;
      return;
    }

    // 配列の順番を入れ替える
    const newMemos = [...memos];
    const [moved] = newMemos.splice(dragIndex, 1);
    newMemos.splice(index, 0, moved);

    // sort_orderを1から振り直す
    const updated = newMemos.map((m, i) => ({ ...m, sort_order: i + 1 }));
    memos = updated;

    dragIndex = null;
    dropIndex = null;

    // Supabaseに保存する
    await saveSortOrder(updated);
  }

  // ドラッグ終了時の処理（ドロップ外でも呼ばれる）
  function handleDragEnd() {
    dragIndex = null;
    dropIndex = null;
  }

  // sort_orderをSupabaseに一括保存する
  async function saveSortOrder(updatedMemos) {
    const updates = updatedMemos.map((m) => supabase.from("memos").update({ sort_order: m.sort_order }).eq("id", m.id));
    await Promise.all(updates);
  }
</script>

<div class="container">
  <header>
    <h1>📝 メモ一覧</h1>
    <div class="header-actions">
      <button class="create-btn" onclick={createMemo}>＋ 新規作成</button>
      <button class="logout-btn" onclick={logout}>ログアウト</button>
    </div>
  </header>

  {#if loading}
    <p class="message">読み込み中...</p>
  {:else if memos.length === 0}
    <p class="message">メモがありません。新規作成してみましょう！</p>
  {:else}
    <div class="memo-list">
      {#each memos as memo, i (memo.id)}
        {#if dropIndex === i && dragIndex !== i}
          <div class="drop-line"></div>
        {/if}

        <div
          class="memo-item"
          class:dragging={dragIndex === i}
          role="button"
          tabindex="0"
          draggable="true"
          ondragstart={() => handleDragStart(i)}
          ondragover={(e) => handleDragOver(e, i)}
          ondragleave={handleDragLeave}
          ondrop={(e) => handleDrop(e, i)}
          ondragend={handleDragEnd}
          onclick={() => (window.location.href = `/memos/${memo.id}`)}
          onkeydown={(e) => e.key === "Enter" && (window.location.href = `/memos/${memo.id}`)}
        >
          <span class="drag-handle">⠿</span>
          <span class="memo-title">{memo.title || "無題のメモ"}</span>
          <button class="delete-btn" onclick={(e) => deleteMemo(e, memo.id)}>🗑</button>
        </div>
      {/each}

      {#if dropIndex === memos.length && dragIndex !== memos.length - 1}
        <div class="drop-line"></div>
      {/if}
    </div>
  {/if}
</div>

<style>
  .container {
    max-width: 600px;
    margin: 0 auto;
    padding: 24px 16px;
  }

  header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 24px;
  }

  h1 {
    font-size: 24px;
  }

  .header-actions {
    display: flex;
    gap: 8px;
  }

  .create-btn {
    padding: 8px 16px;
    background: #4a90e2;
    color: white;
    border: none;
    border-radius: 6px;
    font-size: 14px;
    cursor: pointer;
  }

  .create-btn:hover {
    background: #357abd;
  }

  .logout-btn {
    padding: 8px 16px;
    background: transparent;
    color: #999;
    border: 1px solid #ddd;
    border-radius: 6px;
    font-size: 14px;
    cursor: pointer;
  }

  .logout-btn:hover {
    background: #f5f5f5;
  }

  .memo-list {
    list-style: none;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .memo-item {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 16px;
    background: white;
    border-radius: 8px;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
    cursor: pointer;
    /* ドラッグ中の見た目をなめらかにする */
    transition: opacity 0.15s;
  }

  .memo-item:hover {
    background: #f0f7ff;
  }

  /* ドラッグ中のメモを半透明にする */
  .memo-item.dragging {
    opacity: 0.4;
  }

  .drag-handle {
    color: #ccc;
    cursor: grab;
    font-size: 18px;
    padding: 0 4px;
    user-select: none;
  }

  .drag-handle:active {
    cursor: grabbing;
  }

  .memo-title {
    flex: 1;
    font-size: 16px;
  }

  .delete-btn {
    background: transparent;
    border: none;
    font-size: 18px;
    cursor: pointer;
    padding: 4px 8px;
    border-radius: 4px;
    opacity: 0.5;
  }

  .delete-btn:hover {
    opacity: 1;
    background: #ffe5e5;
  }

  /* ドロップ位置を示すライン */
  .drop-line {
    height: 3px;
    background: #4a90e2;
    border-radius: 2px;
    margin: -4px 0;
  }

  .message {
    text-align: center;
    color: #999;
    margin-top: 48px;
  }
</style>
