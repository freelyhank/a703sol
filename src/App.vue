<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick, watch } from "vue";
import { useCommunityApp } from "./useCommunityApp";
import {
  SOLANA_CLUSTER,
  SOLANA_NETWORK_NAME,
  TOKEN_MINT_ADDRESS,
} from "./solana.js";
import brandMark from "./assets/tabulora-mark.svg";
import Icon from "./Icon.vue";

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
  { id: "home", index: "00", label: "Register", hint: "overview" },
  { id: "quests", index: "01", label: "Quests", hint: "participation" },
  { id: "stake", index: "02", label: "Token", hint: "not configured" },
  { id: "governance", index: "03", label: "Votes", hint: "advisory" },
  { id: "treasury", index: "04", label: "Ledger", hint: "public reads" },
  { id: "learn", index: "05", label: "Guide", hint: "how it works" },
];
const validTabs = routes.map((route) => route.id);
const route = ref(
  validTabs.includes(location.hash.slice(1)) ? location.hash.slice(1) : "home",
);

function syncFromHash() {
  const next = location.hash.slice(1);
  route.value = validTabs.includes(next) ? next : "home";
}
async function go(next) {
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
function focusMain() {
  document.getElementById("main")?.focus({ preventScroll: true });
}
onMounted(() => {
  window.addEventListener("hashchange", syncFromHash);
  window.addEventListener("popstate", syncFromHash);
});
onUnmounted(() => {
  window.removeEventListener("hashchange", syncFromHash);
  window.removeEventListener("popstate", syncFromHash);
});

const current = computed(
  () => routes.find((entry) => entry.id === route.value) || routes[0],
);

/* ---------------------------------------------------------------- overview */

const samples = [
  {
    id: "sample-1",
    sample: true,
    title: "Read the community charter",
    reward: 12n * 10n ** 9n,
    detail: "A sample welcome quest. See how a ledger row is written.",
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

const previewNote = computed(() => {
  if (configured)
    return "The program is not initialised on this cluster. Sample quests and proposals stay on screen; actions submit nothing.";
  if (mintConfigured)
    return "Mint data is read-only and sample program rows remain until a program is configured. Actions submit nothing.";
  return "No token mint is configured and the community program is not connected. Sample rows are shown; actions submit nothing.";
});

const actionDisabled = computed(
  () =>
    busy.value ||
    connecting.value ||
    (configured &&
      (!ready.value ||
        reading.value ||
        paused.value ||
        (connected.value && (!accountReady.value || accountReading.value)))),
);
const tokenIdentity = computed(() => {
  if (!mintConfigured) return "No token mint configured";
  if (tokenName.value) return `${tokenName.value} (${tokenSymbol.value})`;
  return reading.value
    ? "Loading mint data..."
    : `SPL token (${tokenSymbol.value}); metadata pending`;
});
const walletCell = computed(() =>
  !connected.value ? "none" : accountReady.value ? short(wallet.value) : "syncing",
);
const syncCell = computed(() => {
  if (readError.value) return "needs attention";
  if (reading.value) return "reading";
  return lastRead.value || "idle";
});

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

/* -------------------------------------------------------------- vote view */

const proposalText = ref("");
const proposalDays = ref(7);
const composeOpen = ref(false);
const draftSaved = ref(false);
const clearedDraft = ref(null);
watch(proposalText, (text) => {
  if (text) clearedDraft.value = null;
});

const draftKey = "tabulora-proposal-draft";
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
  <div class="shell">
    <a class="skip-link" href="#main" @click.prevent="focusMain">Skip to the ledger</a>

    <aside class="rail">
      <button class="rail-brand" aria-label="Tabulora register" @click="go('home')">
        <img :src="brandMark" width="40" height="40" alt="" />
        <span>
          <b>tabulora</b>
          <small>every hand on the record</small>
        </span>
      </button>
      <p class="rail-mark">
        SOLANA <em>/</em> {{ SOLANA_CLUSTER.toUpperCase() }}<br />
        ROWS <em>/</em> {{ allQuests.length }} QUESTS <em>/</em> {{ visibleProposals.length }} VOTES
      </p>
      <nav class="rail-nav" aria-label="Ledger sections">
        <button
          v-for="entry in routes"
          :key="entry.id"
          :aria-current="route === entry.id ? 'page' : undefined"
          @click="go(entry.id)"
        >
          <i>{{ entry.index }}</i>
          <span>{{ entry.label }}</span>
          <small>{{ entry.hint }}</small>
        </button>
      </nav>
      <div class="rail-foot">
        <span>{{ walletCell === "none" ? "WALLET / NONE" : "WALLET / " + walletCell }}</span>
        <span>READ / {{ syncCell.toUpperCase() }}</span>
        <span>TOKEN / NONE ISSUED</span>
      </div>
    </aside>

    <div class="main">
      <header class="command">
        <div class="command-route">
          <b>{{ current.index }} / {{ current.label }}</b>
          <small>{{ current.hint }} · {{ SOLANA_NETWORK_NAME }}</small>
        </div>
        <div class="command-reads" role="group" aria-label="Network readouts">
          <div class="read-cell">
            <small>cluster</small>
            <b class="acid">{{ SOLANA_CLUSTER }}</b>
          </div>
          <div class="read-cell">
            <small>program</small>
            <b :class="programReady ? 'acid' : 'quiet'">
              {{ programReady ? "ready" : configured ? "pending" : "not set" }}
            </b>
          </div>
          <div class="read-cell">
            <small>open quests</small>
            <b>{{ metricsAvailable ? openQuests : "—" }}</b>
          </div>
          <div class="read-cell">
            <small>open votes</small>
            <b>{{ metricsAvailable ? openProposals : "—" }}</b>
          </div>
          <div class="read-cell">
            <small>token mint</small>
            <b class="quiet">{{ mintConfigured ? short(TOKEN_MINT_ADDRESS) : "none" }}</b>
          </div>
        </div>
        <div class="command-actions">
          <button
            class="sync-button"
            :disabled="reading || busy"
            :aria-label="`Re-read ${SOLANA_NETWORK_NAME} data`"
            @click="refresh"
          >
            <Icon name="refresh" :size="14" :class="{ spin: reading }" />
            <span>{{ reading ? "reading" : "sync" }}</span>
          </button>
          <button
            class="wallet-button"
            :class="{ ghost: connected }"
            :disabled="connecting || busy"
            @click="connected ? disconnectWallet() : connect()"
          >
            <Icon name="wallet" :size="15" />
            <span>{{ connecting ? "connecting" : connected ? short(wallet) : "connect wallet" }}</span>
          </button>
        </div>
      </header>

      <main id="main" class="route" tabindex="-1">
        <div v-if="!programReady" class="notice acid">
          <Icon name="info" :size="18" />
          <div>
            <strong>Preview rows</strong>
            <p>{{ previewNote }}</p>
          </div>
          <button class="action quiet spacer" @click="go('learn')">
            How it works <Icon name="arrow" :size="14" />
          </button>
        </div>
        <div v-if="paused" class="notice ember" role="status">
          <Icon name="info" :size="18" />
          <div>
            <strong>Writes paused</strong>
            <p>The protocol is paused. Reads continue; transactions are unavailable.</p>
          </div>
        </div>
        <div v-if="readError" class="notice bad" role="alert">
          <Icon name="info" :size="18" />
          <div>
            <strong>Sync failed</strong>
            <p>{{ readError }}</p>
          </div>
          <button class="action spacer" :disabled="reading || busy" @click="refresh">Try again</button>
        </div>
        <p v-if="accountError" class="fine" role="alert">{{ accountError }}</p>
        <p v-if="accountReading" class="fine" role="status">Reading wallet balances…</p>

        <!-- 00 / REGISTER ------------------------------------------------- -->
        <template v-if="route === 'home'">
          <div class="route-head">
            <div>
              <span class="eyebrow">Open ledger / community work</span>
              <h1>Every hand <em>on the record.</em></h1>
              <p>
                Quests, advisory votes and wallet state written to one ledge anyone can
                reopen. Nothing here mints a token or moves community funds.
              </p>
            </div>
            <span class="route-tag">{{ SOLANA_NETWORK_NAME }}</span>
          </div>

          <section class="flow" aria-label="Ledger totals">
            <div class="flow-cell">
              <small>open quests</small>
              <b :class="{ acid: metricsAvailable }">{{ metricsAvailable ? openQuests : "—" }}</b>
              <span>{{ metricsAvailable ? "reading the program" : "sample rows shown" }}</span>
            </div>
            <div class="flow-cell">
              <small>advisory votes open</small>
              <b :class="{ acid: metricsAvailable }">{{ metricsAvailable ? openProposals : "—" }}</b>
              <span>one vote per wallet per proposal</span>
            </div>
            <div class="flow-cell">
              <small>token mint</small>
              <b class="dim">{{ mintConfigured ? short(TOKEN_MINT_ADDRESS) : "not configured" }}</b>
              <span>no mint, no rewards, no locks</span>
            </div>
            <div class="flow-cell">
              <small>native balance</small>
              <b class="dim">{{ connected && accountReady ? solUnits(native, 4) + " SOL" : "connect wallet" }}</b>
              <span>{{ SOLANA_NETWORK_NAME }}</span>
            </div>
          </section>

          <div class="ticker" role="status">
            <span>READ <b>{{ syncCell }}</b></span>
            <span>WALLET <b>{{ walletCell }}</b></span>
            <span>TREASURY <b>{{ ready && treasuryNativeKnown ? solUnits(treasuryNative, 4) + " SOL" : "unavailable" }}</b></span>
            <span>PROGRAM <b>{{ programReady ? "initialised" : "preview" }}</b></span>
            <span>MAX CLAIM CAP <b>on chain</b></span>
          </div>

          <div class="split">
            <div class="stack">
              <section class="band">
                <div class="band-head">
                  <h2>Open rows / quests</h2>
                  <button class="action quiet" @click="go('quests')">
                    All quests <Icon name="arrow" :size="14" />
                  </button>
                </div>
                <div class="band-body tight">
                  <div v-if="!registerQuests.length" class="empty">
                    <strong>No rows yet</strong>
                    <p>Quests appear here once the community publishes them.</p>
                  </div>
                  <button
                    v-for="(q, index) in registerQuests"
                    :key="q.id"
                    class="row"
                    @click="openQuest(q)"
                  >
                    <span class="num">{{ String(index + 1).padStart(2, "0") }}</span>
                    <span class="copy">
                      <b>{{ q.title }}</b>
                      <small>{{ q.sample ? "sample row" : "quest / " + String(q.id).padStart(3, "0") }}</small>
                    </span>
                    <span class="tail">{{ q.sample ? "preview" : questStatus(q) }}</span>
                  </button>
                </div>
              </section>

              <section class="band">
                <div class="band-head">
                  <h2>Advisory votes</h2>
                  <button class="action quiet" @click="go('governance')">
                    Open the floor <Icon name="arrow" :size="14" />
                  </button>
                </div>
                <div class="band-body tight">
                  <button
                    v-for="p in registerProposals"
                    :key="p.id"
                    class="row"
                    @click="go('governance')"
                  >
                    <span class="num">{{ p.sample ? "ex" : String(p.id).padStart(3, "0") }}</span>
                    <span class="copy">
                      <b>{{ p.description }}</b>
                      <small>{{ p.sample ? "sample proposal" : "closes " + date(p.endsAt) }}</small>
                    </span>
                    <span class="tail">{{ proposalStatus(p) }}</span>
                  </button>
                </div>
                <div class="band-body tight">
                  <div class="tally">
                    <div>
                      <small>for</small>
                      <b class="for">{{ units(registerProposals[0]?.forVotes || 0n) }}</b>
                    </div>
                    <div>
                      <small>against</small>
                      <b class="against">{{ units(registerProposals[0]?.againstVotes || 0n) }}</b>
                    </div>
                    <div>
                      <small>support</small>
                      <b>{{ registerProposals[0] ? votePercent(registerProposals[0]) + "%" : "—" }}</b>
                    </div>
                  </div>
                </div>
              </section>
            </div>

            <aside class="side">
              <section class="band">
                <div class="band-head">
                  <h2>Wallet read</h2>
                  <span class="band-note">{{ connected ? "connected" : "idle" }}</span>
                </div>
                <div class="band-body tight">
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
                      <dd class="muted">{{ mintConfigured && connected && accountReady ? units(token) : "not configured" }}</dd>
                    </div>
                    <div>
                      <dt>State</dt>
                      <dd>{{ connected ? (accountReady ? "synced" : "reading") : "connect to read" }}</dd>
                    </div>
                  </dl>
                </div>
                <div class="band-body">
                  <button
                    class="action solid wide"
                    :disabled="connecting || busy"
                    @click="connected ? disconnectWallet() : connect()"
                  >
                    <Icon name="wallet" :size="15" />
                    {{ connected ? "Disconnect" : "Connect wallet" }}
                  </button>
                </div>
              </section>

              <section class="band">
                <div class="band-head"><h2>Token status</h2></div>
                <div class="band-body">
                  <p class="fine">{{ tokenIdentity }}</p>
                  <p class="fine">
                    The community program records quest participation and advisory
                    votes. It does not mint, custody or distribute tokens.
                  </p>
                  <button class="action wide" @click="go('stake')">
                    Token details <Icon name="arrow" :size="14" />
                  </button>
                </div>
              </section>

              <section class="band">
                <div class="band-head"><h2>Reading guide</h2></div>
                <div class="band-body">
                  <p class="fine">
                    A preview never sends a transaction. Reviews happen in your wallet
                    before anything is signed. Never share a seed phrase to fix a
                    connection.
                  </p>
                  <button class="action wide" @click="go('learn')">
                    Open the guide <Icon name="arrow" :size="14" />
                  </button>
                </div>
              </section>
            </aside>
          </div>
        </template>

        <!-- 01 / QUESTS --------------------------------------------------- -->
        <template v-if="route === 'quests'">
          <div class="route-head">
            <div>
              <span class="eyebrow">01 / Participation rows</span>
              <h1>Find your part.</h1>
              <p>
                Each row records wallet participation only. Recording a row is not
                proof of off-chain work, and it pays no token.
              </p>
            </div>
            <span class="route-tag">{{ allQuests.length }} rows</span>
          </div>

          <section class="band">
            <div class="toolbar">
              <label class="field">
                <Icon name="search" :size="16" />
                <span class="sr-only">Search quests</span>
                <input
                  v-model="search"
                  type="search"
                  placeholder="Find a row…"
                  autocomplete="off"
                />
              </label>
              <div class="segments" role="group" aria-label="Filter quests">
                <button
                  v-for="f in questFilters"
                  :key="f"
                  :aria-pressed="questFilter === f"
                  @click="questFilter = f"
                >
                  {{ f }}
                </button>
              </div>
              <span class="count" role="status">
                {{ visibleQuests.length }} {{ visibleQuests.length === 1 ? "row" : "rows" }}
              </span>
            </div>

            <div v-if="reading && !allQuests.length" role="status">
              <span class="sr-only">Loading quests</span>
              <div v-for="i in 3" :key="i" class="skeleton"></div>
            </div>

            <div v-else-if="!visibleQuests.length" class="empty">
              <strong>
                {{ readError ? "Rows unavailable" : search || questFilter !== "all" ? "No matching rows" : "Room for the first row" }}
              </strong>
              <p>
                {{
                  readError
                    ? "Retry the connection to read the latest quests."
                    : search || questFilter !== "all"
                      ? "Try another keyword or show every row."
                      : "No quests have been published yet. Check back for the next activity."
                }}
              </p>
              <button
                v-if="search || questFilter !== 'all'"
                class="action"
                @click="resetFilters"
              >
                Clear filters
              </button>
              <button v-else-if="readError" class="action" :disabled="reading" @click="refresh">
                Try again
              </button>
            </div>

            <table v-else class="ledger-table">
              <caption class="sr-only">Community quest rows</caption>
              <thead>
                <tr>
                  <th scope="col">Row</th>
                  <th scope="col">Quest</th>
                  <th scope="col">Status</th>
                  <th scope="col">Capacity</th>
                  <th scope="col">Expires</th>
                  <th scope="col"><span class="sr-only">Action</span></th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="(q, index) in visibleQuests"
                  :key="q.id"
                  :id="'quest-row-' + q.id"
                  tabindex="-1"
                >
                  <td class="id">{{ q.sample ? "EX" : String(index + 1).padStart(2, "0") }}</td>
                  <td class="title">
                    {{ q.title }}
                    <small>{{ q.sample ? "sample row" : q.detail || "Community participation record" }}</small>
                  </td>
                  <td>
                    <span class="tag" :class="questStatus(q).toLowerCase()">{{ questStatus(q) }}</span>
                  </td>
                  <td class="meta">
                    <template v-if="q.sample">preview only</template>
                    <template v-else>
                      <span class="meter-row">
                        <span class="meter">
                          <i :style="{ width: Math.min(100, Math.round((q.claims / (q.maxClaims || 1)) * 100)) + '%' }"></i>
                        </span>
                        {{ q.claims }}/{{ q.maxClaims }}
                      </span>
                    </template>
                  </td>
                  <td class="meta">{{ q.sample ? "—" : date(q.expiresAt) }}</td>
                  <td class="act">
                    <button
                      class="action"
                      :disabled="actionDisabled || (!q.sample && questStatus(q) !== 'Open')"
                      @click="claim(q)"
                    >
                      {{
                        q.sample
                          ? "Preview"
                          : q.claimed
                            ? "Recorded"
                            : questStatus(q) !== "Open"
                              ? questStatus(q)
                              : connected
                                ? "Record row"
                                : "Connect"
                      }}
                    </button>
                    <button
                      v-if="isOwner && !q.sample"
                      class="action quiet"
                      :disabled="busy"
                      @click="setQuest(q)"
                    >
                      {{ q.active ? "Pause" : "Resume" }}
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </section>

          <form v-if="isOwner" class="band" @submit.prevent="createQuest">
            <div class="band-head">
              <h2>Steward tools / publish a row</h2>
              <span class="band-note">authority wallet</span>
            </div>
            <div class="band-body form-grid">
              <div class="form-grid two">
                <label>
                  <span>Quest title</span>
                  <input v-model="adminTitle" maxlength="80" required placeholder="Welcome to the community" />
                </label>
                <label>
                  <span>Details</span>
                  <input v-model="adminDetails" maxlength="240" placeholder="What should participants review?" />
                </label>
                <label>
                  <span>Maximum rows</span>
                  <input v-model="adminMax" type="number" min="1" max="1000000" required />
                </label>
                <label>
                  <span>Expires in days</span>
                  <input v-model="adminDays" type="number" min="1" max="365" required />
                </label>
              </div>
              <div class="form-foot">
                <small>Claims write a participation row only / no token reward</small>
                <button class="action solid" :disabled="actionDisabled" type="submit">
                  <Icon name="plus" :size="15" /> Publish row
                </button>
              </div>
            </div>
          </form>

          <div class="notice">
            <Icon name="shield" :size="18" />
            <div>
              <strong>Your wallet, your decision</strong>
              <p>Review the quest and the network before confirming. A preview never sends a transaction.</p>
            </div>
            <button class="action quiet spacer" @click="go('learn')">
              How quests work <Icon name="arrow" :size="14" />
            </button>
          </div>
        </template>

        <!-- 02 / TOKEN ---------------------------------------------------- -->
        <template v-if="route === 'stake'">
          <div class="route-head">
            <div>
              <span class="eyebrow">02 / Token details</span>
              <h1>No token is <em>wired up.</em></h1>
              <p>
                Quests and advisory votes run through the Solana program. Locking,
                rewards and staking stay disabled until a mint and its custody rules
                are defined.
              </p>
            </div>
            <span class="route-tag">not configured</span>
          </div>

          <div class="split">
            <div class="stack">
              <section class="band">
                <div class="band-head">
                  <h2>Token status</h2>
                  <span class="tag closed">not configured</span>
                </div>
                <div class="band-body tight">
                  <dl class="defs">
                    <div>
                      <dt>On-chain token</dt>
                      <dd class="muted">{{ tokenIdentity }}</dd>
                    </div>
                    <div>
                      <dt>Token staking</dt>
                      <dd class="muted">not implemented</dd>
                    </div>
                    <div>
                      <dt>Locking</dt>
                      <dd class="muted">not implemented</dd>
                    </div>
                    <div>
                      <dt>Reward source</dt>
                      <dd class="muted">undefined</dd>
                    </div>
                    <div>
                      <dt>Decimals</dt>
                      <dd class="muted">{{ mintConfigured ? tokenDecimals : "—" }}</dd>
                    </div>
                  </dl>
                </div>
                <div class="band-body">
                  <p class="fine">
                    The program records participation rows and advisory votes. It does
                    not custody tokens, mint rewards or track staking positions.
                  </p>
                </div>
              </section>

              <section class="band">
                <div class="band-head">
                  <h2>What would have to exist first</h2>
                  <span class="band-note">in order</span>
                </div>
                <div class="band-body tight">
                  <ol class="steps">
                    <li class="step">
                      <span class="idx">01</span>
                      <div>
                        <h3>A verified SPL mint</h3>
                        <p>The mint address, decimals and metadata would be read from Solana before any label appears here.</p>
                      </div>
                    </li>
                    <li class="step">
                      <span class="idx">02</span>
                      <div>
                        <h3>Written custody rules</h3>
                        <p>Who can move locked balances, how a holder exits, and what happens if the program pauses.</p>
                      </div>
                    </li>
                    <li class="step">
                      <span class="idx">03</span>
                      <div>
                        <h3>A funded reward source</h3>
                        <p>Rewards need money behind them. Until that exists, no reward can be promised in a row.</p>
                      </div>
                    </li>
                  </ol>
                </div>
              </section>
            </div>

            <aside class="side">
              <section class="band">
                <div class="band-head"><h2>Network wiring</h2></div>
                <div class="band-body tight">
                  <dl class="defs">
                    <div>
                      <dt>Cluster</dt>
                      <dd>{{ SOLANA_CLUSTER }}</dd>
                    </div>
                    <div>
                      <dt>Network</dt>
                      <dd class="muted">{{ SOLANA_NETWORK_NAME }}</dd>
                    </div>
                    <div>
                      <dt>Program</dt>
                      <dd>{{ programReady ? "initialised" : configured ? "pending" : "not set" }}</dd>
                    </div>
                    <div>
                      <dt>Mint</dt>
                      <dd class="muted">{{ mintConfigured ? short(TOKEN_MINT_ADDRESS) : "none" }}</dd>
                    </div>
                  </dl>
                </div>
                <div class="band-body">
                  <button class="action wide" @click="go('learn')">
                    Read the field guide <Icon name="external" :size="14" />
                  </button>
                </div>
              </section>
              <div class="notice ember">
                <Icon name="info" :size="18" />
                <div>
                  <strong>No mint has been issued</strong>
                  <p>Anything claiming a Tabulora token right now is not from this project.</p>
                </div>
              </div>
            </aside>
          </div>
        </template>

        <!-- 03 / VOTES ---------------------------------------------------- -->
        <template v-if="route === 'governance'">
          <div class="route-head">
            <div>
              <span class="eyebrow">03 / Advisory votes</span>
              <h1>Put it to <em>the room.</em></h1>
              <p>
                Write an idea, choose a 1–30 day window, and let wallets answer. One
                vote per wallet; results are advisory and never move funds.
              </p>
            </div>
            <span class="route-tag">{{ visibleProposals.length }} proposals</span>
          </div>

          <div class="split">
            <div class="stack">
              <section class="band">
                <div class="band-head">
                  <h2>Proposal ledger</h2>
                  <span class="band-note">{{ programReady ? "on chain" : "sample rows" }}</span>
                </div>
                <div class="band-body tight">
                  <div v-if="programReady && reading && !visibleProposals.length" class="skeleton" role="status">
                    <span class="sr-only">Loading proposals</span>
                  </div>
                  <div
                    v-else-if="!visibleProposals.length"
                    class="empty"
                  >
                    <strong>{{ readError ? "Proposals unavailable" : "The floor is open" }}</strong>
                    <p>
                      {{
                        readError
                          ? "Use sync above to reconnect."
                          : "Start the first proposal with a clear idea and a voting window."
                      }}
                    </p>
                    <button v-if="!readError && !composeOpen" class="action" @click="toggleCompose">
                      Write the first proposal
                    </button>
                  </div>
                  <template v-else>
                    <article v-for="p in visibleProposals" :key="p.id">
                      <table class="ledger-table">
                        <tbody>
                          <tr>
                            <td class="id">{{ p.sample ? "EX" : String(p.id).padStart(3, "0") }}</td>
                            <td class="title">
                              {{ p.description }}
                              <small>
                                {{ p.sample ? "sample proposal / no live votes" : "closes " + date(p.endsAt) }}
                              </small>
                            </td>
                            <td class="act">
                              <span class="tag" :class="proposalStatus(p).toLowerCase()">{{ proposalStatus(p) }}</span>
                            </td>
                          </tr>
                        </tbody>
                      </table>
                      <div class="tally">
                        <div>
                          <small>for</small>
                          <b class="for">{{ units(p.forVotes) }}</b>
                        </div>
                        <div>
                          <small>against</small>
                          <b class="against">{{ units(p.againstVotes) }}</b>
                        </div>
                        <div>
                          <small>support</small>
                          <b>{{ p.forVotes + p.againstVotes === 0n ? "—" : votePercent(p) + "%" }}</b>
                        </div>
                        <div>
                          <small>cast weight</small>
                          <span class="meter-row">
                            <span class="meter">
                              <i :style="{ width: votePercent(p) + '%' }"></i>
                            </span>
                          </span>
                        </div>
                      </div>
                      <div class="band-body">
                        <div class="form-foot">
                          <small>
                            {{
                              p.voted
                                ? "Your vote is recorded"
                                : p.forVotes + p.againstVotes === 0n
                                  ? "Waiting for the first vote"
                                  : votePercent(p) + "% of cast weight supports this"
                            }}
                          </small>
                          <span class="step-actions">
                            <button
                              class="action solid"
                              :disabled="actionDisabled || (!p.sample && proposalStatus(p) !== 'Open')"
                              @click="vote(p, true)"
                            >
                              <Icon name="check" :size="14" /> Vote for
                            </button>
                            <button
                              class="action ember"
                              :disabled="actionDisabled || (!p.sample && proposalStatus(p) !== 'Open')"
                              @click="vote(p, false)"
                            >
                              <Icon name="close" :size="14" /> Vote against
                            </button>
                          </span>
                        </div>
                      </div>
                    </article>
                  </template>
                </div>
              </section>

              <div class="notice">
                <Icon name="info" :size="18" />
                <div>
                  <strong>Advisory only</strong>
                  <p>
                    Votes do not execute treasury spending. Voting weight follows the
                    configured program rules; your balance may differ from eligible weight.
                  </p>
                </div>
              </div>
            </div>

            <aside class="side">
              <section class="band">
                <div class="band-head">
                  <h2>Compose</h2>
                  <span class="band-note">{{ draftSaved && !composeOpen ? "draft saved" : "10–2,000 chars" }}</span>
                </div>
                <div class="band-body">
                  <button
                    id="compose-toggle"
                    class="action wide"
                    :aria-expanded="composeOpen"
                    aria-controls="compose-editor"
                    @click="toggleCompose"
                  >
                    <Icon :name="composeOpen ? 'close' : 'plus'" :size="15" />
                    {{ composeOpen ? "Close editor" : draftSaved ? "Continue your draft" : "New proposal" }}
                  </button>
                </div>
                <form
                  v-if="composeOpen"
                  id="compose-editor"
                  class="band-body form-grid"
                  @submit.prevent="createProposal"
                  @keydown.esc.prevent="toggleCompose"
                >
                  <label>
                    <span>Proposal description</span>
                    <textarea
                      id="proposal-description"
                      v-model="proposalText"
                      minlength="10"
                      maxlength="2000"
                      required
                      placeholder="Describe the decision, its purpose, any budget, and the intended outcome."
                    ></textarea>
                  </label>
                  <label>
                    <span>Voting window (days)</span>
                    <input v-model="proposalDays" type="number" min="1" max="30" required />
                  </label>
                  <div class="form-foot">
                    <small>{{ proposalText.length }} / 2,000</small>
                    <button
                      class="action solid"
                      :disabled="actionDisabled || proposalText.trim().length < 10"
                      type="submit"
                    >
                      {{ programReady ? "Publish proposal" : "Preview proposal" }}
                    </button>
                  </div>
                  <div class="form-foot">
                    <button class="action quiet" type="button" @click="toggleCompose">
                      Keep draft &amp; close
                    </button>
                    <button v-if="proposalText" class="action quiet" type="button" @click="clearDraft">
                      Clear draft
                    </button>
                    <button v-else-if="clearedDraft" class="action quiet" type="button" @click="undoClearDraft">
                      Undo clear
                    </button>
                  </div>
                </form>
              </section>

              <section class="band">
                <div class="band-head"><h2>A proposal that carries</h2></div>
                <div class="band-body tight">
                  <ol class="steps">
                    <li class="step">
                      <span class="idx">01</span>
                      <div>
                        <h3>One decision</h3>
                        <p>State a single decision and why it matters now.</p>
                      </div>
                    </li>
                    <li class="step">
                      <span class="idx">02</span>
                      <div>
                        <h3>Context and cost</h3>
                        <p>Name the outcome and any budget the room should expect.</p>
                      </div>
                    </li>
                    <li class="step">
                      <span class="idx">03</span>
                      <div>
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

        <!-- 04 / LEDGER --------------------------------------------------- -->
        <template v-if="route === 'treasury'">
          <div class="route-head">
            <div>
              <span class="eyebrow">04 / Public reads</span>
              <h1>Nothing hidden <em>in the books.</em></h1>
              <p>
                Treasury balances and account rows are read straight from Solana. When a
                read is unavailable, this page says so instead of guessing.
              </p>
            </div>
            <span class="route-tag">{{ SOLANA_NETWORK_NAME }}</span>
          </div>

          <section class="flow" aria-label="Treasury reads">
            <div class="flow-cell">
              <small>treasury token balance</small>
              <b class="dim">{{ ready && treasuryTokenKnown ? units(treasuryBalance) : "—" }}</b>
              <span>{{ tokenIdentity }}</span>
            </div>
            <div class="flow-cell">
              <small>treasury gas</small>
              <b class="dim">
                {{ ready && treasuryNativeKnown ? solUnits(treasuryNative, 6) + " SOL" : "—" }}
              </b>
              <span>native balance for fees</span>
            </div>
            <div class="flow-cell">
              <small>issued supply</small>
              <b class="dim">{{ mintConfigured && ready ? units(totalSupply) : "not configured" }}</b>
              <span>read from the mint</span>
            </div>
            <div class="flow-cell">
              <small>indexer</small>
              <b class="dim">not connected</b>
              <span>no activity index yet</span>
            </div>
          </section>

          <p v-if="treasuryError || (mintConfigured && ready && (!treasuryKnown || !treasuryTokenKnown))" class="fine" role="status">
            {{ treasuryError || "Treasury token or SOL balances are not fully available yet." }}
          </p>

          <div class="split">
            <div class="stack">
              <section class="band">
                <div class="band-head">
                  <h2>Account statement</h2>
                  <span class="band-note">{{ ready ? "read at " + lastRead : "awaiting read" }}</span>
                </div>
                <div class="band-body tight">
                  <dl class="defs">
                    <div>
                      <dt>On-chain token</dt>
                      <dd class="muted">{{ tokenIdentity }}</dd>
                    </div>
                    <div>
                      <dt>{{ mintConfigured ? `Total issued ${tokenSymbol}` : "Issued supply" }}</dt>
                      <dd>{{ mintConfigured && ready ? units(totalSupply) : "—" }}</dd>
                    </div>
                    <div>
                      <dt>Token staking</dt>
                      <dd class="muted">not implemented</dd>
                    </div>
                    <div>
                      <dt>{{ mintConfigured ? `Your ${tokenSymbol}` : "Your token balance" }}</dt>
                      <dd>
                        {{ connected && accountReady ? units(token) : connected ? "reading" : "connect wallet" }}
                      </dd>
                    </div>
                    <div>
                      <dt>Decimals</dt>
                      <dd class="muted">{{ mintConfigured ? tokenDecimals : "—" }}</dd>
                    </div>
                  </dl>
                </div>
              </section>

              <section class="band">
                <div class="band-head">
                  <h2>Activity register</h2>
                  <span class="band-note">not connected</span>
                </div>
                <div class="band-body">
                  <div class="empty">
                    <strong>{{ reading ? "Reading the Solana ledger…" : readError ? "Activity could not load" : "No indexer yet" }}</strong>
                    <p>A Solana activity index will be connected after the program is deployed for production use.</p>
                  </div>
                </div>
              </section>
            </div>

            <aside class="side">
              <section class="band">
                <div class="band-head"><h2>Network &amp; addresses</h2></div>
                <div class="band-body tight">
                  <dl class="defs">
                    <div>
                      <dt>Treasury wallet</dt>
                      <dd>
                        <a
                          v-if="treasury"
                          :href="explorer('address', treasury)"
                          target="_blank"
                          rel="noopener noreferrer"
                        >{{ short(treasury) }} <Icon name="external" :size="13" /></a>
                        <span v-else class="muted">not available</span>
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
                        >{{ short(TOKEN_MINT_ADDRESS) }} <Icon name="external" :size="13" /></a>
                        <span v-else class="muted">not configured</span>
                      </dd>
                    </div>
                    <div>
                      <dt>Network</dt>
                      <dd class="muted">{{ SOLANA_NETWORK_NAME }}</dd>
                    </div>
                    <div>
                      <dt>Cluster</dt>
                      <dd>{{ SOLANA_CLUSTER }}</dd>
                    </div>
                  </dl>
                </div>
                <div class="band-body">
                  <p class="fine">
                    Tabulora is the application brand. {{
                      mintConfigured
                        ? "Decimals come from the mint; display metadata stays pending until it is verified."
                        : "No token mint is deployed or configured."
                    }}
                  </p>
                </div>
              </section>

              <div class="notice">
                <Icon name="shield" :size="18" />
                <div>
                  <strong>Open rows cut both ways</strong>
                  <p>Anyone can reopen these reads, including people who disagree with the last decision.</p>
                </div>
              </div>
            </aside>
          </div>
        </template>

        <!-- 05 / GUIDE ---------------------------------------------------- -->
        <template v-if="route === 'learn'">
          <div class="route-head">
            <div>
              <span class="eyebrow">05 / The field guide</span>
              <h1>A way in. <em>A way to check.</em></h1>
              <p>
                What each row means, what your wallet will show you, and what this
                project deliberately does not do yet.
              </p>
            </div>
            <span class="route-tag">{{ SOLANA_NETWORK_NAME }}</span>
          </div>

          <div class="split">
            <div class="stack">
              <section class="band">
                <div class="band-head">
                  <h2>The route</h2>
                  <span class="band-note">three steps</span>
                </div>
                <div class="band-body tight">
                  <ol class="steps">
                    <li class="step">
                      <span class="idx">01</span>
                      <div>
                        <h3>Record a participation row</h3>
                        <p>
                          Quests carry an expiry and a row cap. Each wallet records once.
                          Rows do not verify off-chain work, prove a unique person, or pay
                          a token.
                        </p>
                        <div class="step-actions">
                          <button class="action" @click="go('quests')">
                            Open quests <Icon name="arrow" :size="14" />
                          </button>
                        </div>
                      </div>
                    </li>
                    <li class="step">
                      <span class="idx">02</span>
                      <div>
                        <h3>Wait for real token details</h3>
                        <p>
                          Locking and rewards need a verified mint, custody rules and a
                          funded reward source. None are enabled by the community program.
                        </p>
                        <div class="step-actions">
                          <button class="action" @click="go('stake')">
                            Token status <Icon name="arrow" :size="14" />
                          </button>
                        </div>
                      </div>
                    </li>
                    <li class="step">
                      <span class="idx">03</span>
                      <div>
                        <h3>Answer the next question</h3>
                        <p>
                          Create a proposal with a 1–30 day window, or vote on an open one.
                          One vote per wallet. Results are advisory.
                        </p>
                        <div class="step-actions">
                          <button class="action" @click="go('governance')">
                            Open votes <Icon name="arrow" :size="14" />
                          </button>
                        </div>
                      </div>
                    </li>
                  </ol>
                </div>
              </section>

              <section class="band">
                <div class="band-head"><h2>Good to know</h2></div>
                <div class="band-body tight">
                  <details class="qa" open>
                    <summary>What does a wallet confirmation do?</summary>
                    <p>
                      Recording a quest row, publishing a proposal and casting a vote use
                      the initialised program on {{ SOLANA_NETWORK_NAME }}. Your wallet
                      shows every instruction before you sign.
                    </p>
                  </details>
                  <details class="qa">
                    <summary>Who controls the protocol?</summary>
                    <p>
                      The program authority can publish quests and pause community actions.
                      Advisory votes do not execute treasury spending.
                    </p>
                  </details>
                  <details class="qa">
                    <summary>Why is the token name separate from Tabulora?</summary>
                    <p>
                      A future mint defines its own name, symbol and decimals. Tabulora is
                      the application brand. Current status: {{ tokenIdentity }}.
                    </p>
                  </details>
                  <details class="qa">
                    <summary>Is a Tabulora token deployed?</summary>
                    <p>
                      No Tabulora token mint is configured. The community program does not
                      create or distribute a token.
                    </p>
                  </details>
                  <details class="qa">
                    <summary>Is Tabulora an official Solana product?</summary>
                    <p>
                      No. Tabulora is an independent community project currently reading
                      {{ SOLANA_NETWORK_NAME }}. It is not affiliated with the Solana
                      Foundation.
                    </p>
                  </details>
                  <details class="qa">
                    <summary>Why do I see preview rows or a connection error?</summary>
                    <p>
                      Preview means sample rows are shown and nothing is submitted. A
                      connection error means configured Solana data could not be read — use
                      sync to retry. Never share a seed phrase or private key to fix a
                      connection.
                    </p>
                  </details>
                </div>
              </section>
            </div>

            <aside class="side">
              <section class="band">
                <div class="band-head"><h2>Checklist</h2></div>
                <div class="band-body tight">
                  <ul class="checklist">
                    <li :class="{ done: connected }">
                      <i>{{ connected ? "✓" : "1" }}</i>
                      <div>
                        <b>Bring a Solana wallet</b>
                        <small>{{ connected ? "connected" : "connect to read balances" }}</small>
                      </div>
                    </li>
                    <li>
                      <i>2</i>
                      <div>
                        <b>Match the cluster</b>
                        <small>set your wallet to {{ SOLANA_NETWORK_NAME }}</small>
                      </div>
                    </li>
                    <li>
                      <i>3</i>
                      <div>
                        <b>Keep a little SOL</b>
                        <small>{{ SOLANA_CLUSTER }} fees are paid in SOL</small>
                      </div>
                    </li>
                  </ul>
                </div>
                <div class="band-body">
                  <a
                    v-if="SOLANA_CLUSTER === 'devnet'"
                    class="action wide"
                    href="https://faucet.solana.com/"
                    target="_blank"
                    rel="noopener noreferrer"
                  >
                    Get Devnet SOL <Icon name="external" :size="14" />
                  </a>
                  <button
                    v-else
                    class="action wide"
                    :disabled="connecting || busy"
                    @click="connected ? disconnectWallet() : connect()"
                  >
                    <Icon name="wallet" :size="15" />
                    {{ connected ? "Disconnect wallet" : "Connect your wallet" }}
                  </button>
                </div>
              </section>

              <div class="notice acid">
                <Icon name="info" :size="18" />
                <div>
                  <strong>Sample mode</strong>
                  <p>
                    Rows marked “sample” are examples. They never send a transaction and
                    disappear as soon as the program reads real data.
                  </p>
                </div>
              </div>
            </aside>
          </div>
        </template>
      </main>

      <footer class="foot">
        <span>2026 Tabulora / every hand on the record</span>
        <span>
          Independent community project / {{ SOLANA_NETWORK_NAME }} /
          {{ configured ? "program configured" : "preview mode" }}
        </span>
      </footer>
    </div>

    <nav class="mobile-bar" aria-label="Ledger sections">
      <button
        v-for="entry in routes"
        :key="entry.id"
        :aria-current="route === entry.id ? 'page' : undefined"
        @click="go(entry.id)"
      >
        <i>{{ entry.index }}</i>
        <b>{{ entry.label }}</b>
      </button>
    </nav>

    <aside
      v-if="txState.stage !== 'idle'"
      class="toast"
      :class="txState.stage"
      :role="txState.stage === 'error' ? 'alert' : 'status'"
      aria-live="polite"
      aria-atomic="true"
    >
      <div>
        <strong>{{ txState.label }}</strong>
        <p>{{ txState.message }}</p>
        <a
          v-if="txState.hash"
          :href="explorer('tx', txState.hash)"
          target="_blank"
          rel="noopener noreferrer"
        >View transaction {{ short(txState.hash) }}</a>
      </div>
      <button
        v-if="!busy"
        class="action quiet"
        aria-label="Dismiss transaction status"
        @click="txState = { stage: 'idle', label: '', message: '', hash: '' }"
      >
        <Icon name="close" :size="15" />
      </button>
    </aside>
  </div>

  <div v-if="walletPickerOpen" class="picker-backdrop" @click.self="closeWalletPicker">
    <section class="picker" role="dialog" aria-modal="true" aria-labelledby="picker-title">
      <header>
        <h2 id="picker-title">Choose a wallet</h2>
        <button class="action quiet" aria-label="Close wallet picker" @click="closeWalletPicker">
          <Icon name="close" :size="15" />
        </button>
      </header>
      <p>Connect a wallet that supports {{ SOLANA_NETWORK_NAME }}.</p>
      <ul>
        <li v-for="candidate in walletOptions" :key="candidate.name">
          <button @click="chooseWallet(candidate)">
            <img v-if="candidate.icon" :src="candidate.icon" :alt="candidate.name + ' icon'" />
            <span v-else aria-hidden="true"></span>
            <span>{{ candidate.name }}</span>
            <Icon name="arrow" :size="15" />
          </button>
        </li>
      </ul>
      <footer>
        <button class="action" @click="closeWalletPicker">Cancel</button>
      </footer>
    </section>
  </div>
</template>
