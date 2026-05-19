<script>
  // 全ページ共通のレイアウト
  let { children } = $props();
</script>

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
