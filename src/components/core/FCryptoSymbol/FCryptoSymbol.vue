<template>
    <div class="f-crypto-symbol">
        <span class="img" :style="{ width: imgWidth, height: imgHeight }">
            <icon v-if="svgIcon" :data="svgIcon" original />
            <img
                v-else-if="token.logoUrl || token.logoURL"
                :src="token.logoUrl || token.logoURL"
                class="not-fluid"
                :alt="$defi.getTokenSymbol(token)"
            />
        </span>
        <span v-if="!noSymbol">{{ $defi.getTokenSymbol(token) }}</span>
    </div>
</template>

<script>
import ftmIcon from '../../../assets/svg/tokens/NEXT.svg';
import wnextIcon from '../../../assets/svg/tokens/wNEXT.svg';
import fusdIcon from '../../../assets/svg/tokens/fUSD.svg';

/**
 * Render crypto logo and name by given token.
 * Requires vue defi plugin ($defi).
 */
export default {
    name: 'FCryptoSymbol',

    props: {
        /** @type {DefiToken} */
        token: {
            type: Object,
            default() {
                return null;
            },
            required: true,
        },
        /** Hide symbol (text). */
        noSymbol: {
            type: Boolean,
            default: false,
        },
        imgWidth: {
            type: String,
            default: '40px',
        },
        imgHeight: {
            type: String,
            default: '40px',
        },
    },

    data() {
        return {
            logoUrl: this.token ? this.token.logoUrl || this.token.logoURL : '',
        };
    },

    computed: {
        svgIcon() {
            const { token } = this;

            if (token) {
                switch (token.symbol) {
                    case 'NEXT':
                        return ftmIcon;
                    case 'WNEXT':
                        return wnextIcon;
                    case 'FUSD':
                        return fusdIcon;
                }
            }

            return null;
        },
    },
};
</script>

<style lang="scss">
@import 'style';
</style>
