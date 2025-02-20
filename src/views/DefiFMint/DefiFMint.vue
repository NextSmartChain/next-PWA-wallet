<template>
    <div class="view-defi-fmint">
        <h1 class="with-back-btn" data-focus>fMint</h1>

        <h2 class="perex">Manage your collateral and minted synths</h2>

        <section class="grid" aria-label="fMint info">
            <div>
                <h2>Collateral</h2>
                <div class="df-data-item smaller">
                    <h3 class="label">
                        <router-link :to="{ name: 'defi-ftrade' }"> Available {{ wnextTokenSymbol }} </router-link>
                        <template v-if="snextToken.address">+ {{ snextTokenSymbol }}</template>
                    </h3>
                    <div class="value">
                        <f-token-value :token="wnextToken" :value="availableWNEXT + availableSNEXT" no-currency />
                    </div>
                </div>
                <div class="df-data-item smaller">
                    <h3 class="label">
                        Locked {{ wnextTokenSymbol }}
                        <template v-if="snextToken.address"> + {{ snextTokenSymbol }}</template>
                    </h3>
                    <div class="value"><f-token-value :token="wnextToken" :value="collateral" no-currency /></div>
                </div>
                <div class="df-data-item smaller">
                    <h3 class="label">Current {{ wnextTokenSymbol }} price</h3>
                    <div class="value">
                        <f-token-value
                            :value="currentPrice"
                            :content-loaded="!!tokenPrice"
                            :decimals="5"
                            number-currency="USD"
                            no-currency
                        />
                    </div>
                </div>
            </div>
            <div class="limit-col align-center">
                <ratio-info :value="collateralRatio">
                    <template #ratio-info-title>
                        <h2>C-Ratio <c-ratio-info /></h2>
                    </template>
                </ratio-info>
            </div>
            <div class="align-end">
                <h2>Synths</h2>
                <div class="df-data-item smaller">
                    <h3 class="label">Max mintable</h3>
                    <div class="value"><f-token-value :token="fusdToken" :value="maxMintable" /></div>
                </div>
                <div class="df-data-item smaller">
                    <h3 class="label">Minted</h3>
                    <div class="value"><f-token-value :token="fusdToken" :value="debtFUSD" /></div>
                </div>
                <div class="df-data-item smaller">
                    <h3 class="label">Est. Pending / Stashed Rewards</h3>
                    <div class="value">
                        <f-placeholder
                            :content-loaded="!!wnextToken.symbol && 'rewardsEarned' in rewards"
                            replacement-text="999 / 999 wNEXT"
                        >
                            <f-token-value
                                no-currency
                                :use-placeholder="false"
                                :token="wnextToken"
                                :value="pendingRewardsWNEXT - stashedRewardsWNEXT"
                            />
                            <span class="currency-light">
                                /
                                <f-token-value
                                    :use-placeholder="false"
                                    :token="wnextToken"
                                    :value="stashedRewardsWNEXT"
                                />
                            </span>
                        </f-placeholder>
                    </div>
                </div>
                <!--
                <div class="df-data-item smaller">
                    <h3 class="label">Liquidation price</h3>
                    <div class="value">{{ liquidationPrice }}</div>
                </div>
                -->
            </div>
            <f-message v-if="closeToLiquidation" type="error" alert class="big">
                You're getting close to your liquidation price. <br />
                Please rebalance your collateral.
            </f-message>
        </section>

        <section class="form-buttons" aria-label="fMint actions">
            <div class="row">
                <div class="col align-start align-center-md">
                    <router-link :to="{ name: 'defi-lock', params: { token: { ...wnextToken } } }" class="btn large">
                        Lock Collateral
                    </router-link>
                    <br />
                    <router-link
                        :to="{ name: 'defi-unlock', params: { token: { ...wnextToken } } }"
                        class="btn large secondary"
                    >
                        Unlock Collateral
                    </router-link>
                </div>
                <div class="col">
                    <!--                    <router-link
                        v-if="canClaimRewards"
                        :to="{
                            name: 'defi-fmint-claim-rewards-confirmation',
                            params: { pendingRewards: pendingRewardsWNEXT, token: { ...wnextToken } },
                        }"
                        class="btn large"
                    >
                        Claim Rewards
                    </router-link>
                    <template v-else>
                        <button type="button" class="btn large" disabled>Claim Rewards</button>
                    </template>-->
                    <button
                        type="button"
                        class="btn large"
                        :disabled="!canClaimRewards"
                        @click="onClaimRewardsBtnClick"
                    >
                        Claim Rewards
                    </button>
                    <button
                        type="button"
                        class="btn large secondary"
                        :disabled="!canPushRewards"
                        @click="onPushRewardsBtnClick"
                    >
                        Push Rewards
                    </button>
                </div>
                <div class="col align-end align-center-md">
                    <button v-if="mintableTokens.length === 0" disabled class="btn large">Mint Synths</button>
                    <router-link v-else :to="{ name: 'defi-mint' }" class="btn large">Mint Synths</router-link>
                    <br />
                    <router-link :to="{ name: 'defi-repay' }" class="btn large secondary">Repay Synths</router-link>
                </div>
            </div>
            <!--
            <router-link
                :to="{ name: $defi.tmpWNEXT ? 'defi-lock-unlock' : 'defi-manage-collateral' }"
                class="btn large"
            >
                Lock/Unlock {{ wnextTokenSymbol }}
            </router-link>
            <router-link :to="{ name: $defi.tmpWNEXT ? 'defi-mint-repay' : 'defi-borrow-fusd' }" class="btn large">
                Mint/Repay fUSD
            </router-link>
            -->
        </section>

        <section aria-label="fMint positions">
            <f-tabs>
                <template #fmint-overview>
                    fMint Overview
                    <span class="f-records-count">({{ fMintOverviewRecordsCount }})</span>
                </template>
                <template #collateral-positions>
                    Collateral Positions
                    <span class="f-records-count">({{ collateralPositionsRecordsCount }})</span>
                </template>
                <template #synths-positions>
                    Synths Positions
                    <span class="f-records-count">({{ synthsPositionsRecordsCount }})</span>
                </template>
                <template #assets>
                    Assets
                    <span class="f-records-count">({{ assetsRecordsCount }})</span>
                </template>

                <f-tab title-slot="fmint-overview">
                    <h2 class="not-visible">fMint Overview - {{ fMintOverviewRecordsCount }} items</h2>
                    <f-mint-overview-list
                        :tokens="tokens"
                        deposit-route-name="defi-lock-unlock"
                        borrow-route-name="defi-mint-repay"
                        @records-count="onFMintOverviewRecordsCount"
                    />
                </f-tab>
                <f-tab title-slot="collateral-positions">
                    <h2 class="not-visible">Collateral Positions - {{ collateralPositionsRecordsCount }} items</h2>
                    <collateral-positions-list
                        :tokens="tokens"
                        :f-mint-account="fMintAccount"
                        deposit-route-name="defi-lock-unlock"
                        borrow-route-name="defi-mint-repay"
                        @records-count="onCollateralPositionsRecordsCount"
                    />
                </f-tab>
                <f-tab title-slot="synths-positions">
                    <h2 class="not-visible">Synths Positions - {{ synthsPositionsRecordsCount }} items</h2>
                    <synths-positions-list
                        :tokens="tokens"
                        :f-mint-account="fMintAccount"
                        deposit-route-name="defi-lock-unlock"
                        borrow-route-name="defi-mint-repay"
                        @records-count="onSynthsPositionsRecordsCount"
                    />
                </f-tab>
                <f-tab title-slot="assets">
                    <h2 class="not-visible">Assets- {{ assetsRecordsCount }} items</h2>
                    <assets-list
                        defi-assets-list
                        :tokens="tokens"
                        :f-mint-account="fMintAccount"
                        @records-count="onAssetsRecordsCount"
                    />
                </f-tab>
            </f-tabs>
        </section>

        <tx-confirmation-window
            ref="confirmationWindow"
            body-min-height="350px"
            :steps-count="1"
            :active-step="1"
            :window-title="windowTitle"
            @cancel-button-click="onCancelButtonClick"
        />

        <!--
        <defi-menu v-else>
            <li class="col-4">
                <div class="menu-item disabled">
                    <h2>Manage collateral</h2>
                    <div class="icon">
                        <icon data="@/assets/svg/defi/mint.svg" width="96" height="96" />
                    </div>
                    <p class="description">Lock NEXT to use it as collateral and mint fUSD</p>
                    <div class="title">Coming Soon</div>
                </div>
            </li>
            <li class="col-4">
                <div class="menu-item" tabindex="0">
                    <h2>Mint</h2>
                    <div class="icon">
                        <icon data="@/assets/svg/defi/mint.svg" width="96" height="96" />
                    </div>
                    <p class="description">Mint fUSD by locking your NEXT</p>
                    <router-link :to="{ name: 'defi-borrow-fusd' }" class="clickable title">
                        Mint fUSD
                    </router-link>
                </div>
            </li>
            <li class="col-4">
                <div class="menu-item disabled" tabindex="0">
                    <h2>Repay</h2>
                    <div class="icon">
                        <icon data="@/assets/svg/defi/repay.svg" width="96" height="96" />
                    </div>
                    <p class="description">Repay the fUSD you minted and unlock your NEXT</p>
                    <div class="title">Coming Soon</div>
                </div>
            </li>
        </defi-menu>
        -->
    </div>
</template>

<script>
import { mapGetters } from 'vuex';
import FMessage from '../../components/core/FMessage/FMessage.vue';
import { getUniqueId } from '../../utils';
import { eventBusMixin } from '../../mixins/event-bus.js';
import FTokenValue from '@/components/core/FTokenValue/FTokenValue.vue';
import FPlaceholder from '@/components/core/FPlaceholder/FPlaceholder.vue';
import RatioInfo from '@/components/RatioInfo/RatioInfo.vue';
import CRatioInfo from '@/components/CRatioInfo/CRatioInfo.vue';
import FMintOverviewList from '@/components/data-tables/FMintOverviewList/FMintOverviewList.vue';
import FTabs from '@/components/core/FTabs/FTabs.vue';
import FTab from '@/components/core/FTabs/FTab.vue';
import CollateralPositionsList from '@/components/data-tables/CollateralPositionsList/CollateralPositionsList.vue';
import SynthsPositionsList from '@/components/data-tables/SynthsPositionsList/SynthsPositionsList.vue';
import AssetsList from '@/components/data-tables/AssetsList/AssetsList.vue';
import TxConfirmationWindow from '@/components/windows/TxConfirmationWindow/TxConfirmationWindow.vue';
import { focusElem } from '@/utils/aria.js';

export default {
    name: 'DefiFMint',

    components: {
        TxConfirmationWindow,
        AssetsList,
        SynthsPositionsList,
        CollateralPositionsList,
        FTab,
        FTabs,
        FMintOverviewList,
        CRatioInfo,
        RatioInfo,
        FPlaceholder,
        FTokenValue,
        FMessage,
    },

    mixins: [eventBusMixin],

    data() {
        return {
            tokenPrice: 0,
            /** @type {FMintAccount} */
            fMintAccount: {
                collateral: [],
                debt: [],
            },
            /** @type {FMintAccount} */
            rewards: {},
            /** @type {DefiToken} */
            wnextToken: {},
            /** @type {DefiToken} */
            snextToken: {},
            /** @type {DefiToken} */
            fusdToken: {},
            /** @type {DefiToken[]} */
            tokens: [],
            /** @type {DefiToken[]} */
            mintableTokens: [],
            fMintOverviewRecordsCount: 0,
            collateralPositionsRecordsCount: 0,
            synthsPositionsRecordsCount: 0,
            assetsRecordsCount: 0,
            windowTitle: '',
            id: getUniqueId(),
        };
    },

    computed: {
        ...mapGetters(['currentAccount', 'defiSlippageReserve']),

        debt() {
            // overall debt
            return this.$defi.fromTokenValue(this.fMintAccount.debtValue, this.fusdToken);
        },

        debtFUSD() {
            return this.tokens.reduce((_prev, _token) => {
                return _prev + this.$defi.convertTokenValue(this.getDebt(_token), _token, this.fusdToken);
            }, 0);
        },

        collateral() {
            /** @type {FMintTokenBalance} */
            const wNEXTtokenBalance = this.$defi.getFMintAccountCollateral(this.fMintAccount, this.wnextToken);
            /** @type {FMintTokenBalance} */
            const sNEXTtokenBalance = this.$defi.getFMintAccountCollateral(this.fMintAccount, this.snextToken);
            const wNEXTtokenBalanceValue = this.$defi.fromTokenValue(wNEXTtokenBalance.balance, this.wnextToken) || 0;
            const sNEXTtokenBalanceValue = this.$defi.fromTokenValue(sNEXTtokenBalance.balance, this.snextToken) || 0;

            return wNEXTtokenBalanceValue + sNEXTtokenBalanceValue;
        },

        availableWNEXT() {
            return this.wnextToken ? this.$defi.fromTokenValue(this.wnextToken.availableBalance, this.wnextToken) || 0 : 0;
        },

        availableSNEXT() {
            return this.snextToken ? this.$defi.fromTokenValue(this.snextToken.availableBalance, this.snextToken) || 0 : 0;
        },

        currentPrice() {
            return this.tokenPrice;
            // return formatNumberByLocale(this.tokenPrice, 5, 'USD');
        },

        pendingRewards() {
            return this.$defi.fromTokenValue(this.rewards.rewardsEarned, this.fusdToken) || 0;
        },

        pendingRewardsWNEXT() {
            return this.$defi.fromTokenValue(this.rewards.rewardsEarned, this.wnextToken) || 0;
            // return this.$defi.convertTokenValue(this.pendingRewards, this.fusdToken, this.wnextToken);
        },

        stashedRewards() {
            return this.$defi.fromTokenValue(this.rewards.rewardsStashed, this.fusdToken) || 0;
        },

        stashedRewardsWNEXT() {
            return this.$defi.fromTokenValue(this.rewards.rewardsStashed, this.wnextToken) || 0;
            // return this.$defi.convertTokenValue(this.stashedRewards, this.fusdToken, this.wnextToken);
        },

        canClaimRewards() {
            const { rewards } = this;

            return rewards.canClaimRewards && (rewards.rewardsEarned !== '0x0' || rewards.rewardsStashed !== '0x0');
        },

        canPushRewards() {
            return this.rewards.canPushNewRewards;
            // return true;
        },

        liquidationPrice() {
            return '-';
            /*
            const liqPrice = this.$defi.getLiquidationPrice(this.debt, this.collateral);

            return liqPrice > 0 ? formatNumberByLocale(liqPrice, 5, 'USD') : '-';
            */
        },

        availableBalance() {
            return this.wnextToken ? this.$defi.fromTokenValue(this.wnextToken.availableBalance, this.wnextToken) || 0 : 0;
        },

        maxMintable() {
            const borrowLimit = this.$defi.getBorrowLimit(this.fMintAccount);

            return this.debtFUSD + borrowLimit - borrowLimit * this.defiSlippageReserve;
            /*
            return (
                this.debtFUSD +
                Math.min(
                    this.availableBalance * this.tokenPrice,
                    this.$defi.getBorrowLimit(this.fMintAccount) / this.tokenPrice
                )
            );
            */
            // return this.$defi.getMaxDebt(this.collateral, this.tokenPrice).toFixed(2);
        },

        debtLimit() {
            return this.$defi.getDebtLimit(this.fMintAccount);
        },

        collateralRatio() {
            return this.$defi.getCollateralRatio(this.fMintAccount);
        },

        closeToLiquidation() {
            return false;
            /*
            const { $defi } = this;

            return this.debtLimit > ($defi.warningCollateralRatio / $defi.minCollateralRatio) * 100;
            */
        },

        wnextTokenSymbol() {
            return this.$defi.getTokenSymbol(this.wnextToken);
        },

        snextTokenSymbol() {
            return this.$defi.getTokenSymbol(this.snextToken);
        },

        /**
         * Property is set to `true`, if 'small' breakpoint is reached.
         *
         * @return {Boolean}
         */
        /*
        smallView() {
            const breakpoint = this.$store.state.breakpoints['small'];

            return breakpoint && breakpoint.matches;
        },
        */
    },

    created() {
        this._eventBus.on('account-picked', this.onAccountPicked);
        this._eventBus.on('claim-mint-rewards', this.onClaimMintRewards);

        this.init();
    },

    mounted() {
        focusElem(this.$el);
    },

    methods: {
        async init() {
            const { $defi } = this;
            const { address } = this.currentAccount;
            const result = await Promise.all([
                $defi.fetchFMintAccount(address, true),
                $defi.fetchTokens(address),
                $defi.init(),
            ]);

            this.fMintAccount = result[0];
            this.tokens = result[1];
            this.fusdToken = this.tokens.find((_item) => _item.symbol === 'FUSD') || {};
            this.wnextToken = this.tokens.find((_item) => _item.symbol === 'WNEXT') || {};
            this.snextToken = this.tokens.find((_item) => _item.symbol === 'SNEXT') || {};

            // this.tokenPrice = $defi.getTokenPrice(this.wnextToken);
            this.tokenPrice = await this.$fWallet.getTokenPrice();

            this.mintableTokens = this.tokens.filter($defi.canTokenBeMinted);

            this.rewards = await $defi.fetchFMintAccountRewards(address);
        },

        /**
         * @param {DefiToken} _token
         * @return {number}
         */
        getDebt(_token) {
            /** @type {FMintTokenBalance} */
            const tokenBalance = this.$defi.getFMintAccountDebt(this.fMintAccount, _token);

            return this.$defi.fromTokenValue(tokenBalance.balance, _token) || 0;
        },

        onAccountPicked() {
            this.init();
        },

        onFMintOverviewRecordsCount(_count) {
            this.fMintOverviewRecordsCount = _count;
        },

        onCollateralPositionsRecordsCount(_count) {
            this.collateralPositionsRecordsCount = _count;
        },

        onSynthsPositionsRecordsCount(_count) {
            this.synthsPositionsRecordsCount = _count;
        },

        onAssetsRecordsCount(_count) {
            this.assetsRecordsCount = _count;
        },

        onClaimMintRewards(_data) {
            if (this.canClaimRewards) {
                this.windowTitle = 'Claim Rewards';
                this.$refs.confirmationWindow.changeComponent('defi-f-mint-claim-rewards-confirmation', {
                    params: _data,
                });
                this.$refs.confirmationWindow.show();
            }
        },

        onClaimRewardsBtnClick() {
            if (this.canClaimRewards) {
                this.windowTitle = 'Claim Rewards';
                this.$refs.confirmationWindow.changeComponent('defi-f-mint-claim-rewards-confirmation', {
                    params: { pendingRewards: this.pendingRewardsWNEXT, token: { ...this.wnextToken } },
                });
                this.$refs.confirmationWindow.show();
            }
        },

        onPushRewardsBtnClick() {
            if (this.canPushRewards) {
                this.windowTitle = 'Push Rewards';
                this.$refs.confirmationWindow.changeComponent('defi-f-mint-push-rewards-confirmation', {
                    params: { token: { ...this.wnextToken } },
                });
                this.$refs.confirmationWindow.show();
            }
        },

        onCancelButtonClick() {
            this.$emit('reload-view');
        },
    },
};
</script>

<style lang="scss">
@import 'style';
</style>
