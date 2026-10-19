<script setup>
import Icon from "../Icon.vue";

defineProps({
  options: { type: Array, default: () => [] },
  networkName: { type: String, default: "" },
});

const emit = defineEmits(["choose", "close"]);
</script>

<template>
  <div class="sheet-backdrop" @click.self="emit('close')">
    <section class="sheet" role="dialog" aria-modal="true" aria-labelledby="wallet-sheet-title">
      <header class="sheet-head">
        <div>
          <p class="kicker">
            <span class="kicker-dot" aria-hidden="true"></span>
            Connect
          </p>
          <h2 id="wallet-sheet-title">Choose a wallet</h2>
        </div>
        <button class="icon-btn" type="button" aria-label="Close wallet picker" @click="emit('close')">
          <Icon name="close" :size="18" />
        </button>
      </header>

      <p class="sheet-note">
        Pick a wallet that can sign on {{ networkName || "this cluster" }}. ORAsol only
        asks it to confirm quest rows, proposals and votes.
      </p>

      <ul v-if="options.length" class="wallet-grid">
        <li v-for="candidate in options" :key="candidate.name">
          <button type="button" @click="emit('choose', candidate)">
            <img
              v-if="candidate.icon"
              :src="candidate.icon"
              alt=""
              width="28"
              height="28"
            />
            <span v-else class="wallet-fallback" aria-hidden="true">
              <Icon name="wallet" :size="17" />
            </span>
            <span class="wallet-name">{{ candidate.name }}</span>
            <Icon name="arrow-right" :size="15" />
          </button>
        </li>
      </ul>

      <p v-else class="sheet-empty">
        No compatible wallet was detected in this browser. Install a Solana wallet
        extension, reload, then try again.
      </p>

      <footer class="sheet-foot">
        <p>Nothing is signed until you approve it inside the wallet.</p>
        <button class="btn btn-outline" type="button" @click="emit('close')">Cancel</button>
      </footer>
    </section>
  </div>
</template>
