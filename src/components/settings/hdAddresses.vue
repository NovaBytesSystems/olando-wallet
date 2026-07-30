<script setup lang="ts">
  import { ref, computed, watch, watchEffect } from 'vue';
  import Toggle from '@vueform/toggle'
  import { copyToClipboard, satsToBch } from 'src/utils/utils';
  import { useStore } from 'src/stores/store'
  import { useSettingsStore } from 'src/stores/settingsStore';
  import { useI18n } from 'vue-i18n'
  import { type HDWallet, type TestNetHDWallet, GAP_SIZE } from 'mainnet-js';
  import { useWindowSize } from '@vueuse/core'
  import LightboxPopup from 'src/components/general/lightbox-popup.vue'
  import type { LightboxButton } from 'src/components/general/lightbox-popup.vue'

  const store = useStore()
  const settingsStore = useSettingsStore()
  const { t } = useI18n()

  const { width } = useWindowSize();
  const isMobile = computed(() => width.value < 480);

  interface AddressRow {
    index: number;
    address: string;
    tokenAddress: string;
    balance: bigint;
    txCount: number;
  }

  const receivingAddresses = ref<AddressRow[]>([]);
  const changeAddresses = ref<AddressRow[]>([]);
  const showUsedReceiving = ref(false);
  const showUsedChange = ref(false);
  const hideZeroBalances = ref(false);
  const showTokenAddresses = ref(false);
  const changeDetailsOpen = ref(false);

  const showTokenFormatInfoPopup = ref(false);
  const tokenFormatInfoButtons: LightboxButton[] = [
    { label: "UNDERSTOOD", action: () => { showTokenFormatInfoPopup.value = false } }
  ];

  const showReceivingInfoPopup = ref(false);
  const receivingInfoButtons: LightboxButton[] = [
    { label: "UNDERSTOOD", action: () => { showReceivingInfoPopup.value = false } }
  ];

  const showChangeInfoPopup = ref(false);
  const changeInfoButtons: LightboxButton[] = [
    { label: "UNDERSTOOD", action: () => { showChangeInfoPopup.value = false } }
  ];

  watch(hideZeroBalances, (newVal) => {
    if (newVal) {
      showUsedReceiving.value = true;
      showUsedChange.value = true;
      changeDetailsOpen.value = true;
    }
  });

  function applyBalanceFilter(rows: AddressRow[]) {
    if (!hideZeroBalances.value) return rows;
    return rows.filter(r => r.balance > 0n);
  }

  const usedReceivingAddresses = computed(() => applyBalanceFilter(receivingAddresses.value.filter(r => r.txCount > 0)));
  const unusedReceivingAddresses = computed(() => applyBalanceFilter(receivingAddresses.value.filter(r => r.txCount === 0)));
  const usedChangeAddresses = computed(() => applyBalanceFilter(changeAddresses.value.filter(r => r.txCount > 0)));
  const unusedChangeAddresses = computed(() => applyBalanceFilter(changeAddresses.value.filter(r => r.txCount === 0)));

  const filteredReceivingCount = computed(() => usedReceivingAddresses.value.length + unusedReceivingAddresses.value.length);
  const filteredChangeCount = computed(() => usedChangeAddresses.value.length + unusedChangeAddresses.value.length);

  function displayAddress(row: AddressRow) {
    return showTokenAddresses.value ? row.tokenAddress : row.address;
  }

  function truncateAddress(address: string) {
    const body = address.split(':')[1] ?? "";
    const chars = isMobile.value ? 5 : 8;
    return body.slice(0, chars) + '...' + body.slice(-chars);
  }

  function getAddressBalance(utxos: { satoshis: bigint }[]): bigint {
    return utxos.reduce((sum, u) => sum + u.satoshis, 0n);
  }

  function buildAddressRows(hdWallet: HDWallet | TestNetHDWallet, index: number, change: boolean): AddressRow[] {
    const cache = hdWallet.walletCache;
    const rawHistory = change ? hdWallet.changeRawHistory : hdWallet.depositRawHistory;
    const rows: AddressRow[] = [];
    for (let i = 0; i < index + GAP_SIZE; i++) {
      const entry = cache.getByIndex(i, change);
      rows.push({
        index: i,
        address: entry.address,
        tokenAddress: entry.tokenAddress,
        balance: getAddressBalance(entry.utxos),
        txCount: rawHistory[i]?.length ?? 0,
      });
    }
    return rows;
  }

  // Rebuild when walletUtxos changes (triggers on balance/address updates)
  watchEffect(() => {
    // Access walletUtxos to establish reactive dependency
    void store.walletUtxos;
    const hdWallet = store.wallet as HDWallet | TestNetHDWallet;
    receivingAddresses.value = buildAddressRows(hdWallet, hdWallet.depositIndex, false);
    changeAddresses.value = buildAddressRows(hdWallet, hdWallet.changeIndex, true);
  });
</script>

<template>
  <fieldset class="item" :class="{ dark: settingsStore.darkMode }">
    <legend>{{ t('hdAddresses.title') }}</legend>

    <div class="filter-toggles">
      <div class="filter-toggle">
        {{ t('hdAddresses.hideZeroBalances') }} <Toggle v-model="hideZeroBalances" />
      </div>
      <div class="filter-toggle">
        {{ t('hdAddresses.showTokenAddresses') }} <Toggle v-model="showTokenAddresses" />
        <img src="images/olando/info.svg" class="action-icon" style="cursor:pointer; margin-left:6px; vertical-align:middle; width:28px; height:28px;" @click="showTokenFormatInfoPopup = true">
      </div>
    </div>

    <LightboxPopup
      v-model="showTokenFormatInfoPopup"
      icon="images/olando/info-white.svg"
      :blur="true"
      :buttons="tokenFormatInfoButtons"
    >
      <p>Normally standard Addresses are shown for receiving BCH.</p>
      <p>You can also show token Addresses for receiving CashTokens (such as OLA).</p>
    </LightboxPopup>

    <LightboxPopup
      v-model="showReceivingInfoPopup"
      icon="images/olando/info-white.svg"
      :blur="true"
      :buttons="receivingInfoButtons"
    >
      <p>An unlimited sequence of receiving Addresses are generated from your wallet seed phrase.</p>
      <p>You can use any of those addresses (you own the keys to them) for receiving, but to improve privacy it is suggested to not reuse addresses, but give out a fresh one each time you receive BCH or tokens.</p>
    </LightboxPopup>

    <!-- Receiving Addresses -->
    <details class="collapsible-section" open>
      <summary>
        <strong>{{ t('hdAddresses.receivingAddresses') }}</strong> ({{ filteredReceivingCount }})
        <img class="icon" :src="settingsStore.darkMode ? 'images/chevron-square-down-lightGrey.svg' : 'images/chevron-square-down.svg'">
        <img src="images/olando/info.svg" style="cursor:pointer; margin-left:6px; vertical-align:middle; width:28px; height:28px;" @click.stop="showReceivingInfoPopup = true">
      </summary>
      <table v-if="filteredReceivingCount" class="address-table">
        <thead>
          <tr>
            <th>{{ t('hdAddresses.columns.index') }}</th>
            <th>{{ t('hdAddresses.columns.address') }}</th>
            <th>{{ t('hdAddresses.columns.balance') }}</th>
            <th>{{ t('hdAddresses.columns.txs') }}</th>
          </tr>
        </thead>
        <!-- Used receiving addresses (collapsible) -->
        <tbody v-if="usedReceivingAddresses.length">
          <tr class="section-toggle" @click="showUsedReceiving = !showUsedReceiving">
            <td colspan="4">
              {{ t('hdAddresses.usedAddresses') }} ({{ usedReceivingAddresses.length }})
              <img class="icon" :class="{ open: showUsedReceiving }" :src="settingsStore.darkMode ? 'images/chevron-square-down-lightGrey.svg' : 'images/chevron-square-down.svg'">
            </td>
          </tr>
        </tbody>
        <tbody v-if="showUsedReceiving" class="used-addresses">
          <tr v-for="row in usedReceivingAddresses" :key="row.index">
            <td class="mono">{{ row.index }}</td>
            <td @click="copyToClipboard(displayAddress(row))" class="address-cell" :title="displayAddress(row)">
              <span class="mono">{{ truncateAddress(displayAddress(row)) }}</span>
              <img class="copyIcon" src="images/copyGrey.svg">
            </td>
            <td class="mono">{{ satsToBch(row.balance) }}</td>
            <td>{{ row.txCount }}</td>
          </tr>
        </tbody>
        <!-- Unused receiving addresses -->
        <tbody>
          <tr v-for="row in unusedReceivingAddresses" :key="row.index">
            <td class="mono">{{ row.index }}</td>
            <td @click="copyToClipboard(displayAddress(row))" class="address-cell" :title="displayAddress(row)">
              <span class="mono">{{ truncateAddress(displayAddress(row)) }}</span>
              <img class="copyIcon" src="images/copyGrey.svg">
            </td>
            <td class="mono">{{ satsToBch(row.balance) }}</td>
            <td>{{ row.txCount }}</td>
          </tr>
        </tbody>
      </table>
      <div v-else class="description">{{ t('hdAddresses.noAddresses') }}</div>
    </details>

    <LightboxPopup
      v-model="showChangeInfoPopup"
      icon="images/olando/info-white.svg"
      :blur="true"
      :buttons="changeInfoButtons"
    >
      <p>Just like the reveiving addresses above, these change adresses are derived from your wallet seed phrase. You own the keys to them and you can use them to receive funds.</p>
      <p>The wallet uses these addresses to send change to it when you make a payment (blockchain transactions have to "by design" spend full amounts you have received earlier, so in most cases there will be leftover change).</p>
    </LightboxPopup>

    <!-- Change Addresses -->
    <details class="collapsible-section" :open="changeDetailsOpen || undefined">
      <summary>
        <strong>{{ t('hdAddresses.changeAddresses') }}</strong> ({{ filteredChangeCount }})
        <img class="icon" :src="settingsStore.darkMode ? 'images/chevron-square-down-lightGrey.svg' : 'images/chevron-square-down.svg'">
        <img src="images/olando/info.svg" style="cursor:pointer; margin-left:6px; vertical-align:middle; width:28px; height:28px;" @click.stop="showChangeInfoPopup = true">
      </summary>
      <table v-if="filteredChangeCount" class="address-table">
        <thead>
          <tr>
            <th>{{ t('hdAddresses.columns.index') }}</th>
            <th>{{ t('hdAddresses.columns.address') }}</th>
            <th>{{ t('hdAddresses.columns.balance') }}</th>
            <th>{{ t('hdAddresses.columns.txs') }}</th>
          </tr>
        </thead>
        <!-- Used change addresses (collapsible) -->
        <tbody v-if="usedChangeAddresses.length">
          <tr class="section-toggle" @click="showUsedChange = !showUsedChange">
            <td colspan="4">
              {{ t('hdAddresses.usedAddresses') }} ({{ usedChangeAddresses.length }})
              <img class="icon" :class="{ open: showUsedChange }" :src="settingsStore.darkMode ? 'images/chevron-square-down-lightGrey.svg' : 'images/chevron-square-down.svg'">
            </td>
          </tr>
        </tbody>
        <tbody v-if="showUsedChange" class="used-addresses">
          <tr v-for="row in usedChangeAddresses" :key="row.index">
            <td class="mono">{{ row.index }}</td>
            <td @click="copyToClipboard(displayAddress(row))" class="address-cell" :title="displayAddress(row)">
              <span class="mono">{{ truncateAddress(displayAddress(row)) }}</span>
              <img class="copyIcon" src="images/copyGrey.svg">
            </td>
            <td class="mono">{{ satsToBch(row.balance) }}</td>
            <td>{{ row.txCount }}</td>
          </tr>
        </tbody>
        <!-- Unused change addresses -->
        <tbody>
          <tr v-for="row in unusedChangeAddresses" :key="row.index">
            <td class="mono">{{ row.index }}</td>
            <td @click="copyToClipboard(displayAddress(row))" class="address-cell" :title="displayAddress(row)">
              <span class="mono">{{ truncateAddress(displayAddress(row)) }}</span>
              <img class="copyIcon" src="images/copyGrey.svg">
            </td>
            <td class="mono">{{ satsToBch(row.balance) }}</td>
            <td>{{ row.txCount }}</td>
          </tr>
        </tbody>
      </table>
      <div v-else class="description">{{ t('hdAddresses.noAddresses') }}</div>
    </details>
  </fieldset>
</template>

<style scoped>
.filter-toggles {
  display: flex;
  flex-wrap: wrap;
  gap: 10px 20px;
  margin-bottom: 10px;
}

.filter-toggle {
  display: flex;
  align-items: center;
  gap: 5px;
}

.collapsible-section {
  margin-bottom: 15px;
}

.collapsible-section summary {
  cursor: pointer;
  user-select: none;
  display: flex;
  align-items: center;
  gap: 5px;
  margin-bottom: 5px;
}

.collapsible-section summary::-webkit-details-marker {
  display: none;
}

.collapsible-section summary::marker {
  display: none;
  content: '';
}

.collapsible-section[open] > summary .icon {
  transform: rotate(180deg);
}

.description {
  color: #888;
  margin: 5px 0 10px 0;
}

.address-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 10px;
  font-size: 13px;
}

.address-table th,
.address-table td {
  padding: 3px 6px;
  text-align: left;
  border-bottom: 1px solid var(--color-border, #ddd);
}

.address-table th {
  color: #888;
}

.mono {
  font-family: monospace;
}

.address-cell {
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
}

.section-toggle {
  cursor: pointer;
  user-select: none;
}

.section-toggle td {
  display: flex;
  align-items: center;
  gap: 5px;
  color: #888;
}

.section-toggle .icon.open {
  transform: rotate(180deg);
}

.icon {
  width: 16px;
  height: 16px;
}

.used-addresses tr {
  border-left: 3px solid #888;
}

.used-addresses tr td:first-child {
  padding-left: 2rem;
}

.dark .used-addresses tr {
  border-left-color: #555;
}

/* Dark mode */
.dark .description,
.dark .address-table th {
  color: #aaa;
}

.dark .address-table th,
.dark .address-table td {
  border-bottom-color: #444;
}
</style>
