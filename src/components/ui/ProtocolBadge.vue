<script setup>
    import { computed } from 'vue';

    const props = defineProps({
        url: {
            type: String,
            default: '',
        },
    });

    const getProtocol = (url) => {
        try {
            if (!url) return 'unknown';
            const lowerUrl = url.toLowerCase();
            if (lowerUrl.startsWith('anytls://')) return 'anytls';
            if (lowerUrl.startsWith('hysteria2://') || lowerUrl.startsWith('hy2://'))
                return 'hysteria2';
            if (lowerUrl.startsWith('hysteria://') || lowerUrl.startsWith('hy://'))
                return 'hysteria';
            if (lowerUrl.startsWith('ssr://')) return 'ssr';
            if (lowerUrl.startsWith('tuic://')) return 'tuic';
            if (lowerUrl.startsWith('ss://')) return 'ss';
            if (lowerUrl.startsWith('vmess://')) return 'vmess';
            if (lowerUrl.startsWith('vless://')) return 'vless';
            if (lowerUrl.startsWith('trojan://')) return 'trojan';
            if (lowerUrl.startsWith('socks5://') || lowerUrl.startsWith('socks://'))
                return 'socks5';
            if (lowerUrl.startsWith('snell://')) return 'snell';
            if (
                lowerUrl.startsWith('naive+https://') ||
                lowerUrl.startsWith('naive+http://') ||
                lowerUrl.startsWith('naive+quic://')
            )
                return 'naive';
            if (lowerUrl.startsWith('http')) return 'http';
        } catch {
            return 'unknown';
        }
        return 'unknown';
    };

    const protocol = computed(() => getProtocol(props.url));

    const protocolStyle = computed(() => {
        const p = protocol.value;
        switch (p) {
            case 'anytls':
                return {
                    text: 'AnyTLS',
                    style: 'bg-slate-500/20 text-slate-500 dark:text-slate-400',
                };
            case 'vless':
                return { text: 'VLESS', style: 'bg-blue-500/20 text-blue-500 dark:text-blue-400' };
            case 'hysteria2':
                return {
                    text: 'HY2',
                    style: 'bg-purple-500/20 text-purple-500 dark:text-purple-400',
                };
            case 'hysteria':
                return {
                    text: 'Hysteria',
                    style: 'bg-fuchsia-500/20 text-fuchsia-500 dark:text-fuchsia-400',
                };
            case 'tuic':
                return { text: 'TUIC', style: 'bg-cyan-500/20 text-cyan-500 dark:text-cyan-400' };
            case 'trojan':
                return { text: 'TROJAN', style: 'bg-red-500/20 text-red-500 dark:text-red-400' };
            case 'ssr':
                return { text: 'SSR', style: 'bg-rose-500/20 text-rose-500 dark:text-rose-400' };
            case 'ss':
                return {
                    text: 'SS',
                    style: 'bg-orange-500/20 text-orange-500 dark:text-orange-400',
                };
            case 'vmess':
                return { text: 'VMESS', style: 'bg-teal-500/20 text-teal-500 dark:text-teal-400' };
            case 'socks5':
                return { text: 'SOCKS5', style: 'bg-lime-500/20 text-lime-500 dark:text-lime-400' };
            case 'http':
                return {
                    text: 'HTTP',
                    style: 'bg-green-500/20 text-green-500 dark:text-green-400',
                };
            case 'snell':
                return {
                    text: 'SNELL',
                    style: 'bg-indigo-500/20 text-indigo-500 dark:text-indigo-400',
                };
            case 'naive':
                return { text: 'NAIVE', style: 'bg-pink-500/20 text-pink-500 dark:text-pink-400' };
            default:
                return { text: 'LINK', style: 'bg-gray-500/20 text-gray-500 dark:text-gray-400' };
        }
    });
</script>

<template>
    <span
        class="shrink-0 rounded-full px-2 py-0.5 text-[11px] font-semibold"
        :class="protocolStyle.style"
    >
        {{ protocolStyle.text }}
    </span>
</template>
