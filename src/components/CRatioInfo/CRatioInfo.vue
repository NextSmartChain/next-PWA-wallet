<template>
    <f-info window-closeable window-class="light" window-style="max-width: 500px;" class="debt-limit-f-info">
        C-Ratio needs to be at least 300%.<br />
        If your C-Ratio is above 500%, you’ll earn rewards in wNEXT.<br />
        If your C-Ratio falls below 300%, your wNEXT will be locked (not liquidated) until the C-Ratio goes back to at
        least 300%. That can happen either via NEXT price fluctuations or if you add more wNEXT as collateral.
    </f-info>
</template>

<script>
import FInfo from '@/components/core/FInfo/FInfo.vue';

export default {
    name: 'CRatioInfo',

    components: { FInfo },
};
</script>
