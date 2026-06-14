<script lang="ts">
  export let postSlug: string;
  export let serverUrl: string;

  type Comment = {
    id: string;
    post_slug: string;
    author: string;
    content: string;
    created_at: string;
    reply_to?: string;
    replies?: Comment[];
  };

  let comments: Comment[] = [];
  let loading = true;
  let error = '';
  let author = '';
  let content = '';
  let submitting = false;
  let replyTo: string | null = null;

  async function fetchComments() {
    try {
      const res = await fetch(`${serverUrl}/comments?post_slug=${encodeURIComponent(postSlug)}`);
      if (!res.ok) throw new Error('Failed to load comments');
      const data = await res.json();
      comments = buildTree(data.comments || []);
    } catch (e: any) {
      error = e.message;
    } finally {
      loading = false;
    }
  }

  function buildTree(flat: Comment[]): Comment[] {
    const map = new Map<string, Comment>();
    const roots: Comment[] = [];
    for (const c of flat) {
      c.replies = [];
      map.set(c.id, c);
    }
    for (const c of flat) {
      if (c.reply_to && map.has(c.reply_to)) {
        map.get(c.reply_to)!.replies!.push(c);
      } else {
        roots.push(c);
      }
    }
    return roots;
  }

  async function submit() {
    if (!author.trim() || !content.trim()) return;
    submitting = true;
    try {
      const body: any = {
        post_slug: postSlug,
        author: author.trim(),
        content: content.trim(),
      };
      if (replyTo) body.reply_to = replyTo;

      const res = await fetch(`${serverUrl}/comments`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
      });
      if (!res.ok) {
        const data = await res.json().catch(() => ({}));
        throw new Error(data.error || 'Failed to submit');
      }
      content = '';
      replyTo = null;
      await fetchComments();
    } catch (e: any) {
      error = e.message;
    } finally {
      submitting = false;
    }
  }

  function startReply(id: string, name: string) {
    replyTo = id;
    content = `@${name} `;
  }

  function cancelReply() {
    replyTo = null;
    content = '';
  }

  function formatDate(d: string) {
    return new Date(d).toLocaleString('zh-CN');
  }

  fetchComments();
</script>

<div class="backend-comments mt-4">
  <h3 class="text-lg font-bold mb-4 text-black/80 dark:text-white/80">评论</h3>

  <!-- Comment Form -->
  <div class="comment-form card-base p-4 mb-4 rounded-xl">
    {#if replyTo}
      <div class="flex items-center gap-2 mb-2 text-sm text-black/50 dark:text-white/50">
        <span>回复中...</span>
        <button on:click={cancelReply} class="text-red-400 hover:text-red-500">取消</button>
      </div>
    {/if}
    <input
      bind:value={author}
      placeholder="昵称"
      class="w-full mb-2 px-3 py-2 rounded-lg bg-black/5 dark:bg-white/10
             text-black/80 dark:text-white/80 border-none outline-none
             focus:ring-2 focus:ring-[var(--primary)]"
    />
    <textarea
      bind:value={content}
      placeholder="写下你的评论..."
      rows="3"
      class="w-full mb-2 px-3 py-2 rounded-lg bg-black/5 dark:bg-white/10
             text-black/80 dark:text-white/80 border-none outline-none resize-y
             focus:ring-2 focus:ring-[var(--primary)]"
    ></textarea>
    <button
      on:click={submit}
      disabled={submitting || !author.trim() || !content.trim()}
      class="px-4 py-2 rounded-lg bg-[var(--primary)] text-white font-medium
             hover:opacity-90 disabled:opacity-50 disabled:cursor-not-allowed
             transition"
    >
      {submitting ? '提交中...' : '发表评论'}
    </button>
  </div>

  <!-- Comments List -->
  {#if loading}
    <div class="text-center py-4 text-black/40 dark:text-white/40">加载中...</div>
  {:else if error}
    <div class="text-center py-4 text-red-400">{error}</div>
  {:else if comments.length === 0}
    <div class="text-center py-4 text-black/40 dark:text-white/40">暂无评论，来说点什么吧！</div>
  {:else}
    <div class="comments-list space-y-3">
      {#each comments as comment}
        <div class="comment card-base p-4 rounded-xl">
          <div class="flex items-center gap-2 mb-2">
            <span class="font-semibold text-black/80 dark:text-white/80">{comment.author}</span>
            <span class="text-xs text-black/40 dark:text-white/40">{formatDate(comment.created_at)}</span>
          </div>
          <p class="text-black/70 dark:text-white/70 whitespace-pre-wrap mb-2">{comment.content}</p>
          <button
            on:click={() => startReply(comment.id, comment.author)}
            class="text-sm text-[var(--primary)] hover:opacity-80"
          >回复</button>

          {#if comment.replies && comment.replies.length > 0}
            <div class="replies ml-4 mt-3 space-y-3 border-l-2 border-black/10 dark:border-white/10 pl-4">
              {#each comment.replies as reply}
                <div class="reply">
                  <div class="flex items-center gap-2 mb-1">
                    <span class="font-semibold text-sm text-black/80 dark:text-white/80">{reply.author}</span>
                    <span class="text-xs text-black/40 dark:text-white/40">{formatDate(reply.created_at)}</span>
                  </div>
                  <p class="text-sm text-black/70 dark:text-white/70 whitespace-pre-wrap mb-1">{reply.content}</p>
                  <button
                    on:click={() => startReply(reply.id, reply.author)}
                    class="text-xs text-[var(--primary)] hover:opacity-80"
                  >回复</button>
                </div>
              {/each}
            </div>
          {/if}
        </div>
      {/each}
    </div>
  {/if}
</div>
