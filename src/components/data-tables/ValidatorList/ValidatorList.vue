<template>
    <div class="validator-list-dt" @click="onClick">
        <template v-if="!dValidatorListError">
            <f-data-table
                :columns="dColumns"
                :items="dItems"
                :mobile-view="cMobileView"
                :loading="cLoading"
                first-m-v-column-width="6"
                fixed-header
                f-card-off
                caption="Validators"
                class="f-data-table-body-bg-color"
            >
                <template v-slot:column-logo="{ value, item, column }">
                    <div v-if="column" class="row no-collapse no-vert-col-padding">
                        <div class="col-6 f-row-label">{{ column.label }}</div>
                        <div class="col break-word">
                            <div class="inline-img validator-img">
                                <img v-if="value" :src="value" :alt="item.stakerInfo.name" class="not-fluid" />
                                <img v-else src="img/next-logo.png" alt="next logo" class="not-fluid" />
                            </div>
                        </div>
                    </div>
                    <template v-else>
                        <div class="inline-img validator-img">
                            <img v-if="value" :src="value" :alt="item.stakerInfo.name" class="not-fluid" />
                            <img v-else src="img/next-logo.png" alt="next logo" class="not-fluid" />
                        </div>
                    </template>
                </template>

                <template v-slot:column-name="{ value, item, column }">
                    <div v-if="column" class="row no-collapse no-vert-col-padding">
                        <div class="col-6 f-row-label">{{ column.label }}</div>
                        <div class="col break-word">
                            <a
                                :href="`${explorerUrl}validator/${item.stakerAddress}`"
                                target="_blank"
                                rel="noopener"
                                class="break-word"
                            >
                                {{ value }}
                            </a>
                            <a
                                v-if="(item.stakerInfo ? item.stakerInfo.website || item.stakerInfo.contact : '')"
                                :href="(item.stakerInfo ? item.stakerInfo.website || item.stakerInfo.contact : '')"
                                target="_blank"
                                rel="noopener"
                                class="validator-website"
                            >
                                <icon data="@/assets/svg/external-link-alt.svg"></icon>
                            </a>
                        </div>
                    </div>
                    <template v-else>
                        <a
                            :href="`${explorerUrl}validator/${item.stakerAddress}`"
                            rel="noopener"
                            target="_blank"
                            class="break-word"
                        >
                            {{ value }}
                        </a>
                        <a
                            v-if="(item.stakerInfo ? item.stakerInfo.website || item.stakerInfo.contact : '')"
                            :href="(item.stakerInfo ? item.stakerInfo.website || item.stakerInfo.contact : '')"
                            target="_blank"
                            rel="noopener"
                            class="validator-website"
                        >
                            <icon data="@/assets/svg/external-link-alt.svg"></icon>
                        </a>
                    </template>
                </template>

                <template v-slot:column-select="{ value, item, column }">
                    <div v-if="column" class="row no-collapse no-vert-col-padding">
                        <div class="col break-word align-center select-validator-mv">
                            <button
                                :disabled="item.isOffline || item.isCheater || item.alreadyDelegated"
                                class="btn select-btn"
                                :data-validator-id="value"
                            >
                                Select
                            </button>
                        </div>
                    </div>
                    <template v-else>
                        <button
                            :disabled="item.isOffline || item.isCheater || item.alreadyDelegated"
                            class="btn select-btn"
                            :data-validator-id="value"
                        >
                            Select
                        </button>
                    </template>
                </template>

                <!--
                <template v-slot:column-stakerAddress="{ value, item, column }">
                    <div v-if="column" class="row no-collapse no-vert-col-padding">
                        <div class="col-6 f-row-label">{{ column.label }}</div>
                        <div class="col break-word">
                            <div v-if="item.isOffline" class="offline">{{ $t('view_validator_list.offline') }}</div>
                            <router-link
                                :to="{ name: 'validator-detail', params: { address: value } }"
                                :title="value"
                                >{{ value | formatHash }}</router-link
                            >
                        </div>
                    </div>
                    <template v-else>
                        <div v-if="item.isOffline" class="offline">{{ $t('view_validator_list.offline') }}</div>
                        <router-link :to="{ name: 'validator-detail', params: { address: value } }" :title="value">{{
                            value | formatHash
                        }}</router-link>
                    </template>
                </template>
                -->
            </f-data-table>
        </template>

        <template v-else>
            <div class="query-error">{{ dValidatorListError }}</div>
        </template>
    </div>
</template>

<script>
import FDataTable from '../../core/FDataTable/FDataTable.vue';
import gql from 'graphql-tag';
import { WEIToNEXT } from '../../../utils/transactions.js';
import { formatHexToInt, timestampToDate, numToFixed, formatNumberByLocale } from '../../../filters.js';
import { sortByHex, sortByLocaleString } from '../../../utils/array-sorting.js';
import appConfig from '../../../../app.config.js';
import { cloneObject } from '@/utils';
import { mapGetters } from 'vuex';
import { shuffle } from '@/utils/array.js';

export default {
    name: 'ValidatorList',

    components: {
        FDataTable,
    },

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
        /** Component is used in FWindow. */
        windowMode: {
            type: Boolean,
            default: false,
        },
    },

    apollo: {
        stakers: {
            query: gql`
                query Stakers {
                    stakers {
                        id
                        stakerAddress
                        isCheater
                        createdTime
                        stake
                        totalStake
                        delegatedMe
                        isOffline
                        isCheater
                        validatorInfo {
                            name
                            website
                            contact
                            logoUrl
                        }
                    }
                }
            `,
            async result(_data, _key) {
                const totals = {
                    selfStaked: 0,
                    totalDelegated: 0,
                    totalStaked: 0,
                };
                let data;
                const flagged = [];
                const tUnknown = this.$t('view_validator_list.unknown');

                if (_key === 'stakers') {
                    data = cloneObject(_data.data.stakers);

                    // exclude validators with zero self-stake
                    data = data.filter((validator) => validator.stake !== '0x0');

                    shuffle(data);

                    data.forEach((_item, _idx) => {
                        // _item.total_staked = WEIToNEXT(_item.stake) + WEIToNEXT(_item.delegatedMe);
                        totals.selfStaked += parseFloat(numToFixed(WEIToNEXT(_item.stake), 0));
                        totals.totalDelegated += parseFloat(numToFixed(WEIToNEXT(_item.delegatedMe), 0));
                        totals.totalStaked += parseFloat(numToFixed(WEIToNEXT(_item.totalStake), 0));

                        if (!_item.stakerInfo) {
                            _item.stakerInfo = {};
                        }

                        if (!_item.stakerInfo.name) {
                            _item.stakerInfo.name = tUnknown;
                        }

                        if (_item.isCheater) {
                            flagged.push(_idx);
                        }

                        _item.alreadyDelegated = true;
                    });

                    if (flagged.length > 0) {
                        for (let i = flagged.length - 1; i >= 0; i--) {
                            flagged[i] = data.splice(flagged[i], 1)[0];
                        }

                        this.$emit('validator-list-flagged', flagged);
                    }

                    this.dItems = data;

                    this.$emit('records-count', this.dItems.length);
                    this.$emit('validator-list-totals', totals);

                    const stakerIds = await this.fetchStakerIds();

                    if (stakerIds.length > 0) {
                        data.forEach((_item) => {
                            if (stakerIds.indexOf(_item.id) === -1) {
                                _item.alreadyDelegated = false;
                            }
                        });
                    } else {
                        data.forEach((_item) => {
                            _item.alreadyDelegated = false;
                        });
                    }
                }
            },
            skip() {
                return this.items.length > 0;
            },
            error(_error) {
                this.dValidatorListError = _error.message;
            },
        },
    },

    data() {
        return {
            explorerUrl: appConfig.explorerUrl2,
            dItems: [],
            dValidatorListError: '',
            dColumns: [
                {
                    name: 'logo',
                    label: this.$t('view_validator_list.logo'),
                    itemProp: 'stakerInfo.logoUrl',
                    cssClass: 'align-center',
                    width: '80px',
                },
                {
                    name: 'name',
                    label: this.$t('view_validator_list.name'),
                    itemProp: 'stakerInfo.name',
                    sortFunc: sortByLocaleString,
                    width: '200px',
                },
                {
                    name: 'id',
                    label: this.$t('view_validator_list.id'),
                    formatter: formatHexToInt,
                    sortFunc: sortByHex,
                    width: '80px',
                },
                /*
                {
                    name: 'stakerAddress',
                    label: this.$t('view_validator_list.address'),
                    sortFunc: sortByString,
                    width: '200px',
                },
                */
                {
                    name: 'stake',
                    label: this.$t('view_validator_list.self_staked'),
                    formatter: (_value) => formatNumberByLocale(WEIToNEXT(_value), 0),
                    sortFunc: sortByHex,
                    cssClass: 'align-end',
                },
                {
                    name: 'delegatedMe',
                    label: this.$t('view_validator_list.delegated'),
                    formatter: (_value) => formatNumberByLocale(WEIToNEXT(_value), 0),
                    sortFunc: sortByHex,
                    cssClass: 'align-end',
                },
                // computed
                {
                    name: 'totalStake',
                    label: this.$t('view_validator_list.total_staked'),
                    formatter: (_value) => formatNumberByLocale(WEIToNEXT(_value), 0),
                    sortFunc: sortByHex,
                    // sortDir: 'desc',
                    cssClass: 'align-end',
                },
                {
                    name: 'select',
                    // label: this.$t('view_validator_list.logo'),
                    itemProp: 'id',
                    cssClass: 'align-center',
                    width: '140px',
                },
            ],
        };
    },

    computed: {
        ...mapGetters(['currentAccount']),

        /**
         * Property is set to `true`, if 'tvalidator-list-dt-mobile-view' breakpoint is reached.
         *
         * @return {Boolean}
         */
        cMobileView() {
            const dataTableBreakpoint = this.$store.state.breakpoints['validator-list-dt-mobile-view'];

            return dataTableBreakpoint && dataTableBreakpoint.matches;
        },

        cLoading() {
            return this.loading || this.$apollo.queries.stakers.loading;
        },
    },

    created() {
        if (this.items.length > 0) {
            this.dItems = this.items;
        }
    },

    methods: {
        /**
         * @return {Promise<[]>}
         */
        async fetchStakerIds() {
            const delegations = await this.$fWallet.fetchAll(
                {
                    query: gql`
                        query DelegationsByAddress($address: Address!, $cursor: Cursor, $count: Int!) {
                            delegationsByAddress(address: $address, cursor: $cursor, count: $count) {
                                pageInfo {
                                    first
                                    last
                                    hasNext
                                    hasPrevious
                                }
                                totalCount
                                edges {
                                    cursor
                                    delegation {
                                        toStakerId
                                        amount
                                    }
                                }
                            }
                        }
                    `,
                    variables: {
                        address: this.currentAccount.address,
                        count: 100,
                        cursor: null,
                    },
                    fetchPolicy: 'network-only',
                },
                'delegationsByAddress'
            );

            return delegations
                .filter((_item) => _item.delegation && _item.delegation.amount !== '0x0' && _item.delegation.toStakerId)
                .map((_item) => _item.delegation.toStakerId);
            // return delegations.map((_item) => (_item.delegation ? _item.delegation.toStakerId : ''));
        },

        onClick(_event) {
            const eSelectBtn = _event.target.closest('.select-btn');

            if (eSelectBtn) {
                const id = eSelectBtn.getAttribute('data-validator-id');
                const validator = id ? this.dItems.find((_item) => _item.id === id) : null;

                if (validator) {
                    this.$emit('validator-selected', {
                        id: validator.id,
                        address: validator.stakerAddress,
                        name: validator.stakerInfo.name,
                    });
                }
            }
        },

        WEIToNEXT,
        timestampToDate,
        numToFixed,
    },
};
</script>

<style lang="scss">
@import 'style';
</style>
