<script setup>
    import { computed } from 'vue';

    const props = defineProps({
        subscriptionUrl: {
            type: String,
            default: '',
        },
        isLoading: {
            type: Boolean,
            default: false,
        },
    });

    const emit = defineEmits(['update:subscriptionUrl', 'submit']);

    const urlModel = computed({
        get: () => props.subscriptionUrl,
        set: (val) => emit('update:subscriptionUrl', val),
    });
</script>

<template>
    <div>
        <label
            for="subscription-content"
            class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-2"
        >
            订阅链接 / Base64 内容
        </label>
        <textarea
            id="subscription-content"
            v-model="urlModel"
            rows="4"
            placeholder="支持两种，二选一：&#10;1) 订阅链接：https://example.com/subscription-link&#10;2) 直接粘贴 Base64，或节点链接（vmess://、vless:// 等）"
            class="w-full px-3 py-2 border border-gray-300 dark:border-gray-600 misub-radius-md bg-white dark:bg-gray-800 text-gray-900 dark:text-gray-100 font-mono text-xs leading-relaxed resize-y focus:outline-hidden focus:ring-2 focus:ring-blue-500 focus:border-blue-500 disabled:opacity-50"
            :disabled="isLoading"
        ></textarea>
    </div>

    <div class="text-xs text-gray-500 dark:text-gray-400">
        <p>
            提示：填订阅链接会联网抓取；直接粘 Base64 或节点链接则本地解析、不发请求（不会
            403）。导入的节点将添加到手动节点列表。
        </p>
    </div>
</template>
