<script>
  // トップページ：ログイン状態に応じてリダイレクト
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabase";

  onMount(async () => {
    const {
      data: { session },
    } = await supabase.auth.getSession();
    if (session) {
      // ログイン済みなら一覧へ
      window.location.href = "/memos";
    } else {
      // 未ログインならログインページへ
      window.location.href = "/login";
    }
  });
</script>
