<template>
    <div class="funiswap-add-liquidity-confirmation">
        <tx-confirmation
            v-if="hasCorrectParams"
            :tx="tx"
            card-off
            set-tmp-pwd
            :tmp-pwd-code="tmpPwdCode"
            :tmp-pwd-count="params.step === 1 ? 2 : 0"
            :send-button-label="sendButtonLabel"
            :password-label="passwordLabel"
            :show-cancel-button="!isView"
            :window-mode="!isView"
            class="min-h-100"
            :on-send-transaction-success="onSendTransactionSuccess"
            @cancel-button-click="$emit('cancel-button-click', $event)"
            @change-component="onChangeComponent"
        >
            <h1 v-if="isView" class="with-back-btn" data-focus>
                Confirm Supply
                <template v-if="params.steps">({{ params.step }}/{{ params.steps }})</template>
                <f-back-button
                    v-if="!params.steps || params.step === 1"
                    :route-name="backButtonRoute"
                    :params="{ fromToken: params.fromToken, toToken: params.toToken }"
                />
            </h1>

            <div class="confirmation-info" tabindex="0" data-focus>
                <template v-if="params.step === 1">
                    You’re allowing
                    <span class="price">
                        <f-token-value
                            :value="params.fromValue"
                            :token="params.fromToken"
                            :use-placeholder="false"
                            no-currency
                        />
                        {{ fromTokenSymbol }}
                    </span>
                </template>
                <template v-else-if="params.step === 2">
                    You’re allowing
                    <span class="price">
                        <f-token-value
                            :value="params.toValue"
                            :token="params.toToken"
                            :use-placeholder="false"
                            no-currency
                        />
                        {{ toTokenSymbol }}
                    </span>
                </template>
                <template v-else>
                    You're adding
                    <span class="price">
                        <f-token-value
                            :value="params.fromValue"
                            :token="params.fromToken"
                            :use-placeholder="false"
                            no-currency
                        />
                        {{ fromTokenSymbol }}
                    </span>
                    ,
                    <span class="price">
                        <f-token-value
                            :value="params.toValue"
                            :token="params.toToken"
                            :use-placeholder="false"
                            no-currency
                        />
                        {{ toTokenSymbol }}
                    </span>
                </template>
            </div>

            <template #window-content>
                <ledger-confirmation-content :to="tx.to" :amount="0" :max-fee="tx._fee" />
            </template>
        </tx-confirmation>
        <template v-else>
            <f-message type="info" role="alert" class="big"> Trade tokens first, please. </f-message>
        </template>
    </div>
</template>

<script>
import TxConfirmation from '@/components/TxConfirmation/TxConfirmation.vue';
import LedgerConfirmationContent from '@/components/LedgerConfirmationContent/LedgerConfirmationContent.vue';
import { mapGetters } from 'vuex';
import { toNEXT } from '@/utils/transactions.js';
import FBackButton from '@/components/core/FBackButton/FBackButton.vue';
import { getAppParentNode } from '@/app-structure.js';
import FMessage from '@/components/core/FMessage/FMessage.vue';
import uniswapUtils from 'next-ledgerjs/src/uniswap-utils.js';
// import erc20Utils from 'next-ledgerjs/src/erc20-utils.js';
import Web3 from 'web3';
import erc20Utils from 'next-ledgerjs/src/erc20-utils.js';
import appConfig from '../../../../app.config.js';
import { getUniqueId } from '@/utils';
import FTokenValue from '@/components/core/FTokenValue/FTokenValue.vue';

export default {
    name: 'FUniswapAddLiquidityConfirmation',

    components: { FTokenValue, FMessage, FBackButton, LedgerConfirmationContent, TxConfirmation },

    props: {
        /** Address of smart contract. */
        contractAddress: {
            type: String,
            default: '',
        },
        props: {
            type: Object,
            default() {
                return {};
            },
        },
        /** Identifies if component is view (has route). */
        isView: {
            type: Boolean,
            default: false,
        },
    },

    data() {
        return {
            compName: 'funiswap-pools',
            confirmationCompName: 'funiswap-add-liquidity',
            priceDecimals: 6,
            tx: {},
            tmpPwdCode: '',
        };
    },

    computed: {
        ...mapGetters(['currentAccount']),

        /**
         * @return {{fromValue: number, toValue: number, fromToken: DefiToken, toToken: DefiToken}}
         */
        params() {
            const { $route } = this;

            if (this.isView) {
                return $route && $route.params ? $route.params : {};
            }

            return this.props;
        },

        hasCorrectParams() {
            return !!this.params.fromValue;
        },

        increasedDebt() {
            return this.params.currDebt - this.params.debt;
        },

        decreasedDebt() {
            return this.params.debt - this.params.currDebt;
        },

        sendButtonLabel() {
            let label = '';

            // if (this.params.step === 1) {
            //     label = 'Continue to the next step';
            // } else {
            label = 'Submit';
            // label = 'Trade now';
            // }

            return label;
        },

        passwordLabel() {
            let label = '';

            if (this.params.step === 1) {
                label = 'Please enter your wallet password to allow your tokens';
            } else {
                label = 'Please enter your wallet password to trade your tokens';
            }

            return label;
        },

        fromTokenSymbol() {
            return this.$defi.getTokenSymbol(this.params.fromToken);
        },

        toTokenSymbol() {
            return this.$defi.getTokenSymbol(this.params.toToken);
        },

        backButtonRoute() {
            const parentNode = getAppParentNode(`${this.confirmationCompName}-confirmation`);

            return parentNode ? parentNode.id : '';
        },
    },

    created() {
        if (!this.hasCorrectParams) {
            // redirect to <this.compName>
            setTimeout(() => {
                this.$router.replace({ name: this.compName });
            }, 3000);
        } else {
            this.setTx();
        }
    },

    methods: {
        async setTx() {
            let { contractAddress } = this;

            const { params } = this;
            const { $defi } = this;
            const { fromToken } = params;
            const { toToken } = params;
            let txToSign;
            const web3 = new Web3();
            const { slippageTolerance } = params;

            if (!fromToken || !toToken) {
                return;
            }

            this.tmpPwdCode = params.tmpPwdCode || getUniqueId();

            if (!contractAddress) {
                contractAddress = $defi.contracts.uniswapRouter;
            }

            if (params.step === 1) {
                txToSign = erc20Utils.erc20IncreaseAllowanceTx(
                    fromToken.address,
                    contractAddress,
                    Web3.utils.toHex($defi.shiftDecPointRight(params.fromValue.toString(), fromToken.decimals))
                );
            } else if (params.step === 2) {
                txToSign = erc20Utils.erc20IncreaseAllowanceTx(
                    toToken.address,
                    contractAddress,
                    Web3.utils.toHex($defi.shiftDecPointRight(params.toValue.toString(), toToken.decimals))
                );
            } else {
                /*
                const amounts = await this.$defi.fetchUniswapAmountsOut(
                    Web3.utils.toHex(this.$defi.shiftDecPointRight(params.fromValue.toString(), fromToken.decimals)),
                    [fromToken.address, toToken.address]
                );
*/

                let amounts = await this.$defi.fetchUniswapQuoteLiquidity(
                    [fromToken.address, toToken.address],
                    [
                        Web3.utils.toHex($defi.shiftDecPointRight(params.fromValue.toString(), fromToken.decimals)),
                        Web3.utils.toHex($defi.shiftDecPointRight(params.toValue.toString(), toToken.decimals)),
                    ]
                );

                // apply slippage tolerance
                amounts = [$defi.fromTokenValue(amounts[0], fromToken), $defi.fromTokenValue(amounts[1], toToken)];
                amounts = amounts.map((_item) => _item * (1 - slippageTolerance));
                amounts = [
                    Web3.utils.toHex($defi.shiftDecPointRight(amounts[0].toString(), fromToken.decimals)),
                    Web3.utils.toHex($defi.shiftDecPointRight(amounts[1].toString(), toToken.decimals)),
                ];

                txToSign = uniswapUtils.uniswapAddLiquidity(
                    web3,
                    contractAddress,
                    fromToken.address,
                    toToken.address,
                    Web3.utils.toHex($defi.shiftDecPointRight(params.fromValue.toString(), fromToken.decimals)),
                    Web3.utils.toHex($defi.shiftDecPointRight(params.toValue.toString(), toToken.decimals)),
                    amounts[0],
                    amounts[1],
                    /*
                    // slippage 0.5%
                    Web3.utils.toHex(
                        $defi.shiftDecPointRight(
                            (params.fromValue * (1 - slippageTolerance)).toString(),
                            fromToken.decimals
                        )
                    ),
                    // slippage 0.5%
                    Web3.utils.toHex(
                        $defi.shiftDecPointRight(
                            (params.toValue * (1 - slippageTolerance)).toString(),
                            toToken.decimals
                        )
                    ),
*/
                    this.currentAccount.address,
                    (Math.floor(new Date().getTime() / 1000) + 20 * 60).toString()
                );
            }

            if (txToSign) {
                this.tx = await this.$fWallet.getDefiTransactionToSign(txToSign, this.currentAccount.address);
            }
        },

        onSendTransactionSuccess(_data) {
            const params = {
                tx: _data.data.sendTransaction.hash,
                title: 'Success',
                continueTo: this.compName,
            };
            let transactionSuccessComp = `${this.confirmationCompName}-transaction-success-message`;

            if (this.params.step === 1) {
                params.continueTo = `${this.confirmationCompName}-confirmation2`;
                params.continueToParams = { ...this.params, step: 2, tmpPwdCode: this.tmpPwdCode };
                params.autoContinueToAfter = appConfig.settings.autoContinueToAfter;
                params.continueButtonLabel = 'Next Step';
                params.title = `${this.params.step}/${this.params.steps}  ${params.title}`;
            } else if (this.params.step === 2) {
                transactionSuccessComp = `${this.confirmationCompName}-transaction-success-message2`;
                params.continueTo = `${this.confirmationCompName}-confirmation3`;
                params.continueToParams = { ...this.params, step: 3, tmpPwdCode: this.tmpPwdCode };
                params.autoContinueToAfter = appConfig.settings.autoContinueToAfter;
                params.continueButtonLabel = 'Next Step';
                params.title = `${this.params.step}/${this.params.steps}  ${params.title}`;
            } else if (this.params.step === 3) {
                transactionSuccessComp = `${this.confirmationCompName}-transaction-success-message3`;
                params.continueToParams = {
                    fromToken: { ...this.params.fromToken },
                    toToken: { ...this.params.toToken },
                    tmpPwdCode: this.tmpPwdCode,
                };
            }

            if (this.isView) {
                this.$router.replace({
                    name: transactionSuccessComp,
                    params,
                });
            } else {
                if (this.params.step === 1 || this.params.step === 2) {
                    params.continueToParams = {
                        props: { ...params.continueToParams },
                    };
                    params.title = `Success`;
                } else if (this.params.step === 3 || !this.params.step) {
                    params.continueTo = 'hide-window';
                    params.continueButtonLabel = 'Close';
                }

                this.$emit('change-component', {
                    to: transactionSuccessComp,
                    data: { ...params, cardOff: true, windowMode: true },
                });
            }
        },

        /**
         * Re-target `'change-component'` event.
         *
         * @param {object} _data
         */
        onChangeComponent(_data) {
            if (!this.isView) {
                return;
            }

            let transactionRejectComp = `${this.confirmationCompName}-transaction-reject-message`;

            if (_data.to === 'transaction-reject-message') {
                if (this.params.step === 3) {
                    transactionRejectComp = `${this.confirmationCompName}-transaction-reject-message3`;
                }

                this.$router.replace({
                    name: transactionRejectComp,
                    params: {
                        continueTo: this.compName,
                        continueToParams: {
                            fromToken: { ...this.params.fromToken },
                            toToken: { ...this.params.toToken },
                            tmpPwdCode: this.tmpPwdCode,
                        },
                    },
                });
            }
        },

        toNEXT,
    },
};
</script>
