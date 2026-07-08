<script lang="ts">
import { onMount } from "svelte";

export let slug: string;

const SUPABASE_URL = "https://xvmhehekolonxizsrabj.supabase.co";
const SUPABASE_KEY = "sb_publishable_5XMcq7_hzXTrW9g-TZiSPQ_-dVnJ8eD";

const emojiList = [
	{ emoji: "👍", label: "Like" },
	{ emoji: "❤️", label: "Love it" },
	{ emoji: "🔥", label: "Impressive" },
	{ emoji: "💡", label: "Insightful" },
	{ emoji: "🎯", label: "Nailed it" },
];

let counts: Record<string, number> = {};
let reacted: Record<string, boolean> = {};
let popping: Record<string, boolean> = {};
let loaded = false;

function storageKey(emoji: string) {
	return `reacted-${slug}-${emoji}`;
}

onMount(async () => {
	for (const { emoji } of emojiList) {
		reacted[emoji] = localStorage.getItem(storageKey(emoji)) === "1";
	}

	try {
		const res = await fetch(
			`${SUPABASE_URL}/rest/v1/post_reactions?post_slug=eq.${encodeURIComponent(slug)}&select=emoji,count`,
			{
				headers: {
					apikey: SUPABASE_KEY,
					Authorization: `Bearer ${SUPABASE_KEY}`,
				},
			},
		);
		if (res.ok) {
			const rows: { emoji: string; count: number }[] = await res.json();
			for (const row of rows) {
				counts[row.emoji] = row.count;
			}
			counts = { ...counts };
		}
	} catch {
		// silently ignore, reactions are a nice-to-have, not critical
	}
	loaded = true;
});

async function react(emoji: string) {
	if (reacted[emoji]) return;

	reacted[emoji] = true;
	reacted = { ...reacted };
	counts[emoji] = (counts[emoji] || 0) + 1;
	counts = { ...counts };
	localStorage.setItem(storageKey(emoji), "1");

	popping[emoji] = true;
	popping = { ...popping };
	setTimeout(() => {
		popping[emoji] = false;
		popping = { ...popping };
	}, 420);

	try {
		await fetch(`${SUPABASE_URL}/rest/v1/rpc/increment_reaction`, {
			method: "POST",
			headers: {
				apikey: SUPABASE_KEY,
				Authorization: `Bearer ${SUPABASE_KEY}`,
				"Content-Type": "application/json",
			},
			body: JSON.stringify({ p_slug: slug, p_emoji: emoji }),
		});
	} catch {
		// optimistic update already applied, a failed request just means
		// the count may be slightly behind server state on next load
	}
}
</script>

<div class="flex flex-wrap items-center justify-center gap-3 py-2">
	{#each emojiList as { emoji, label } (emoji)}
		<button
			type="button"
			aria-label={label}
			title={label}
			on:click={() => react(emoji)}
			disabled={reacted[emoji]}
			class="reaction-btn card-base flex flex-col items-center justify-center gap-0.5 w-16 h-16 rounded-full transition disabled:cursor-default"
			class:reacted={reacted[emoji]}
			class:pop={popping[emoji]}
		>
			<span class="reaction-emoji text-2xl leading-none">{emoji}</span>
			<span class="text-50 text-xs tabular-nums leading-none">{loaded ? (counts[emoji] || 0) : ""}</span>
		</button>
	{/each}
</div>

<style>
	.reaction-btn {
		box-shadow: 0 1px 2px rgba(0, 0, 0, 0.06);
	}
	.reaction-btn:hover:not(:disabled) {
		transform: translateY(-3px) scale(1.08);
		box-shadow: 0 8px 16px -4px rgba(0, 0, 0, 0.18);
	}
	.reaction-btn:hover:not(:disabled) .reaction-emoji {
		animation: wiggle 0.5s ease-in-out infinite;
	}
	.reaction-btn.reacted {
		box-shadow: inset 0 0 0 1.5px var(--primary);
	}
	.reaction-btn.pop .reaction-emoji {
		animation: pop 0.42s cubic-bezier(0.34, 1.56, 0.64, 1);
	}

	@keyframes wiggle {
		0%, 100% { transform: rotate(0deg); }
		25% { transform: rotate(-10deg); }
		75% { transform: rotate(10deg); }
	}

	@keyframes pop {
		0% { transform: scale(1); }
		40% { transform: scale(1.6) rotate(-8deg); }
		70% { transform: scale(0.9) rotate(4deg); }
		100% { transform: scale(1) rotate(0deg); }
	}

	@media (prefers-reduced-motion: reduce) {
		.reaction-btn:hover:not(:disabled) .reaction-emoji,
		.reaction-btn.pop .reaction-emoji {
			animation: none;
		}
	}
</style>
