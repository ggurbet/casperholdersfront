<template>
  <div class="operations">
    <v-data-table
      :headers="operationHeaders"
      :items="operations"
      :server-items-length="totalOperations"
      :single-expand="true"
      item-key="hash"
      show-expand
      :options.sync="options"
      :loading="loading"
      :footer-props="{
        itemsPerPageOptions: [5,10,15]
      }"
    >
      <template #top>
        <v-toolbar
          flat
          rounded
        >
          <v-toolbar-title>Operations</v-toolbar-title>
        </v-toolbar>
        <v-container>
          <v-autocomplete
            v-model="typesSelected"
            item-color="white"
            :items="types"
            item-text="text"
            item-value="value"
            chips
            label="Deploy types"
            color="white"
            multiple
            deletable-chips
          />
        </v-container>
      </template>
      <template #[`item.hash`]="{ item }">
        <a
          :href="deployUrl(item.hash)"
          target="_blank"
          rel="noopener"
        >
          {{ truncate(item.hash) }}
        </a>
      </template>
      <template #[`item.from`]="{ item }">
        <a
          :href="accountUrl(item.from)"
          target="_blank"
          rel="noopener"
        >
          {{ truncate(item.from) }}
        </a>
      </template>
      <template #[`item.cost`]="{ item }">
        {{ motesToCsprString(item.cost) }}
      </template>
      <template #[`item.result`]="{ item }">
        <template v-if="item.result === true">
          <v-icon color="green">
            mdi-checkbox-marked-circle
          </v-icon>
        </template>
        <template v-else>
          <v-icon color="red">
            mdi-alert-circle
          </v-icon>
        </template>
      </template>
      <template #[`item.timestamp`]="{ item }">
        {{ localeTimestamp(item.timestamp) }}
      </template>
      <template #[`item.block`]="{ item }">
        <a
          :href="blockUrl(item.block)"
          target="_blank"
          rel="noopener"
        >
          {{ truncate(item.block) }}
        </a>
      </template>
      <template #[`item.type`]="{ item }">
        {{ capitalizeFirstLetter(item.metadata_type) }}
      </template>
      <template
        #expanded-item="{ headers, item }"
      >
        <td :colspan="headers.length">
          <v-card-title v-if="item.metadata">
            Metadata
          </v-card-title>
          <v-simple-table v-if="item.metadata">
            <template #default>
              <tbody>
                <tr>
                  <th scope="col">
                    Argument
                  </th>
                  <th scope="col">
                    Value
                  </th>
                </tr>
                <template v-for="[key, value] in Object.entries(item.metadata)">
                  <tr :key="key">
                    <td>
                      {{ capitalizeFirstLetter(key) }}
                    </td>
                    <td v-if="key === 'amount'">
                      {{ motesToCsprString(value) }}
                    </td>
                    <td v-else-if="key === 'hash'">
                      <a
                        :href="deployUrl(value)"
                        target="_blank"
                        rel="noopener"
                      >
                        {{ value.toString() }}
                      </a>
                    </td>
                    <td v-else-if="key === 'validator'">
                      <a
                        :href="validatorUrl(value)"
                        target="_blank"
                        rel="noopener"
                      >
                        {{ value.toString() }}
                      </a>
                    </td>
                    <td v-else-if="['from', 'delegator', 'target'].indexOf(key) !== -1">
                      <a
                        :href="accountUrl(value)"
                        target="_blank"
                        rel="noopener"
                      >
                        {{ value.toString() }}
                      </a>
                    </td>
                    <td v-else-if="typeof value === 'object'">
                      <template v-for="[key, value] in Object.entries(value)">
                        <div>
                          {{key}} : {{value}}
                        </div>
                      </template>
                    </td>
                    <td v-else>
                      {{ value.toString() }}
                    </td>
                  </tr>
                </template>
              </tbody>
            </template>
          </v-simple-table>
          <v-card-title v-else>
            No metadata
          </v-card-title>
          <template v-if="item.events">
            <v-card-title>
              Events
            </v-card-title>
            <v-expansion-panels>
              <template v-for="[eventName, event] in Object.entries(item.events)">
                <v-expansion-panel>
                  <v-expansion-panel-header>
                    {{ eventName }}
                  </v-expansion-panel-header>
                  <v-expansion-panel-content>
                    <v-simple-table>
                      <template #default>
                        <tbody>
                          <tr>
                            <th scope="col">
                              Argument
                            </th>
                            <th scope="col">
                              Value
                            </th>
                          </tr>
                          <template v-for="[eventArgName, eventArgValue] in Object.entries(event)">
                            <tr>
                              <td>{{eventArgName}}</td>
                              <td>{{eventArgValue}}</td>
                            </tr>
                          </template>
                        </tbody>
                      </template>
                    </v-simple-table>
                  </v-expansion-panel-content>
                </v-expansion-panel>
              </template>
            </v-expansion-panels>
          </template>

        </td>
      </template>
    </v-data-table>
  </div>
</template>

<script>

import { CSPR_LIVE_URL, DATA_API } from '@/helpers/env';
import { CurrencyUtils } from '@casperholders/core';
import { mapState } from 'vuex';

/**
 * Operations component. List all operations of a user
 */
export default {
  name: 'Operations',
  data() {
    return {
      operationHeaders: [
        { text: 'Hash', value: 'hash', sortable: false },
        { text: 'From', value: 'from', sortable: false },
        { text: 'Cost', value: 'cost' },
        { text: 'Result', value: 'result' },
        { text: 'Timestamp', value: 'timestamp' },
        { text: 'Block', value: 'block', sortable: false },
        { text: 'Type', value: 'type' },
        { text: '', value: 'data-table-expand', sortable: false },
      ],
      types: [
        { text: 'Transfer', value: 'transfer' },
        { text: 'Simple Transfer', value: 'simpleTransfer' },
        { text: 'Add Bid', value: 'addBid' },
        { text: 'Withdraw Bid', value: 'withdrawBid' },
        { text: 'Delegate', value: 'delegate' },
        { text: 'Undelegate', value: 'undelegate' },
        { text: 'Unknown', value: 'unknown' },
        { text: 'Activate Bid', value: 'activateBid' },
        { text: 'Account Info', value: 'accountInfo' },
        { text: 'Wasm Deploy', value: 'wasmDeploy' },
        { text: 'Key Management', value: 'keyManagement' },
        { text: 'Key Weight', value: 'keyWeight' },
        { text: 'Key Management Threshold', value: 'keyManagementThreshold' },
        { text: 'CasperSign Contract', value: 'casperSignContract' },
        { text: 'ERC20', value: 'ERC20' },
        { text: 'Faucet', value: 'faucet' },
      ],
      operations: [],
      options: {
        sortBy: ['timestamp'],
        sortDesc: [true],
      },
      loading: true,
      totalOperations: 0,
      typesSelected: [],
    };
  },
  computed: {
    ...mapState(['signer']),
  },
  watch: {
    /**
     * Watch the state of the active key. In case of an update, re-fetch the balance data
     */
    'signer.activeKey': {
      async handler() {
        await this.getDataFromApi();
      },
      immediate: true,
    },
    options: {
      async handler() {
        await this.getDataFromApi();
      },
    },
    typesSelected: {
      async handler() {
        await this.getDataFromApi();
      },
    },
  },
  methods: {
    async getDataFromApi() {
      if (!this.signer.activeKey) {
        this.loading = false;
        return;
      }
      this.loading = true;
      let { page, limit } = this.options;
      if (!limit) {
        limit = 10;
      }
      if (!page) {
        page = 1;
      }
      const offset = (page - 1) * limit;
      let order = '';
      if (this.options.sortBy[0]) {
        order = `&order=${this.options.sortBy[0]}`;
        if (this.options.sortDesc[0]) {
          order += '.desc';
        }
      }
      let where = '';
      if (this.typesSelected.length > 0) {
        where = `&metadata_type=in.("${this.typesSelected.join('","')}")`;
      }
      const response = await fetch(`${DATA_API}/deploys?from=ilike.${this.signer.activeKey}&limit=${limit}&offset=${offset}${order}${where}`, {
        method: 'GET',
        headers: new Headers({
          'Range-Unit': 'items',
          Prefer: 'count=exact',
        }),
      });
      this.totalOperations = Number(response.headers.get('content-range').replace(/^.*\//, ''));
      this.operations = await response.json();
      this.loading = false;
    },
    truncate(fullStr) {
      const strLen = 15;
      const separator = '...';

      if (fullStr.length <= strLen) return fullStr;

      const sepLen = separator.length;
      const charsToShow = strLen - sepLen;
      const frontChars = Math.ceil(charsToShow / 2);
      const backChars = Math.floor(charsToShow / 2);

      return fullStr.substr(0, frontChars) + separator + fullStr.substr(fullStr.length - backChars);
    },
    parseData(data) {
      let parsed = '';
      for (let i = 0; i < Object.entries(data).length; i++) {
        const [key, value] = Object.entries(data)[i];
        parsed += `${this.capitalizeFirstLetter(key)} : ${value}\n`;
      }
      return parsed;
    },
    motesToCsprString(amount) {
      return `${CurrencyUtils.convertMotesToCasper(amount)} CSPR`;
    },
    capitalizeFirstLetter(string) {
      return string.charAt(0).toUpperCase() + string.slice(1);
    },
    remove(item) {
      const index = this.typesSelected.indexOf(item.name);
      if (index >= 0) this.typesSelected.splice(index, 1);
    },
    validatorUrl(value) {
      return `${CSPR_LIVE_URL}validator/${value}`;
    },
    accountUrl(value) {
      return `${CSPR_LIVE_URL}account/${value}`;
    },
    deployUrl(value) {
      return `${CSPR_LIVE_URL}deploy/${value}`;
    },
    blockUrl(value) {
      return `${CSPR_LIVE_URL}block/${value}`;
    },
    localeTimestamp(value) {
      const date = new Date(value);
      return `${date.toLocaleDateString()} ${date.toLocaleTimeString()}`;
    },
  },
};
</script>
