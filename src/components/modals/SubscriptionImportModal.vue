<script setup>
    import { ref, watch } from 'vue';
    import { useToastStore } from '../../stores/toast.js';
    import Modal from '../forms/Modal.vue';
    import { handleError } from '../../utils/errorHandler.js';
    import { generateNodeId } from '../../utils/id.js';
    import { api, APIError } from '../../lib/http.js';
    import FormatDetector from './SubscriptionImport/FormatDetector.vue';
    import ImportForm from './SubscriptionImport/ImportForm.vue';
    import ParseResult from './SubscriptionImport/ParseResult.vue';
    import GroupSelector from '../ui/GroupSelector.vue'; // Added

    const isDev = import.meta.env.DEV;

    const props = defineProps({
        show: Boolean,
        addNodesFromBulk: Function,
        groups: {
            // Added
            type: Array,
            default: () => [],
        },
    });

    const emit = defineEmits(['update:show']);

    const subscriptionUrl = ref('');
    const isLoading = ref(false);
    const errorMessage = ref('');
    const successMessage = ref('');
    const parseStatus = ref('');
    const groupName = ref(''); // Added

    const toastStore = useToastStore();

    watch(
        () => props.show,
        (newVal) => {
            if (!newVal) {
                // 重置所有状态
                subscriptionUrl.value = '';
                groupName.value = ''; // Added
                errorMessage.value = '';
                successMessage.value = '';
                parseStatus.value = '';
                isLoading.value = false;
            }
        }
    );

    /**
     * 验证URL格式
     */
    const isValidUrl = (url) => {
        try {
            const u = new URL(url);
            return u.protocol === 'http:' || u.protocol === 'https:';
        } catch {
            return false;
        }
    };

    /**
     * 导入订阅
     */
    const importSubscription = async () => {
        const targetGroupName = groupName.value; // Capture immediately to avoid reset by watcher when modal closes
        const input = subscriptionUrl.value.trim();

        if (!input) {
            errorMessage.value = '请输入订阅链接或 Base64 内容。';
            return;
        }

        isLoading.value = true;
        errorMessage.value = '';
        successMessage.value = '';

        try {
            let content;

            if (isValidUrl(input)) {
                // —— 订阅链接：联网抓取（保持原逻辑，后端含 403 UA 轮换重试）——
                parseStatus.value = '正在获取订阅内容...';
                const controller = new AbortController();
                const timeoutId = setTimeout(() => controller.abort(), 20000); // 20秒超时

                let responseData;
                try {
                    responseData = await api.post(
                        '/api/fetch_external_url',
                        {
                            url: input,
                            timeout: 15000,
                        },
                        {
                            signal: controller.signal,
                        }
                    );
                } catch (error) {
                    if (error instanceof APIError) {
                        const errorMsg =
                            error.data?.error ||
                            error.data?.message ||
                            error.message ||
                            `HTTP ${error.status}`;

                        // 根据错误类型提供友好的错误信息
                        if (error.status === 408 || errorMsg.includes('timeout')) {
                            throw new Error('请求超时，请检查网络连接或稍后重试');
                        } else if (error.status === 413 || errorMsg.includes('too large')) {
                            throw new Error('订阅内容过大，请使用较小的订阅链接');
                        } else if (errorMsg.includes('DNS')) {
                            throw new Error('域名解析失败，请检查订阅链接是否正确');
                        } else if (error.status >= 500) {
                            throw new Error('服务器错误，请稍后重试');
                        }
                        throw new Error(errorMsg);
                    }
                    throw error;
                } finally {
                    clearTimeout(timeoutId);
                }

                if (!responseData.success) {
                    throw new Error(responseData.error || '获取订阅内容失败');
                }
                content = responseData.content;
            } else {
                // —— Base64 / 原始节点链接：直接交后端解析，不发外网请求，不会 403 ——
                // 后端 parseNodeList 会自动识别并解码整段 Base64 订阅内容
                parseStatus.value = '正在解析订阅内容...';
                content = input;
            }

            parseStatus.value = `正在解析订阅内容...`;

            // [重构] 调用后端解析API
            const parseResult = await api.post('/api/parse_subscription', {
                content,
            });

            if (!parseResult.success) {
                throw new Error(parseResult.error || '解析订阅失败');
            }

            const backendNodes = parseResult.data.nodes || [];

            if (backendNodes.length > 0) {
                // 转换为前端格式
                const nodes = backendNodes.map((node) => ({
                    id: generateNodeId(),
                    name: node.name || 'Unknown',
                    url: node.url,
                    enabled: true,
                    protocol: node.protocol || 'unknown',
                    source: 'import',
                }));

                // 去重处理
                const uniqueNodes = nodes.filter(
                    (node, index, self) => index === self.findIndex((n) => n.url === node.url)
                );

                const duplicateCount = nodes.length - uniqueNodes.length;

                props.addNodesFromBulk(uniqueNodes, targetGroupName); // Updated

                const successMsg =
                    `成功添加 ${uniqueNodes.length} 个节点` +
                    (targetGroupName ? ` 到分组 "${targetGroupName}"` : '') +
                    (duplicateCount > 0 ? `（去重 ${duplicateCount} 个重复节点）` : '');
                successMessage.value = successMsg;

                toastStore.showToast(successMsg, 'success');
                if (isDev) {
                    console.debug(
                        `[Import] Success: Backend API, ${uniqueNodes.length} unique nodes, ${duplicateCount} duplicates`
                    );
                }

                setTimeout(() => {
                    emit('update:show', false);
                }, 2000);
            } else {
                parseStatus.value = '';
                throw new Error(
                    '未能解析出任何有效节点。请检查是否为有效的 Base64 订阅或 vmess/vless 等节点链接。'
                );
            }
        } catch (error) {
            console.error('导入订阅失败:', error);
            handleError(error, 'Subscription Import Error', {
                url: subscriptionUrl.value,
                parseStatus: parseStatus.value,
            });

            errorMessage.value = error.message || '导入失败';
            toastStore.showToast(`导入失败: ${error.message}`, 'error');
        } finally {
            isLoading.value = false;
        }
    };
</script>

<template>
    <Modal
        :show="show"
        @update:show="emit('update:show', $event)"
        @confirm="importSubscription"
        confirm-text="导入"
        :confirm-disabled="isLoading || !subscriptionUrl.trim()"
    >
        <template #title>导入订阅</template>
        <template #body>
            <div class="space-y-4">
                <FormatDetector />

                <!-- Group Selector Added -->
                <div class="relative">
                    <label
                        class="text-xs font-semibold text-gray-500 dark:text-gray-400 mb-1.5 ml-1 block"
                    >
                        导入分组
                    </label>
                    <GroupSelector
                        v-model="groupName"
                        :groups="groups"
                        placeholder="选择或输入分组（可选）"
                        class="w-full"
                    />
                </div>

                <ImportForm
                    :subscription-url="subscriptionUrl"
                    :is-loading="isLoading"
                    @update:subscription-url="subscriptionUrl = $event"
                    @submit="importSubscription"
                />
                <ParseResult
                    :is-loading="isLoading"
                    :parse-status="parseStatus"
                    :error-message="errorMessage"
                    :success-message="successMessage"
                />
            </div>
        </template>
    </Modal>
</template>
