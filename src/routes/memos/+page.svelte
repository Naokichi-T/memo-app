<script>
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabase";

  // メモ一覧の状態管理
  let memos = $state([]);
  let loading = $state(true);

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

    // sort_orderは現在の最大値+1にする
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
      // 作成したメモのエディタへ移動
      window.location.href = `/memos/${data.id}`;
    }
  }

  // メモを削除する
  async function deleteMemo(event, id) {
    // クリックがメモ本体に伝わらないようにする
    event.stopPropagation();

    if (!confirm("このメモを削除しますか？")) return;

    const { error } = await supabase.from("memos").delete().eq("id", id);

    if (error) {
      console.error("メモの削除に失敗しました", error);
    } else {
      // 画面から該当メモを取り除く
      memos = memos.filter((m) => m.id !== id);
    }
  }

  // ログアウトする
  async function logout() {
    await supabase.auth.signOut();
    window.location.href = "/login";
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
    <ul class="memo-list">
      {#each memos as memo (memo.id)}
        <li class="memo-item" onclick={() => (window.location.href = `/memos/${memo.id}`)}>
          <span class="memo-title">{memo.title || "無題のメモ"}</span>
          <button class="delete-btn" onclick={(e) => deleteMemo(e, memo.id)}>🗑</button>
        </li>
      {/each}
    </ul>
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
    justify-content: space-between;
    padding: 16px;
    background: white;
    border-radius: 8px;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
    cursor: pointer;
  }

  .memo-item:hover {
    background: #f0f7ff;
  }

  .memo-title {
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

  .message {
    text-align: center;
    color: #999;
    margin-top: 48px;
  }
</style>
