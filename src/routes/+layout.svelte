<script>
  import favicon from "$lib/assets/favicon.svg";
  // 全ページ共通のレイアウト
  let { children } = $props();
</script>

<svelte:head>
  <link rel="icon" href={favicon} />
  <title>メモ帳</title>
</svelte:head>

{@render children()}

<style>
  :global(*, *::before, *::after) {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  :global(body) {
    font-family: sans-serif;
    background: #f5f5f5;
    color: #333;
  }
</style>
