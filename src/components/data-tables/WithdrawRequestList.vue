<template>
    <div class="withdraw-request-list-dt" @click="onClick">
        <template v-if="!withdrawRequestListError">
            <f-data-table
                :columns="columns"
                :items="dItems"
                first-m-v-column-width="6"
                fixed-header
                sticky0
                f-card-off
                no-f-l-padding
                caption="Withdraw Requests"
                class="f-data-table-body-bg-color"
            >
                <template v-slot:column-amount="{ value, item, column }">
                    <div v-if="column" class="row no-collapse no-vert-col-padding">
                        <div class="col-6 f-row-label">{{ column.label }}</div>
                        <div class="col break-word">
                            <f-token-value
                                :value="value"
                                :decimals="filtersOptions.fractionDigits"
                                :use-placeholder="false"
                                no-currency
                            />
                        </div>
                    </div>
                    <template v-else>
                        <f-token-value
                            :value="value"
                            :decimals="filtersOptions.fractionDigits"
                            :use-placeholder="false"
                            no-currency
                        />
                    </template>
                </template>

                <template v-slot:column-withdrawal="{ value, item, column }">
                    <div v-if="column" class="row no-collapse no-vert-col-padding">
                        <div class="col-6 f-row-label">{{ column.label }}</div>
                        <div class="col break-word">
                            <template v-if="value">{{ formatDate(timestampToDate(value), false, true) }}</template>
                            <template v-else>
                                <button
                                    :disabled="!!canNotWithdraw(item.createdTime) || !canWithdraw(item)"
                                    class="btn withdraw-btn"
                                    :data-item-id="item.id"
                                >
                                    {{ withdrawBtnLabel(item.createdTime) }}
                                </button>
                            </template>
                        </div>
                    </div>
                    <template v-else>
                        <template v-if="value">{{ formatDate(timestampToDate(value), false, true) }}</template>
                        <template v-else>
                            <button
                                :disabled="!!canNotWithdraw(item.createdTime) || !canWithdraw(item) || disableWithdraw"
                                class="btn withdraw-btn"
                                :data-item-id="item.id"
                            >
                                {{ withdrawBtnLabel(item.createdTime) }}
                            </button>
                        </template>
                    </template>
                </template>
            </f-data-table>
        </template>
        <template v-else>
            <div class="query-error">{{ withdrawRequestListError }}</div>
        </template>
    </div>
</template>

<script>
import { filtersOptions, formatDate, prepareTimestamp, timestampToDate } from '../../filters.js';
import { WEIToNEXT } from '../../utils/transactions.js';
import FDataTable from '../core/FDataTable/FDataTable.vue';
import dayjs from 'dayjs';
import { sortByHex } from '../../utils/array-sorting.js';
import FTokenValue from '@/components/core/FTokenValue/FTokenValue.vue';

/** Estimated time of block in seconds. */
const blockTime = 15 * 60;

export default {
    name: 'WithdrawRequestList',

    components: { FTokenValue, FDataTable },

    props: {
        /**
         * Data and action.
         * Actions:
         * '' - replace items
         * 'append' - append new items
         */
        items: {
            type: Array,
            default() {
                return [];
            },
        },
        disableWithdraw: {
            type: Boolean,
            default: false,
        },
    },

    data() {
        return {
            dItems: [],
            withdrawRequestListError: '',
            columns: [
                {
                    name: 'amount',
                    label: 'Amount (NEXT)',
                    formatter: (_value) => {
                        return WEIToNEXT(_value);
                    },
                    width: '234px',
                },
                {
                    name: 'undelegation_time',
                    label: 'Undelegation Time',
                    itemProp: 'createdTime',
                    formatter: (_value) => {
                        return formatDate(timestampToDate(_value), false, true);
                    },
                },
                {
                    name: 'withdrawal',
                    label: 'Withdrawal',
                    itemProp: 'withdrawTime',
                },
            ],
            sfcConfig: {},
            filtersOptions,
        };
    },

    computed: {
        pendingPartialUndelegations() {
            let count = 0;

            this.dItems.forEach((_item) => {
                if (!_item.withdrawBlock) {
                    count++;
                }
            });

            return count > 1;
        },
    },

    watch: {
        items: {
            handler(_value) {
                this.dItems = _value.sort(sortByHex('createdTime', 'desc'));
            },
            deep: true,
            immediate: true,
        },
    },

    async created() {
        this.sfcConfig = await this.$fWallet.getSFCConfig();
    },

    methods: {
        /**
         * Get label for withdraw button.
         *
         * @param {string} _timestamp
         * @return {string}
         */
        withdrawBtnLabel(_timestamp) {
            return this.canNotWithdraw(_timestamp) || 'Withdraw';
        },

        /**
         * Check if amount is available to withdraw.
         *
         * @param {string} _timestamp
         * @return {string}
         */
        canNotWithdraw(_timestamp) {
            const start = dayjs(this.prepareTimestamp(_timestamp)).utc();
            const { withdrawalPeriodTime } = this.sfcConfig;
            let end = withdrawalPeriodTime ? start.add(withdrawalPeriodTime.num + blockTime, 'second') : null; // start.add(7, 'days');
            const now = dayjs().utc();

            /*if (appConfig.useTestnet) {
                end = start.add(10, 'm');
            }*/

            if (end && now.diff(end) < 0) {
                return end.from(now);
            } else {
                return '';
            }
        },

        /**
         * Check if amount is available to withdraw.
         *
         * @param {object} _withdrawRequest
         * @return {boolean}
         */
        canWithdraw(_withdrawRequest) {
            return (!this.pendingPartialUndelegations || !_withdrawRequest.final) && !_withdrawRequest.withdrawBlock;
        },

        onClick(_event) {
            const eWithdrawBtn = _event.target.closest('.withdraw-btn');

            if (eWithdrawBtn) {
                const id = eWithdrawBtn.getAttribute('data-item-id');
                const withdrawRequest = this.dItems.find((_item) => _item.id === id);

                if (withdrawRequest && this.canWithdraw(withdrawRequest)) {
                    this.$emit('withdraw-request-selected', withdrawRequest);
                }
            }
        },

        formatDate,
        timestampToDate,
        prepareTimestamp,
    },
};
</script>
