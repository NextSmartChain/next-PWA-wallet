<template>
    <div class="view-defi-mint-repay">
        <h1 class="with-back-btn" data-focus>
            Mint/Repay fUSD
            <f-back-button :route-name="backButtonRoute" />
        </h1>

        <h2 class="perex">
            Mint fUSD with your locked wNEXT or repay fUSD to unlock your wNEXT. You can use fUSD to trade synths, lend
            to the liquidity pool to earn interest, and use it as a collateral to borrow synths.
        </h2>

        <defi-borrow
            :token="params.token"
            single-token
            mint-repay-mode
            token-symbol="FUSD"
            on-submit-route="defi-mint-repay-confirmation"
        />
    </div>
</template>

<script>
import FBackButton from '@/components/core/FBackButton/FBackButton.vue';
import { getAppParentNode } from '@/app-structure.js';
import DefiBorrow from '@/components/DefiBorrow/DefiBorrow.vue';
import { focusElem } from '@/utils/aria.js';

export default {
    name: 'DefiMintRepay',

    components: { DefiBorrow, FBackButton },

    computed: {
        /**
         * @return {{token: DefiToken}|{}}
         */
        params() {
            const { $route } = this;

            return $route && $route.params ? $route.params : {};
        },

        backButtonRoute() {
            const parentNode = getAppParentNode('defi-mint-repay');

            return parentNode ? parentNode.id : '';
        },
    },

    mounted() {
        focusElem(this.$el);
    },
};
</script>
