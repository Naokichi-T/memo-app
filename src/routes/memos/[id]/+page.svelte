<script>
  import { onMount } from "svelte";
  import { page } from "$app/stores";
  import { supabase } from "$lib/supabase";

  // URLからメモのIDを取得する
  const memoId = $derived($page.params.id);

  // メモの状態管理
  let title = $state("");
  let content = $state("");
  let loading = $state(true);
  let saveStatus = $state(""); // '' | '保存中...' | '保存しました'

  // 自動保存用タイマー
  let saveTimer = null;

  // ページ表示時にメモを取得する
  onMount(async () => {
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
      content = data.content;
    }

    loading = false;
  });

  // 入力が止まったら1.5秒後に自動保存する
  function scheduleAutoSave() {
    // 前のタイマーをリセット
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
        content,
        updated_at: new Date().toISOString(),
      })
      .eq("id", memoId);

    if (error) {
      console.error("保存に失敗しました", error);
      saveStatus = "保存失敗";
    } else {
      saveStatus = "保存しました";
      // 2秒後にステータス表示を消す
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

    <textarea class="content-input" bind:value={content} placeholder="本文を入力..." oninput={scheduleAutoSave}></textarea>
  {/if}
</div>

<style>
  .container {
    max-width: 800px;
    margin: 0 auto;
    padding: 24px 16px;
    display: flex;
    flex-direction: column;
    height: 100vh;
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

  .content-input {
    width: 100%;
    flex: 1;
    font-size: 16px;
    border: none;
    outline: none;
    background: transparent;
    resize: none;
    line-height: 1.8;
  }
</style>
