<script setup>
import Icon from "../Icon.vue";
import brandMark from "../assets/orasol-mark.svg";

defineProps({
  tabs: { type: Array, default: () => [] },
  active: { type: String, default: "home" },
  walletLabel: { type: String, default: "Connect wallet" },
  connected: { type: Boolean, default: false },
  reading: { type: Boolean, default: false },
  busy: { type: Boolean, default: false },
  connecting: { type: Boolean, default: false },
  cluster: { type: String, default: "" },
  menuOpen: { type: Boolean, default: false },
});

const emit = defineEmits(["navigate", "sync", "wallet", "menu"]);
</script>

<template>
  <header class="site-header">
    <div class="header-inner">
      <button class="brand" type="button" aria-label="ORAsol home" @click="emit('navigate', 'home')">
        <span class="brand-mark" aria-hidden="true">
          <img :src="brandMark" width="34" height="34" alt="" />
        </span>
        <span class="brand-text">
          <span class="brand-name">ORA<span class="brand-sol">sol</span></span>
          <span class="brand-sub">Time on the record</span>
        </span>
      </button>

      <nav class="header-nav" aria-label="Primary">
        <button
          v-for="tab in tabs"
          :key="tab.id"
          class="nav-pill"
          :class="{ 'is-active': active === tab.id }"
          :aria-current="active === tab.id ? 'page' : undefined"
          type="button"
          @click="emit('navigate', tab.id)"
        >
          <Icon :name="tab.icon" :size="17" />
          <span>{{ tab.label }}</span>
          <b v-if="tab.badge !== null && tab.badge !== undefined">{{ tab.badge }}</b>
        </button>
      </nav>

      <div class="header-tools">
        <span v-if="cluster" class="cluster-chip">
          <i aria-hidden="true"></i>{{ cluster }}
        </span>
        <button
          class="icon-btn"
          type="button"
          :disabled="reading || busy"
          :aria-label="`Re-read ${cluster || 'network'} data`"
          @click="emit('sync')"
        >
          <Icon name="refresh" :size="18" :class="{ spin: reading }" />
        </button>
        <button
          class="btn btn-primary btn-wallet"
          type="button"
          :disabled="connecting || busy"
          :aria-label="connected ? 'Disconnect wallet' : 'Connect a Solana wallet'"
          @click="emit('wallet')"
        >
          <Icon name="wallet" :size="16" />
          <span>{{ connecting ? "Connecting…" : walletLabel }}</span>
        </button>
        <button
          class="icon-btn burger"
          type="button"
          :aria-expanded="menuOpen ? 'true' : 'false'"
          :aria-label="menuOpen ? 'Close menu' : 'Open menu'"
          @click="emit('menu')"
        >
          <Icon :name="menuOpen ? 'close' : 'menu'" :size="20" />
        </button>
      </div>
    </div>
  </header>
</template>
