<script setup>
import Icon from "../Icon.vue";
import brandMark from "../assets/orasol-mark.svg";

defineProps({
  tabs: { type: Array, default: () => [] },
  networkName: { type: String, default: "" },
  cluster: { type: String, default: "" },
  programState: { type: String, default: "" },
  mintState: { type: String, default: "" },
  walletLabel: { type: String, default: "" },
  lastRead: { type: String, default: "" },
});

const emit = defineEmits(["navigate"]);
</script>

<template>
  <footer class="site-footer">
    <div class="footer-inner">
      <div class="footer-brand">
        <span class="footer-mark">
          <img :src="brandMark" width="28" height="28" alt="" />
          <b>ORA<span class="footer-sol">sol</span></b>
        </span>
        <p class="footer-tag">Time on the record.</p>
        <p class="footer-note">
          An independent project reading {{ networkName }}. Quests and advisory votes are
          recorded by a public Solana program; no token is minted, sold or promised here.
        </p>
      </div>

      <div class="footer-col">
        <h3>Explore</h3>
        <nav class="footer-links" aria-label="Footer">
          <button
            v-for="tab in tabs"
            :key="tab.id"
            type="button"
            @click="emit('navigate', tab.id)"
          >{{ tab.label }}</button>
        </nav>
      </div>

      <div class="footer-col">
        <h3>Network</h3>
        <div class="footer-facts">
          <div><span>Cluster</span><b>{{ cluster }}</b></div>
          <div><span>Network</span><b>{{ networkName }}</b></div>
          <div><span>Program</span><b>{{ programState }}</b></div>
          <div><span>Token mint</span><b>{{ mintState }}</b></div>
        </div>
      </div>

      <div class="footer-col">
        <h3>Session</h3>
        <div class="footer-facts">
          <div><span>Wallet</span><b>{{ walletLabel }}</b></div>
          <div><span>Last read</span><b>{{ lastRead }}</b></div>
        </div>
        <p class="footer-note">
          Never share a seed phrase or private key to fix a connection.
        </p>
      </div>
    </div>

    <div class="footer-wordmark" aria-hidden="true">ORA<span class="wordmark-sol">sol</span></div>

    <div class="footer-fine">
      <span>© 2026 ORAsol · orasol.online. Written to be reopened.</span>
      <span>Independent community project · not affiliated with the Solana Foundation.</span>
    </div>
  </footer>
</template>
