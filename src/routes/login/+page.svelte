<script>
  // ログインページ
  import { supabase } from "$lib/supabase";

  // 入力値の状態管理
  let email = $state("");
  let password = $state("");
  let errorMessage = $state("");
  let loading = $state(false);

  // ログインボタンを押したときの処理
  async function handleLogin() {
    loading = true;
    errorMessage = "";

    const { error } = await supabase.auth.signInWithPassword({ email, password });

    if (error) {
      errorMessage = "メールアドレスまたはパスワードが違います";
      loading = false;
    } else {
      window.location.href = "/memos";
    }
  }
</script>

<div class="container">
  <h1>ログイン</h1>

  <div class="form">
    <label>
      メールアドレス
      <input type="email" bind:value={email} placeholder="example@email.com" />
    </label>

    <label>
      パスワード
      <input type="password" bind:value={password} placeholder="パスワード" />
    </label>

    {#if errorMessage}
      <p class="error">{errorMessage}</p>
    {/if}

    <button onclick={handleLogin} disabled={loading}>
      {loading ? "ログイン中..." : "ログイン"}
    </button>
  </div>
</div>

<style>
  .container {
    max-width: 400px;
    margin: 80px auto;
    padding: 0 16px;
  }

  h1 {
    font-size: 24px;
    margin-bottom: 24px;
    text-align: center;
  }

  .form {
    background: white;
    padding: 32px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  label {
    display: flex;
    flex-direction: column;
    gap: 6px;
    font-size: 14px;
    font-weight: bold;
  }

  input {
    padding: 10px 12px;
    border: 1px solid #ddd;
    border-radius: 6px;
    font-size: 16px;
    outline: none;
  }

  input:focus {
    border-color: #4a90e2;
  }

  button {
    padding: 12px;
    background: #4a90e2;
    color: white;
    border: none;
    border-radius: 6px;
    font-size: 16px;
    cursor: pointer;
  }

  button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .error {
    color: #e24a4a;
    font-size: 14px;
  }
</style>
