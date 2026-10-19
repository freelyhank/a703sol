<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick, watch } from "vue";
import { useCommunityApp } from "./useCommunityApp";
import {
  SOLANA_CLUSTER,
  SOLANA_NETWORK_NAME,
  TOKEN_MINT_ADDRESS,
} from "./solana.js";
import Icon from "./Icon.vue";
import SiteHeader from "./components/SiteHeader.vue";
import SiteFooter from "./components/SiteFooter.vue";
import WalletSheet from "./components/WalletSheet.vue";

const communityApp = useCommunityApp();
const {
  configured,
  mintConfigured,
  programReady,
  wallet,
  token,
  tokenName,
  tokenSymbol,
  tokenDecimals,
  native,
  treasury,
  treasuryKnown,
  treasuryNativeKnown,
  treasuryTokenKnown,
  treasuryError,
  totalSupply,
  treasuryBalance,
  treasuryNative,
  paused,
  quests,
  proposals,
  accountError,
  accountReading,
  accountReady,
  readError,
  reading,
  ready,
  lastRead,
  connected,
  isOwner,
  busy,
  txState,
  connecting,
  short,
  units,
  solUnits,
  date,
  explorer,
  notify,
  refresh,
  connect,
  disconnectWallet,
  walletOptions,
  walletPickerOpen,
  closeWalletPicker,
  chooseWallet,
  claimQuest: recordQuest,
  createQuest: publishQuest,
  setQuestActive: changeQuestActive,
  createProposal: publishProposal,
  castVote,
  questStatus,
  proposalStatus,
  votePercent,
} = communityApp;

/* ------------------------------------------------------------------ routes */

const routes = [
  { id: "home", label: "Overview", icon: "portico" },
  { id: "quests", label: "Quests", icon: "flag" },
  { id: "governance", label: "Votes", icon: "ballot" },
  { id: "treasury", label: "Treasury", icon: "safe" },
  { id: "stake", label: "Token", icon: "coin" },
  { id: "learn", label: "Guide", icon: "book" },
];
const validTabs = routes.map((entry) => entry.id);
const route = ref(
  validTabs.includes(location.hash.slice(1)) ? location.hash.slice(1) : "home",
);

const menuOpen = ref(false);

function syncFromHash() {
  const next = location.hash.slice(1);
  route.value = validTabs.includes(next) ? next : "home";
  menuOpen.value = false;
}

async function go(next) {
  menuOpen.value = false;
  if (route.value !== next) history.pushState(null, "", `#${next}`);
  route.value = next;
  focusMain();
  await nextTick();
  window.scrollTo({
    top: 0,
    behavior: matchMedia("(prefers-reduced-motion: reduce)").matches
      ? "instant"
      : "smooth",
  });
}

function navigate(next) {
  go(next);
}

function toggleMenu() {
  menuOpen.value = !menuOpen.value;
}

function dismissToast() {
  txState.value = { stage: "idle", label: "", message: "", hash: "" };
}

function focusMain() {
  document.getElementById("main")?.focus({ preventScroll: true });
}

function onKeydown(event) {
  if (event.key !== "Escape") return;
  if (walletPickerOpen.value) closeWalletPicker();
  if (menuOpen.value) menuOpen.value = false;
}

onMounted(() => {
  window.addEventListener("hashchange", syncFromHash);
  window.addEventListener("popstate", syncFromHash);
  window.addEventListener("keydown", onKeydown);
});
onUnmounted(() => {
  window.removeEventListener("hashchange", syncFromHash);
  window.removeEventListener("popstate", syncFromHash);
  window.removeEventListener("keydown", onKeydown);
});

/* ---------------------------------------------------------------- overview */

const samples = [
  {
    id: "sample-1",
    sample: true,
    title: "Read the community charter",
    reward: 12n * 10n ** 9n,
    detail: "A sample welcome quest. See how a recorded row looks.",
  },
  {
    id: "sample-2",
    sample: true,
    title: "Map one useful signal",
    reward: 28n * 10n ** 9n,
    detail: "A sample research quest. Share a source-backed observation.",
  },
  {
    id: "sample-3",
    sample: true,
    title: "Review a treasury route",
    reward: 45n * 10n ** 9n,
    detail: "A sample review quest. Compare options and explain your reasoning.",
  },
];
const sampleProposals = [
  {
    id: "sample-1",
    sample: true,
    description: "Reserve a season budget for community research bounties",
    forVotes: 0n,
    againstVotes: 0n,
    endsAt: 0,
  },
];

const allQuests = computed(() => (programReady.value ? quests.value : samples));
const visibleProposals = computed(() =>
  programReady.value ? proposals.value : sampleProposals,
);
const openQuests = computed(
  () => quests.value.filter((q) => questStatus(q) === "Open").length,
);
const openProposals = computed(
  () => proposals.value.filter((p) => proposalStatus(p) === "Open").length,
);
const metricsAvailable = computed(() => programReady.value && ready.value);
const registerQuests = computed(() => allQuests.value.slice(0, 4));
const registerProposals = computed(() => visibleProposals.value.slice(0, 3));

const leadProposal = computed(() => registerProposals.value[0] || null);
const leadBreakdown = computed(() => voteBreakdown(leadProposal.value));
const leadSupport = computed(() => {
  if (!leadProposal.value || leadBreakdown.value.empty) return null;
  return votePercent(leadProposal.value);
});

const todayLabel = computed(() =>
  new Date().toLocaleDateString("en-US", {
    weekday: "long",
    month: "long",
    day: "numeric",
  }),
);

const programState = computed(() =>
  programReady.value
    ? "live"
    : configured.value
      ? "pending"
      : "not set",
);

const mintState = computed(() =>
  mintConfigured.value ? short(TOKEN_MINT_ADDRESS) : "none",
);

const navTabs = computed(() =>
  routes.map((entry) => {
    let badge = null;
    if (entry.id === "quests" && metricsAvailable.value) badge = openQuests.value;
    if (entry.id === "governance" && metricsAvailable.value)
      badge = openProposals.value;
    return { ...entry, badge };
  }),
);

const walletLabel = computed(() =>
  connected.value ? short(wallet.value) : "Connect wallet",
);

const previewNote = computed(() => {
  if (configured.value)
    return "The community program is not initialised on this cluster. Sample quests and proposals stay on screen, and actions submit nothing.";
  if (mintConfigured.value)
    return "Mint data is read-only and sample program rows remain until a program is configured. Actions submit nothing.";
  return "No token mint is configured and the community program is not connected. Sample rows are shown; actions submit nothing.";
});

const actionDisabled = computed(
  () =>
    busy.value ||
    connecting.value ||
    (configured.value &&
      (!ready.value ||
        reading.value ||
        paused.value ||
        (connected.value && (!accountReady.value || accountReading.value)))),
);

const tokenIdentity = computed(() => {
  if (!mintConfigured.value) return "No token mint configured";
  if (tokenName.value) return `${tokenName.value} (${tokenSymbol.value})`;
  return reading.value
    ? "Loading mint data…"
    : `SPL token (${tokenSymbol.value}); metadata pending`;
});

function pillTone(status) {
  const key = String(status || "").toLowerCase();
  if (key === "open") return "is-open";
  if (key === "sample") return "is-sample";
  if (key === "paused" || key === "expired" || key === "filled") return "is-warn";
  if (key === "claimed" || key === "voted") return "is-info";
  return "is-neutral";
}

function questProgress(q) {
  if (!q || q.sample) return 0;
  const cap = Number(q.maxClaims) || 0;
  if (!cap) return 0;
  return Math.max(0, Math.min(100, Math.round((Number(q.claims) / cap) * 100)));
}

function voteBreakdown(proposal) {
  const forVotes = Number(proposal?.forVotes ?? 0);
  const againstVotes = Number(proposal?.againstVotes ?? 0);
  const total = forVotes + againstVotes;
  if (!total) return { forPct: 0, againstPct: 0, empty: true };
  const forPct = Math.round((forVotes / total) * 100);
  return { forPct, againstPct: 100 - forPct, empty: false };
}

/* --------------------------------------------------------------- quest view */

const search = ref("");
const questFilter = ref("all");
const questFilters = computed(() =>
  programReady.value ? ["all", "open", "claimed"] : ["all", "sample"],
);
const visibleQuests = computed(() =>
  allQuests.value.filter(
    (q) =>
      (questFilter.value === "all" ||
        questStatus(q).toLowerCase() === questFilter.value) &&
      q.title.toLowerCase().includes(search.value.trim().toLowerCase()),
  ),
);

function resetFilters() {
  search.value = "";
  questFilter.value = "all";
}

async function openQuest(q) {
  resetFilters();
  await go("quests");
  const row = document.getElementById(`quest-row-${q.id}`);
  row?.focus({ preventScroll: true });
  row?.scrollIntoView({
    block: "center",
    behavior: matchMedia("(prefers-reduced-motion: reduce)").matches
      ? "instant"
      : "smooth",
  });
}

function preview() {
  notify(
    "This is an example row. Explore the workflow here; transactions require a configured Solana program.",
    "info",
  );
}

async function claim(q) {
  if (q.sample) return preview();
  if (questStatus(q) !== "Open")
    return notify("This quest is not available to record.");
  await recordQuest(q);
}

async function setQuest(q) {
  await changeQuestActive(q, !q.active);
}

/* ------------------------------------------------------------- quest admin */

const adminTitle = ref("");
const adminDetails = ref("");
const adminMax = ref(100);
const adminDays = ref(30);

async function createQuest() {
  const title = adminTitle.value.trim();
  const details = adminDetails.value.trim();
  if (!title || new TextEncoder().encode(title).length > 80)
    return notify("Use a quest title between 1 and 80 UTF-8 bytes.");
  if (new TextEncoder().encode(details).length > 240)
    return notify("Quest details must be 240 UTF-8 bytes or fewer.");
  if (
    !Number.isInteger(Number(adminMax.value)) ||
    adminMax.value < 1 ||
    adminMax.value > 1000000
  )
    return notify("Maximum rows must be 1 to 1,000,000.");
  if (
    !Number.isInteger(Number(adminDays.value)) ||
    adminDays.value < 1 ||
    adminDays.value > 365
  )
    return notify("Expiry must be 1 to 365 whole days from today.");
  const expiry = Math.floor(Date.now() / 1000) + Number(adminDays.value) * 86400;
  if (
    await publishQuest({
      title,
      details,
      maxClaims: Number(adminMax.value),
      expiresAt: expiry,
    })
  ) {
    adminTitle.value = "";
    adminDetails.value = "";
  }
}

/* ---------------------------------------------------------------- vote view */

const proposalText = ref("");
const proposalDays = ref(7);
const composeOpen = ref(false);
const draftSaved = ref(false);
const clearedDraft = ref(null);
watch(proposalText, (text) => {
  if (text) clearedDraft.value = null;
});

const draftKey = "orasol-proposal-draft";
const legacyDraftKeys = [
  "konvoxa-proposal-draft",
  "solstir-proposal-draft",
  "solvyrra-proposal-draft",
  "oncivra-proposal-draft",
  "pactelora-proposal-draft",
  "solkintra-proposal-draft",
  "quorivana-proposal-draft",
  "ralliva-proposal-draft",
];
try {
  const savedDraft = sessionStorage.getItem(draftKey);
  const legacyDraft = savedDraft
    ? null
    : legacyDraftKeys.map((key) => sessionStorage.getItem(key)).find(Boolean);
  const draft = JSON.parse(savedDraft || legacyDraft || "null");
  if (draft && typeof draft.text === "string") {
    proposalText.value = draft.text.slice(0, 2000);
    proposalDays.value =
      Number.isInteger(draft.days) && draft.days >= 1 && draft.days <= 30
        ? draft.days
        : 7;
    draftSaved.value = Boolean(proposalText.value.trim());
    if (legacyDraft)
      sessionStorage.setItem(
        draftKey,
        JSON.stringify({ text: proposalText.value, days: proposalDays.value }),
      );
    legacyDraftKeys.forEach((key) => sessionStorage.removeItem(key));
  }
} catch {
  /* Draft persistence is optional when browser storage is unavailable. */
}

watch([proposalText, proposalDays], ([text, days]) => {
  try {
    if (text.trim())
      sessionStorage.setItem(
        draftKey,
        JSON.stringify({ text, days: Number(days) }),
      );
    else sessionStorage.removeItem(draftKey);
    draftSaved.value = Boolean(text.trim());
  } catch {
    draftSaved.value = false;
  }
});

async function toggleCompose() {
  composeOpen.value = !composeOpen.value;
  await nextTick();
  document
    .getElementById(composeOpen.value ? "proposal-description" : "compose-toggle")
    ?.focus();
}

function clearDraft() {
  clearedDraft.value = { text: proposalText.value, days: proposalDays.value };
  proposalText.value = "";
  proposalDays.value = 7;
  document.getElementById("proposal-description")?.focus();
}

function undoClearDraft() {
  if (!clearedDraft.value || proposalText.value) return;
  const draft = clearedDraft.value;
  clearedDraft.value = null;
  proposalText.value = draft.text;
  proposalDays.value = draft.days;
  document.getElementById("proposal-description")?.focus();
}

async function createProposal() {
  const description = proposalText.value.trim();
  if (description.length < 10 || description.length > 2000)
    return notify("Use 10 to 2,000 characters for a proposal.");
  if (
    !Number.isInteger(Number(proposalDays.value)) ||
    proposalDays.value < 1 ||
    proposalDays.value > 30
  )
    return notify("Voting duration must be 1 to 30 whole days.");
  const submittedDays = Number(proposalDays.value);
  if (await publishProposal(description, submittedDays)) {
    if (
      proposalText.value.trim() === description &&
      Number(proposalDays.value) === submittedDays
    ) {
      proposalText.value = "";
      composeOpen.value = false;
      await nextTick();
      document.getElementById("compose-toggle")?.focus();
    }
  }
}

async function vote(p, support) {
  if (p.sample) return preview();
  if (proposalStatus(p) !== "Open")
    return notify("Voting is closed or this wallet already voted.");
  await castVote(p, support);
}
</script>

<template>
  <div class="site">
    <a class="skip-link" href="#main" @click.prevent="focusMain">Skip to content</a>

    <SiteHeader
      :tabs="navTabs"
      :active="route"
      :wallet-label="walletLabel"
      :connected="connected"
      :reading="reading"
      :busy="busy"
      :connecting="connecting"
      :cluster="SOLANA_CLUSTER"
      :menu-open="menuOpen"
      @navigate="navigate"
      @sync="refresh"
      @wallet="connected ? disconnectWallet() : connect()"
      @menu="toggleMenu"
    />

    <template v-if="menuOpen">
      <div class="drawer-backdrop" @click="menuOpen = false"></div>
      <aside class="drawer" aria-label="Sections and wallet">
        <header class="drawer-head">
          <span class="drawer-title">ORAsol</span>
          <button class="icon-btn" type="button" aria-label="Close menu" @click="menuOpen = false">
            <Icon name="close" :size="18" />
          </button>
        </header>
        <nav class="drawer-nav" aria-label="Sections">
          <button
            v-for="tab in navTabs"
            :key="tab.id"
            type="button"
            :class="{ 'is-active': route === tab.id }"
            :aria-current="route === tab.id ? 'page' : undefined"
            @click="navigate(tab.id)"
          >
            <Icon :name="tab.icon" :size="18" />
            <span>{{ tab.label }}</span>
            <b v-if="tab.badge !== null && tab.badge !== undefined">{{ tab.badge }}</b>
          </button>
        </nav>
        <div class="drawer-foot">
          <button
            class="btn btn-primary btn-block"
            type="button"
            :disabled="connecting || busy"
            @click="connected ? disconnectWallet() : connect()"
          >
            <Icon name="wallet" :size="16" />
            {{ connected ? "Disconnect wallet" : "Connect wallet" }}
          </button>
          <div class="drawer-facts">
            <span>Cluster <b>{{ SOLANA_CLUSTER }}</b></span>
            <span>Program <b>{{ programState }}</b></span>
            <span>Wallet <b>{{ walletLabel }}</b></span>
          </div>
        </div>
      </aside>
    </template>

    <main id="main" class="page" tabindex="-1">
      <div class="page-inner">
        <div v-if="!programReady" class="banner is-warn">
          <Icon name="info" :size="18" />
          <div class="banner-copy">
            <strong>Preview rows</strong>
            <p>{{ previewNote }}</p>
          </div>
          <button class="btn btn-quiet" type="button" @click="go('learn')">
            How it works <Icon name="arrow-right" :size="14" />
          </button>
        </div>

        <div v-if="paused" class="banner is-warn" role="status">
          <Icon name="alert" :size="18" />
          <div class="banner-copy">
            <strong>Writes are paused</strong>
            <p>The program is paused. Reads continue; transactions are unavailable.</p>
          </div>
        </div>

        <div v-if="readError" class="banner is-danger" role="alert">
          <Icon name="alert" :size="18" />
          <div class="banner-copy">
            <strong>Sync failed</strong>
            <p>{{ readError }}</p>
          </div>
          <button class="btn btn-quiet" type="button" :disabled="reading || busy" @click="refresh">
            Try again
          </button>
        </div>

        <div v-if="accountError" class="banner is-danger" role="alert">
          <Icon name="alert" :size="18" />
          <div class="banner-copy">
            <strong>Wallet read failed</strong>
            <p>{{ accountError }}</p>
          </div>
        </div>

        <div v-if="accountReading" class="banner is-info" role="status">
          <Icon name="clock" :size="18" />
          <div class="banner-copy">
            <strong>Reading wallet balances…</strong>
          </div>
        </div>

        <Transition name="page" mode="out-in">
          <section :key="route" class="view">
            <!-- ================================================== OVERVIEW == -->
            <template v-if="route === 'home'">
              <section class="hero" aria-labelledby="hero-title">
                <div class="hero-copy">
                  <p class="kicker">
                    <span class="kicker-dot" aria-hidden="true"></span>
                    {{ SOLANA_NETWORK_NAME }} · open record
                  </p>
                  <h1 id="hero-title" class="hero-title">
                    Time on the<br /><em>record.</em>
                  </h1>
                  <p class="hero-lede">
                    ORAsol writes down who took part, how the room voted and where
                    the money sits — a public register, stamped hour by hour, that
                    anyone can reopen. No private minutes, no unlisted spreadsheet.
                  </p>
                  <div class="hero-actions">
                    <button class="btn btn-primary btn-lg" type="button" @click="go('quests')">
                      Browse quests <Icon name="arrow-right" :size="16" />
                    </button>
                    <button class="btn btn-outline btn-lg" type="button" @click="go('governance')">
                      <Icon name="ballot" :size="16" /> Open votes
                    </button>
                  </div>
                  <ul class="hero-facts">
                    <li><Icon name="shield" :size="15" /><span>No token mint exists</span></li>
                    <li><Icon name="users" :size="15" /><span>One row per wallet</span></li>
                    <li><Icon name="clock" :size="15" /><span>Reads refresh on demand</span></li>
                  </ul>
                </div>

                <aside class="panel" aria-label="Live readouts">
                  <div class="panel-head">
                    <h2 class="panel-title">On the record today</h2>
                    <span class="panel-date">{{ todayLabel }}</span>
                  </div>
                  <div class="metric-grid">
                    <div class="metric">
                      <span class="metric-label">Open quests</span>
                      <strong class="metric-value">{{ metricsAvailable ? openQuests : "—" }}</strong>
                      <span class="metric-note">
                        {{ metricsAvailable ? "read from the program" : "sample rows shown" }}
                      </span>
                    </div>
                    <div class="metric">
                      <span class="metric-label">Votes open</span>
                      <strong class="metric-value">{{ metricsAvailable ? openProposals : "—" }}</strong>
                      <span class="metric-note">advisory · one vote per wallet</span>
                    </div>
                    <div class="metric">
                      <span class="metric-label">Your wallet</span>
                      <strong class="metric-value is-text">
                        {{ connected ? short(wallet) : "—" }}
                      </strong>
                      <span class="metric-note">
                        {{
                          connected && accountReady
                            ? solUnits(native, 4) + " SOL"
                            : connected
                              ? "reading…"
                              : "not connected"
                        }}
                      </span>
                    </div>
                    <div class="metric">
                      <span class="metric-label">Token</span>
                      <strong class="metric-value is-text">
                        {{ mintConfigured ? tokenSymbol || "mint set" : "None yet" }}
                      </strong>
                      <span class="metric-note">
                        {{ mintConfigured ? short(TOKEN_MINT_ADDRESS) : "nothing to buy or hold" }}
                      </span>
                    </div>
                  </div>
                  <div class="panel-foot">
                    <span class="dot-live" :class="{ 'is-idle': !programReady }" aria-hidden="true"></span>
                    <span>{{ programReady ? "Program reports live" : "Preview mode · nothing is submitted" }}</span>
                    <button class="link-btn" type="button" @click="go('learn')">
                      Why? <Icon name="arrow-right" :size="14" />
                    </button>
                  </div>
                </aside>
              </section>

              <section class="tile-grid" aria-label="Ways to take part">
                <article class="tile">
                  <span class="tile-icon"><Icon name="flag" :size="22" /></span>
                  <h2>Take part</h2>
                  <p>
                    Pick a quest, record your participation and leave a row anyone can
                    check later. Rows are participation only — they do not pay a token.
                  </p>
                  <div class="tile-foot">
                    <button class="btn btn-outline" type="button" @click="go('quests')">
                      See {{ allQuests.length }} quests <Icon name="arrow-right" :size="15" />
                    </button>
                  </div>
                </article>
                <article class="tile is-dark">
                  <span class="tile-icon"><Icon name="ballot" :size="22" /></span>
                  <h2>Weigh in</h2>
                  <p>
                    Write a proposal, set a window between one and thirty days, and let
                    wallets answer. Results are advisory and never move funds.
                  </p>
                  <div class="tile-foot">
                    <button class="btn btn-light" type="button" @click="go('governance')">
                      Open the floor <Icon name="arrow-right" :size="15" />
                    </button>
                  </div>
                </article>
              </section>

              <div class="split section-gap">
                <div class="col">
                  <section class="card">
                    <header class="card-head">
                      <h2>Newest quests</h2>
                      <button class="link-btn" type="button" @click="go('quests')">
                        All quests <Icon name="arrow-right" :size="14" />
                      </button>
                    </header>
                    <ul v-if="registerQuests.length" class="row-list">
                      <li v-for="(q, index) in registerQuests" :key="q.id">
                        <button class="row-item" type="button" @click="openQuest(q)">
                          <span class="row-index">{{ String(index + 1).padStart(2, "0") }}</span>
                          <span class="row-copy">
                            <b>{{ q.title }}</b>
                            <small>
                              {{
                                q.sample
                                  ? "Sample row · preview only"
                                  : "Quest / " + String(q.id).padStart(3, "0")
                              }}
                            </small>
                          </span>
                          <span class="pill" :class="pillTone(questStatus(q))">{{ questStatus(q) }}</span>
                          <span class="row-chevron"><Icon name="arrow-right" :size="16" /></span>
                        </button>
                      </li>
                    </ul>
                    <div v-else class="card-body">
                      <div class="blank">
                        <span class="blank-icon"><Icon name="flag" :size="20" /></span>
                        <h3>No rows yet</h3>
                        <p>Quests appear here once the community publishes them.</p>
                      </div>
                    </div>
                  </section>

                  <section class="card">
                    <header class="card-head">
                      <h2>Votes on the floor</h2>
                      <button class="link-btn" type="button" @click="go('governance')">
                        All votes <Icon name="arrow-right" :size="14" />
                      </button>
                    </header>
                    <ul v-if="registerProposals.length" class="row-list">
                      <li v-for="(p, index) in registerProposals" :key="p.id">
                        <button class="row-item" type="button" @click="go('governance')">
                          <span class="row-index">{{ String(index + 1).padStart(2, "0") }}</span>
                          <span class="row-copy">
                            <b>{{ p.description }}</b>
                            <small>
                              {{ p.sample ? "Sample proposal · no live votes" : "Closes " + date(p.endsAt) }}
                            </small>
                          </span>
                          <span class="pill" :class="pillTone(proposalStatus(p))">{{ proposalStatus(p) }}</span>
                          <span class="row-chevron"><Icon name="arrow-right" :size="16" /></span>
                        </button>
                      </li>
                    </ul>
                    <div v-else class="card-body">
                      <div class="blank">
                        <span class="blank-icon"><Icon name="ballot" :size="20" /></span>
                        <h3>The floor is open</h3>
                        <p>Start the first proposal when the community has a decision to make.</p>
                      </div>
                    </div>
                    <div v-if="registerProposals.length" class="card-body">
                      <span class="card-note">Support on the leading proposal</span>
                      <div class="tally-bar">
                        <i class="is-for" :style="{ width: leadBreakdown.forPct + '%' }"></i>
                        <i class="is-against" :style="{ width: leadBreakdown.againstPct + '%' }"></i>
                      </div>
                      <div class="tally-legend">
                        <span class="key">
                          For <b>{{ units(leadProposal?.forVotes || 0n) }}</b>
                        </span>
                        <span class="key is-against">
                          Against <b>{{ units(leadProposal?.againstVotes || 0n) }}</b>
                        </span>
                        <span>
                          Support <b>{{ leadSupport === null ? "—" : leadSupport + "%" }}</b>
                        </span>
                      </div>
                    </div>
                  </section>
                </div>

                <aside class="aside">
                  <section class="card">
                    <header class="card-head">
                      <h2>Your wallet</h2>
                      <span class="card-note">{{ connected ? "connected" : "idle" }}</span>
                    </header>
                    <div class="card-body is-flush">
                      <dl class="defs">
                        <div>
                          <dt>Address</dt>
                          <dd>{{ connected ? short(wallet) : "not connected" }}</dd>
                        </div>
                        <div>
                          <dt>SOL</dt>
                          <dd>{{ connected && accountReady ? solUnits(native, 4) : "—" }}</dd>
                        </div>
                        <div>
                          <dt>{{ mintConfigured ? tokenSymbol : "Token" }}</dt>
                          <dd class="is-muted">
                            {{ mintConfigured && connected && accountReady ? units(token) : "not configured" }}
                          </dd>
                        </div>
                        <div>
                          <dt>State</dt>
                          <dd>
                            {{ connected ? (accountReady ? "synced" : "reading") : "connect to read" }}
                          </dd>
                        </div>
                      </dl>
                    </div>
                    <div class="card-foot">
                      <button
                        class="btn btn-primary btn-block"
                        type="button"
                        :disabled="connecting || busy"
                        @click="connected ? disconnectWallet() : connect()"
                      >
                        <Icon name="wallet" :size="15" />
                        {{ connected ? "Disconnect wallet" : "Connect wallet" }}
                      </button>
                    </div>
                  </section>

                  <section class="card">
                    <header class="card-head">
                      <h2>Token</h2>
                      <span class="pill is-neutral">{{ mintConfigured ? "configured" : "none yet" }}</span>
                    </header>
                    <div class="card-body">
                      <p>{{ tokenIdentity }}</p>
                      <p>
                        The community program records participation and advisory votes.
                        It never mints, holds or pays out a token.
                      </p>
                      <button class="btn btn-outline btn-block" type="button" @click="go('stake')">
                        Token status <Icon name="arrow-right" :size="14" />
                      </button>
                    </div>
                  </section>

                  <div class="note">
                    <Icon name="shield" :size="18" />
                    <div>
                      <strong>Before you sign</strong>
                      <p>Your wallet reviews every instruction. A preview row never sends a transaction.</p>
                    </div>
                  </div>
                </aside>
              </div>
            </template>

            <!-- ==================================================== QUESTS == -->
            <template v-if="route === 'quests'">
              <header class="page-head">
                <div>
                  <p class="kicker">
                    <span class="kicker-dot" aria-hidden="true"></span>Participation
                  </p>
                  <h1 class="page-title">Find your <em>part in it.</em></h1>
                  <p class="page-lede">
                    Each quest is an open invitation to take part. Recording one writes a
                    single row for your wallet — it does not verify off-chain work, prove
                    a unique person or pay a token.
                  </p>
                </div>
                <div class="page-meta">
                  <span class="meta-chip is-ink">{{ allQuests.length }} rows</span>
                  <span class="meta-chip">{{ programReady ? "program live" : "preview" }}</span>
                </div>
              </header>

              <div class="toolbar">
                <label class="search">
                  <Icon name="search" :size="17" />
                  <span class="sr-only">Search quests</span>
                  <input v-model="search" type="search" placeholder="Search quests…" autocomplete="off" />
                </label>
                <div class="chip-row" role="group" aria-label="Filter quests">
                  <button
                    v-for="f in questFilters"
                    :key="f"
                    class="chip"
                    :class="{ 'is-active': questFilter === f }"
                    :aria-pressed="questFilter === f"
                    type="button"
                    @click="questFilter = f"
                  >
                    {{ f }}
                  </button>
                </div>
                <span class="result-count" role="status">
                  {{ visibleQuests.length }} {{ visibleQuests.length === 1 ? "quest" : "quests" }}
                </span>
              </div>

              <div
                v-if="reading && !allQuests.length"
                class="card-grid"
                role="status"
                aria-label="Loading quests"
              >
                <div v-for="i in 3" :key="i" class="skeleton"></div>
              </div>

              <div v-else-if="!visibleQuests.length" class="blank">
                <span class="blank-icon"><Icon name="flag" :size="20" /></span>
                <h3>
                  {{
                    readError
                      ? "Rows unavailable"
                      : search || questFilter !== "all"
                        ? "No matching quests"
                        : "Room for the first row"
                  }}
                </h3>
                <p>
                  {{
                    readError
                      ? "Refresh the connection to read the latest quests."
                      : search || questFilter !== "all"
                        ? "Try another keyword or show every quest."
                        : "No quests have been published yet. Check back for the next activity."
                  }}
                </p>
                <button
                  v-if="search || questFilter !== 'all'"
                  class="btn btn-outline"
                  type="button"
                  @click="resetFilters"
                >
                  Clear filters
                </button>
                <button
                  v-else-if="readError"
                  class="btn btn-primary"
                  type="button"
                  :disabled="reading"
                  @click="refresh"
                >
                  Try again
                </button>
              </div>

              <div v-else class="card-grid">
                <article
                  v-for="(q, index) in visibleQuests"
                  :id="'quest-row-' + q.id"
                  :key="q.id"
                  class="quest-card"
                  tabindex="-1"
                >
                  <div class="quest-top">
                    <span class="pill" :class="pillTone(questStatus(q))">{{ questStatus(q) }}</span>
                    <span class="quest-id">
                      {{ q.sample ? "sample" : "quest " + String(index + 1).padStart(2, "0") }}
                    </span>
                  </div>
                  <h3 class="quest-title">{{ q.title }}</h3>
                  <p class="quest-text">{{ q.detail || "Community participation record." }}</p>
                  <div class="quest-meta">
                    <div class="quest-meta-row">
                      <span>Capacity</span>
                      <b>{{ q.sample ? "preview only" : q.claims + " / " + q.maxClaims }}</b>
                    </div>
                    <div class="meter"><i :style="{ width: questProgress(q) + '%' }"></i></div>
                    <div class="quest-meta-row">
                      <span>Expires</span>
                      <b>{{ q.sample ? "—" : date(q.expiresAt) }}</b>
                    </div>
                  </div>
                  <div class="quest-foot">
                    <button
                      class="btn btn-primary btn-sm"
                      type="button"
                      :disabled="actionDisabled || (!q.sample && questStatus(q) !== 'Open')"
                      @click="claim(q)"
                    >
                      {{
                        q.sample
                          ? "Preview row"
                          : q.claimed
                            ? "Recorded"
                            : questStatus(q) !== "Open"
                              ? questStatus(q)
                              : connected
                                ? "Record my part"
                                : "Connect to record"
                      }}
                    </button>
                    <button
                      v-if="isOwner && !q.sample"
                      class="btn btn-outline btn-sm"
                      type="button"
                      :disabled="busy"
                      @click="setQuest(q)"
                    >
                      {{ q.active ? "Pause" : "Resume" }}
                    </button>
                  </div>
                </article>
              </div>

              <form v-if="isOwner" class="studio" @submit.prevent="createQuest">
                <div class="studio-head">
                  <h2>Publish a quest</h2>
                  <span class="meta-chip">authority wallet</span>
                </div>
                <div class="form-grid two">
                  <label class="field">
                    <span>Quest title</span>
                    <input v-model="adminTitle" maxlength="80" required placeholder="Welcome to the community" />
                  </label>
                  <label class="field">
                    <span>Details</span>
                    <input v-model="adminDetails" maxlength="240" placeholder="What should participants review?" />
                  </label>
                  <label class="field">
                    <span>Maximum rows</span>
                    <input v-model="adminMax" type="number" min="1" max="1000000" required />
                  </label>
                  <label class="field">
                    <span>Open for (days)</span>
                    <input v-model="adminDays" type="number" min="1" max="365" required />
                  </label>
                </div>
                <div class="form-actions">
                  <p class="form-note">Rows record participation only. No token reward is attached.</p>
                  <button class="btn btn-primary" type="submit" :disabled="actionDisabled">
                    <Icon name="plus" :size="15" /> Publish quest
                  </button>
                </div>
              </form>

              <div class="note">
                <Icon name="shield" :size="18" />
                <div>
                  <strong>Your wallet, your decision</strong>
                  <p>Review the quest and the network before confirming. A preview never sends a transaction.</p>
                </div>
              </div>
            </template>

            <!-- ===================================================== VOTES == -->
            <template v-if="route === 'governance'">
              <header class="page-head">
                <div>
                  <p class="kicker">
                    <span class="kicker-dot" aria-hidden="true"></span>Advisory votes
                  </p>
                  <h1 class="page-title">Put it <em>to the room.</em></h1>
                  <p class="page-lede">
                    Write a proposal, choose a window of one to thirty days, and let
                    wallets answer. One vote per wallet; results are advisory and never
                    move funds.
                  </p>
                </div>
                <div class="page-meta">
                  <span class="meta-chip is-ink">{{ visibleProposals.length }} proposals</span>
                  <span class="meta-chip">{{ programReady ? "on chain" : "sample rows" }}</span>
                </div>
              </header>

              <div class="split">
                <div class="col">
                  <template v-if="visibleProposals.length">
                    <article v-for="p in visibleProposals" :key="p.id" class="ballot">
                      <div class="ballot-head">
                        <span class="ballot-id">
                          {{ p.sample ? "sample proposal" : "proposal " + String(p.id).padStart(3, "0") }}
                        </span>
                        <span class="pill" :class="pillTone(proposalStatus(p))">{{ proposalStatus(p) }}</span>
                      </div>
                      <h3 class="ballot-title">{{ p.description }}</h3>
                      <div class="tally-bar">
                        <i class="is-for" :style="{ width: voteBreakdown(p).forPct + '%' }"></i>
                        <i class="is-against" :style="{ width: voteBreakdown(p).againstPct + '%' }"></i>
                      </div>
                      <div class="tally-legend">
                        <span class="key">For <b>{{ units(p.forVotes) }}</b></span>
                        <span class="key is-against">Against <b>{{ units(p.againstVotes) }}</b></span>
                        <span>
                          {{ voteBreakdown(p).empty ? "No votes yet" : votePercent(p) + "% support" }}
                        </span>
                        <span v-if="!p.sample">Closes {{ date(p.endsAt) }}</span>
                      </div>
                      <div class="ballot-foot">
                        <p>
                          {{
                            p.voted
                              ? "Your vote is on the record."
                              : voteBreakdown(p).empty
                                ? "Waiting for the first vote."
                                : "Advisory only — nothing is executed."
                          }}
                        </p>
                        <button
                          class="btn btn-primary btn-sm"
                          type="button"
                          :disabled="actionDisabled || (!p.sample && proposalStatus(p) !== 'Open')"
                          @click="vote(p, true)"
                        >
                          <Icon name="check" :size="14" /> Vote for
                        </button>
                        <button
                          class="btn btn-outline btn-sm"
                          type="button"
                          :disabled="actionDisabled || (!p.sample && proposalStatus(p) !== 'Open')"
                          @click="vote(p, false)"
                        >
                          <Icon name="close" :size="14" /> Against
                        </button>
                      </div>
                    </article>
                  </template>

                  <div v-else-if="programReady && reading" class="skeleton" role="status">
                    <span class="sr-only">Loading proposals</span>
                  </div>

                  <div v-else class="blank">
                    <span class="blank-icon"><Icon name="ballot" :size="20" /></span>
                    <h3>{{ readError ? "Proposals unavailable" : "The floor is open" }}</h3>
                    <p>
                      {{
                        readError
                          ? "Refresh the connection to reconnect."
                          : "Start the first proposal with one clear decision and a voting window."
                      }}
                    </p>
                    <button
                      v-if="!readError && !composeOpen"
                      class="btn btn-primary"
                      type="button"
                      @click="toggleCompose"
                    >
                      Write the first proposal
                    </button>
                  </div>

                  <div class="note">
                    <Icon name="info" :size="18" />
                    <div>
                      <strong>Advisory, not binding</strong>
                      <p>Votes do not execute treasury spending. Weight follows the program's own rules.</p>
                    </div>
                  </div>
                </div>

                <aside class="aside">
                  <section class="card">
                    <header class="card-head">
                      <h2>Start a proposal</h2>
                      <span class="card-note">
                        {{ draftSaved && !composeOpen ? "draft saved" : "10–2,000 chars" }}
                      </span>
                    </header>
                    <div class="card-body">
                      <button
                        id="compose-toggle"
                        class="btn btn-outline btn-block"
                        type="button"
                        :aria-expanded="composeOpen"
                        aria-controls="compose-editor"
                        @click="toggleCompose"
                      >
                        <Icon :name="composeOpen ? 'close' : 'plus'" :size="15" />
                        {{
                          composeOpen
                            ? "Close editor"
                            : draftSaved
                              ? "Continue your draft"
                              : "New proposal"
                        }}
                      </button>
                    </div>
                    <form
                      v-if="composeOpen"
                      id="compose-editor"
                      class="card-body"
                      @submit.prevent="createProposal"
                      @keydown.esc.prevent="toggleCompose"
                    >
                      <label class="field">
                        <span>Proposal</span>
                        <textarea
                          id="proposal-description"
                          v-model="proposalText"
                          minlength="10"
                          maxlength="2000"
                          required
                          placeholder="One decision, its purpose, any budget, and the outcome you expect."
                        ></textarea>
                      </label>
                      <label class="field">
                        <span>Voting window (days)</span>
                        <input v-model="proposalDays" type="number" min="1" max="30" required />
                      </label>
                      <div class="form-actions">
                        <span class="form-note">{{ proposalText.length }} / 2,000</span>
                        <button
                          class="btn btn-primary"
                          type="submit"
                          :disabled="actionDisabled || proposalText.trim().length < 10"
                        >
                          {{ programReady ? "Publish" : "Preview" }}
                        </button>
                      </div>
                      <div class="form-actions">
                        <button class="btn btn-quiet btn-sm" type="button" @click="toggleCompose">
                          Keep draft
                        </button>
                        <button
                          v-if="proposalText"
                          class="btn btn-quiet btn-sm"
                          type="button"
                          @click="clearDraft"
                        >
                          Clear
                        </button>
                        <button
                          v-else-if="clearedDraft"
                          class="btn btn-quiet btn-sm"
                          type="button"
                          @click="undoClearDraft"
                        >
                          Undo clear
                        </button>
                      </div>
                    </form>
                  </section>

                  <section class="card">
                    <header class="card-head">
                      <h2>What lands</h2>
                      <span class="card-note">three moves</span>
                    </header>
                    <div class="card-body is-flush">
                      <ol class="timeline">
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">1</span>
                          <div class="tl-body">
                            <h3>One decision</h3>
                            <p>State a single decision and why it matters now.</p>
                          </div>
                        </li>
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">2</span>
                          <div class="tl-body">
                            <h3>Context and cost</h3>
                            <p>Name the outcome and any budget the room should expect.</p>
                          </div>
                        </li>
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">3</span>
                          <div class="tl-body">
                            <h3>Time to answer</h3>
                            <p>Give wallets 1–30 days so the vote is not a reflex.</p>
                          </div>
                        </li>
                      </ol>
                    </div>
                  </section>
                </aside>
              </div>
            </template>

            <!-- =================================================== TREASURY == -->
            <template v-if="route === 'treasury'">
              <header class="page-head">
                <div>
                  <p class="kicker">
                    <span class="kicker-dot" aria-hidden="true"></span>Public reads
                  </p>
                  <h1 class="page-title">Nothing hidden <em>in the books.</em></h1>
                  <p class="page-lede">
                    Treasury balances and account rows are read straight from Solana.
                    When a read is unavailable, this page says so instead of guessing.
                  </p>
                </div>
                <div class="page-meta">
                  <span class="meta-chip is-ink">{{ SOLANA_NETWORK_NAME }}</span>
                </div>
              </header>

              <section class="stat-grid" aria-label="Treasury reads">
                <div class="stat">
                  <span class="stat-label">Treasury token</span>
                  <b class="stat-value">{{ ready && treasuryTokenKnown ? units(treasuryBalance) : "—" }}</b>
                  <span class="stat-note">{{ tokenIdentity }}</span>
                </div>
                <div class="stat">
                  <span class="stat-label">Treasury SOL</span>
                  <b class="stat-value">
                    {{ ready && treasuryNativeKnown ? solUnits(treasuryNative, 4) : "—" }}
                  </b>
                  <span class="stat-note">native balance for fees</span>
                </div>
                <div class="stat">
                  <span class="stat-label">Issued supply</span>
                  <b class="stat-value">{{ mintConfigured && ready ? units(totalSupply) : "—" }}</b>
                  <span class="stat-note">read from the mint</span>
                </div>
                <div class="stat">
                  <span class="stat-label">Activity index</span>
                  <b class="stat-value">None</b>
                  <span class="stat-note">no indexer connected yet</span>
                </div>
              </section>

              <p
                v-if="treasuryError || (mintConfigured && ready && (!treasuryKnown || !treasuryTokenKnown))"
                class="form-note"
                role="status"
              >
                {{ treasuryError || "Treasury token or SOL balances are not fully available yet." }}
              </p>

              <div class="split">
                <div class="col">
                  <section class="card">
                    <header class="card-head">
                      <h2>Account statement</h2>
                      <span class="card-note">{{ ready ? "read at " + lastRead : "awaiting read" }}</span>
                    </header>
                    <div class="receipt">
                      <div class="receipt-row">
                        <span class="receipt-key">On-chain token</span>
                        <span class="receipt-leader" aria-hidden="true"></span>
                        <span class="receipt-val is-muted">{{ tokenIdentity }}</span>
                      </div>
                      <div class="receipt-row">
                        <span class="receipt-key">
                          {{ mintConfigured ? "Total issued " + tokenSymbol : "Issued supply" }}
                        </span>
                        <span class="receipt-leader" aria-hidden="true"></span>
                        <span class="receipt-val">
                          {{ mintConfigured && ready ? units(totalSupply) : "—" }}
                        </span>
                      </div>
                      <div class="receipt-row">
                        <span class="receipt-key">Token staking</span>
                        <span class="receipt-leader" aria-hidden="true"></span>
                        <span class="receipt-val is-muted">not implemented</span>
                      </div>
                      <div class="receipt-row">
                        <span class="receipt-key">
                          {{ mintConfigured ? "Your " + tokenSymbol : "Your token balance" }}
                        </span>
                        <span class="receipt-leader" aria-hidden="true"></span>
                        <span class="receipt-val">
                          {{
                            connected && accountReady
                              ? units(token)
                              : connected
                                ? "reading"
                                : "connect wallet"
                          }}
                        </span>
                      </div>
                      <div class="receipt-row">
                        <span class="receipt-key">Decimals</span>
                        <span class="receipt-leader" aria-hidden="true"></span>
                        <span class="receipt-val is-muted">{{ mintConfigured ? tokenDecimals : "—" }}</span>
                      </div>
                    </div>
                  </section>

                  <section class="card">
                    <header class="card-head">
                      <h2>Activity</h2>
                      <span class="card-note">not connected</span>
                    </header>
                    <div class="card-body">
                      <div class="blank">
                        <span class="blank-icon"><Icon name="clock" :size="20" /></span>
                        <h3>
                          {{
                            reading
                              ? "Reading the chain…"
                              : readError
                                ? "Activity could not load"
                                : "No indexer yet"
                          }}
                        </h3>
                        <p>
                          A Solana activity index will be connected after the program is
                          deployed for production use.
                        </p>
                      </div>
                    </div>
                  </section>
                </div>

                <aside class="aside">
                  <section class="card">
                    <header class="card-head">
                      <h2>Addresses</h2>
                    </header>
                    <div class="card-body is-flush">
                      <dl class="defs">
                        <div>
                          <dt>Treasury wallet</dt>
                          <dd>
                            <a
                              v-if="treasury"
                              :href="explorer('address', treasury)"
                              target="_blank"
                              rel="noopener noreferrer"
                            >{{ short(treasury) }} <Icon name="arrow-up-right" :size="13" /></a>
                            <span v-else class="is-muted">not available</span>
                          </dd>
                        </div>
                        <div>
                          <dt>Token mint</dt>
                          <dd>
                            <a
                              v-if="mintConfigured"
                              :href="explorer('address', TOKEN_MINT_ADDRESS)"
                              target="_blank"
                              rel="noopener noreferrer"
                            >{{ short(TOKEN_MINT_ADDRESS) }} <Icon name="arrow-up-right" :size="13" /></a>
                            <span v-else class="is-muted">not configured</span>
                          </dd>
                        </div>
                        <div>
                          <dt>Cluster</dt>
                          <dd>{{ SOLANA_CLUSTER }}</dd>
                        </div>
                        <div>
                          <dt>Network</dt>
                          <dd class="is-muted">{{ SOLANA_NETWORK_NAME }}</dd>
                        </div>
                      </dl>
                    </div>
                    <div class="card-body">
                      <p>
                        {{
                          mintConfigured
                            ? "Decimals come from the mint; display metadata stays pending until it is verified."
                            : "No token mint is deployed or configured."
                        }}
                      </p>
                    </div>
                  </section>

                  <div class="note is-warn">
                    <Icon name="alert" :size="18" />
                    <div>
                      <strong>An open record cuts both ways</strong>
                      <p>Anyone can reopen these reads, including people who disagree with the last decision.</p>
                    </div>
                  </div>
                </aside>
              </div>
            </template>

            <!-- ====================================================== TOKEN == -->
            <template v-if="route === 'stake'">
              <header class="page-head">
                <div>
                  <p class="kicker">
                    <span class="kicker-dot" aria-hidden="true"></span>Token details
                  </p>
                  <h1 class="page-title">No token is <em>wired up.</em></h1>
                  <p class="page-lede">
                    Quests and advisory votes run through the Solana program. Staking,
                    locking and rewards stay switched off until a mint and its custody
                    rules exist.
                  </p>
                </div>
                <div class="page-meta">
                  <span class="meta-chip is-ink">not configured</span>
                </div>
              </header>

              <div class="split">
                <div class="col">
                  <section class="card">
                    <header class="card-head">
                      <h2>Token status</h2>
                      <span class="pill is-neutral">not configured</span>
                    </header>
                    <div class="card-body is-flush">
                      <dl class="defs">
                        <div>
                          <dt>On-chain token</dt>
                          <dd class="is-muted">{{ tokenIdentity }}</dd>
                        </div>
                        <div>
                          <dt>Token staking</dt>
                          <dd class="is-muted">not implemented</dd>
                        </div>
                        <div>
                          <dt>Locking</dt>
                          <dd class="is-muted">not implemented</dd>
                        </div>
                        <div>
                          <dt>Reward source</dt>
                          <dd class="is-muted">undefined</dd>
                        </div>
                        <div>
                          <dt>Decimals</dt>
                          <dd class="is-muted">{{ mintConfigured ? tokenDecimals : "—" }}</dd>
                        </div>
                      </dl>
                    </div>
                    <div class="card-body">
                      <p>
                        The program records participation rows and advisory votes. It does
                        not custody tokens, mint rewards or track staking positions.
                      </p>
                    </div>
                  </section>

                  <section class="card">
                    <header class="card-head">
                      <h2>What has to exist first</h2>
                      <span class="card-note">in order</span>
                    </header>
                    <div class="card-body is-flush">
                      <ol class="timeline">
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">1</span>
                          <div class="tl-body">
                            <h3>A verified SPL mint</h3>
                            <p>
                              The mint address, decimals and metadata would be read from
                              Solana before any label appears here.
                            </p>
                          </div>
                        </li>
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">2</span>
                          <div class="tl-body">
                            <h3>Written custody rules</h3>
                            <p>
                              Who can move locked balances, how a holder exits, and what
                              happens if the program pauses.
                            </p>
                          </div>
                        </li>
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">3</span>
                          <div class="tl-body">
                            <h3>A funded reward source</h3>
                            <p>
                              Rewards need money behind them. Until that exists, no reward
                              can be promised in a row.
                            </p>
                          </div>
                        </li>
                      </ol>
                    </div>
                  </section>
                </div>

                <aside class="aside">
                  <section class="card">
                    <header class="card-head">
                      <h2>Wiring</h2>
                    </header>
                    <div class="card-body is-flush">
                      <dl class="defs">
                        <div>
                          <dt>Cluster</dt>
                          <dd>{{ SOLANA_CLUSTER }}</dd>
                        </div>
                        <div>
                          <dt>Network</dt>
                          <dd class="is-muted">{{ SOLANA_NETWORK_NAME }}</dd>
                        </div>
                        <div>
                          <dt>Program</dt>
                          <dd>{{ programState }}</dd>
                        </div>
                        <div>
                          <dt>Mint</dt>
                          <dd class="is-muted">{{ mintState }}</dd>
                        </div>
                      </dl>
                    </div>
                    <div class="card-foot">
                      <button class="btn btn-outline btn-block" type="button" @click="go('learn')">
                        Read the guide <Icon name="arrow-right" :size="14" />
                      </button>
                    </div>
                  </section>

                  <div class="note is-warn">
                    <Icon name="alert" :size="18" />
                    <div>
                      <strong>No mint has been issued</strong>
                      <p>Anything claiming an ORAsol token right now is not from this project.</p>
                    </div>
                  </div>
                </aside>
              </div>
            </template>

            <!-- ====================================================== GUIDE == -->
            <template v-if="route === 'learn'">
              <header class="page-head">
                <div>
                  <p class="kicker">
                    <span class="kicker-dot" aria-hidden="true"></span>The field guide
                  </p>
                  <h1 class="page-title">A way in. <em>A way to check.</em></h1>
                  <p class="page-lede">
                    What each row means, what your wallet will ask you to confirm, and
                    what this project deliberately does not do yet.
                  </p>
                </div>
                <div class="page-meta">
                  <span class="meta-chip is-ink">{{ SOLANA_NETWORK_NAME }}</span>
                </div>
              </header>

              <div class="split">
                <div class="col">
                  <section class="card">
                    <header class="card-head">
                      <h2>Three ways through</h2>
                      <span class="card-note">start anywhere</span>
                    </header>
                    <div class="card-body is-flush">
                      <ol class="timeline">
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">1</span>
                          <div class="tl-body">
                            <h3>Record a participation row</h3>
                            <p>
                              Quests carry an expiry and a row cap. Each wallet records
                              once. Rows do not verify off-chain work, prove a unique
                              person, or pay a token.
                            </p>
                            <div class="tl-actions">
                              <button class="btn btn-outline btn-sm" type="button" @click="go('quests')">
                                See quests <Icon name="arrow-right" :size="14" />
                              </button>
                            </div>
                          </div>
                        </li>
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">2</span>
                          <div class="tl-body">
                            <h3>Wait for real token details</h3>
                            <p>
                              Locking and rewards need a verified mint, custody rules and a
                              funded reward source. None are enabled by the community
                              program.
                            </p>
                            <div class="tl-actions">
                              <button class="btn btn-outline btn-sm" type="button" @click="go('stake')">
                                Token status <Icon name="arrow-right" :size="14" />
                              </button>
                            </div>
                          </div>
                        </li>
                        <li class="tl-item">
                          <span class="tl-num" aria-hidden="true">3</span>
                          <div class="tl-body">
                            <h3>Answer the next question</h3>
                            <p>
                              Create a proposal with a 1–30 day window, or vote on an open
                              one. One vote per wallet. Results are advisory.
                            </p>
                            <div class="tl-actions">
                              <button class="btn btn-outline btn-sm" type="button" @click="go('governance')">
                                Open votes <Icon name="arrow-right" :size="14" />
                              </button>
                            </div>
                          </div>
                        </li>
                      </ol>
                    </div>
                  </section>

                  <section class="card">
                    <header class="card-head">
                      <h2>Good to know</h2>
                    </header>
                    <div class="card-body is-flush">
                      <div class="faq">
                        <details class="faq-item" open>
                          <summary>What does a wallet confirmation do?</summary>
                          <p class="faq-a">
                            Recording a quest row, publishing a proposal and casting a vote
                            use the initialised program on {{ SOLANA_NETWORK_NAME }}. Your
                            wallet shows every instruction before you sign.
                          </p>
                        </details>
                        <details class="faq-item">
                          <summary>Who controls the protocol?</summary>
                          <p class="faq-a">
                            The program authority can publish quests and pause community
                            actions. Advisory votes do not execute treasury spending.
                          </p>
                        </details>
                        <details class="faq-item">
                          <summary>Why is the token name separate from ORAsol?</summary>
                          <p class="faq-a">
                            A future mint defines its own name, symbol and decimals.
                            ORAsol is the application brand. Current status:
                            {{ tokenIdentity }}.
                          </p>
                        </details>
                        <details class="faq-item">
                          <summary>Is an ORAsol token deployed?</summary>
                          <p class="faq-a">
                            No ORAsol token mint is configured. The community program
                            does not create or distribute a token.
                          </p>
                        </details>
                        <details class="faq-item">
                          <summary>Is ORAsol an official Solana product?</summary>
                          <p class="faq-a">
                            No. ORAsol is an independent community project currently
                            reading {{ SOLANA_NETWORK_NAME }}. It is not affiliated with the
                            Solana Foundation.
                          </p>
                        </details>
                        <details class="faq-item">
                          <summary>Why do I see preview rows or a connection error?</summary>
                          <p class="faq-a">
                            Preview means sample rows are shown and nothing is submitted. A
                            connection error means configured Solana data could not be read
                            — use the refresh button to retry. Never share a seed phrase or
                            private key to fix a connection.
                          </p>
                        </details>
                      </div>
                    </div>
                  </section>
                </div>

                <aside class="aside">
                  <section class="card">
                    <header class="card-head">
                      <h2>Checklist</h2>
                    </header>
                    <div class="card-body is-flush">
                      <ul class="check-list">
                        <li :class="{ 'is-done': connected }">
                          <span class="check-mark">{{ connected ? "✓" : "1" }}</span>
                          <div>
                            <b>Bring a Solana wallet</b>
                            <small>{{ connected ? "connected" : "connect to read balances" }}</small>
                          </div>
                        </li>
                        <li>
                          <span class="check-mark">2</span>
                          <div>
                            <b>Match the cluster</b>
                            <small>set your wallet to {{ SOLANA_NETWORK_NAME }}</small>
                          </div>
                        </li>
                        <li>
                          <span class="check-mark">3</span>
                          <div>
                            <b>Keep a little SOL</b>
                            <small>{{ SOLANA_CLUSTER }} fees are paid in SOL</small>
                          </div>
                        </li>
                      </ul>
                    </div>
                    <div class="card-foot">
                      <a
                        v-if="SOLANA_CLUSTER === 'devnet'"
                        class="btn btn-primary btn-block"
                        href="https://faucet.solana.com/"
                        target="_blank"
                        rel="noopener noreferrer"
                      >
                        Get devnet SOL <Icon name="arrow-up-right" :size="14" />
                      </a>
                      <button
                        v-else
                        class="btn btn-primary btn-block"
                        type="button"
                        :disabled="connecting || busy"
                        @click="connected ? disconnectWallet() : connect()"
                      >
                        <Icon name="wallet" :size="15" />
                        {{ connected ? "Disconnect wallet" : "Connect wallet" }}
                      </button>
                    </div>
                  </section>

                  <div class="note">
                    <Icon name="info" :size="18" />
                    <div>
                      <strong>Sample rows</strong>
                      <p>
                        Rows marked “sample” are examples. They never send a transaction and
                        disappear as soon as the program reads real data.
                      </p>
                    </div>
                  </div>
                </aside>
              </div>
            </template>
          </section>
        </Transition>
      </div>
    </main>

    <SiteFooter
      :tabs="navTabs"
      :network-name="SOLANA_NETWORK_NAME"
      :cluster="SOLANA_CLUSTER"
      :program-state="programState"
      :mint-state="mintState"
      :wallet-label="walletLabel"
      :last-read="lastRead || 'Not yet'"
      @navigate="navigate"
    />

    <aside
      v-if="txState.stage !== 'idle'"
      class="toast"
      :class="{ 'is-error': txState.stage === 'error', 'is-success': txState.stage === 'success' }"
      :role="txState.stage === 'error' ? 'alert' : 'status'"
      aria-live="polite"
      aria-atomic="true"
    >
      <span class="toast-dot" aria-hidden="true"></span>
      <div class="toast-copy">
        <strong>{{ txState.label }}</strong>
        <p>{{ txState.message }}</p>
        <a
          v-if="txState.hash"
          :href="explorer('tx', txState.hash)"
          target="_blank"
          rel="noopener noreferrer"
        >View transaction <Icon name="arrow-up-right" :size="13" /></a>
      </div>
      <button
        v-if="!busy"
        class="icon-btn toast-close"
        type="button"
        aria-label="Dismiss transaction status"
        @click="dismissToast"
      >
        <Icon name="close" :size="16" />
      </button>
    </aside>

    <WalletSheet
      v-if="walletPickerOpen"
      :options="walletOptions"
      :network-name="SOLANA_NETWORK_NAME"
      @choose="chooseWallet"
      @close="closeWalletPicker"
    />
  </div>
</template>
