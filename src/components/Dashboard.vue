<script setup>
import { ref, computed, onMounted, onUnmounted, defineAsyncComponent } from 'vue';
import { saveMisubs } from '../lib/api.js';
import { extractNodeName } from '../lib/utils.js';
import { useToastStore } from '../stores/toast.js';
import { useUIStore } from '../stores/ui.js';
import { useSubscriptions } from '../composables/useSubscriptions.js';
import { useManualNodes } from '../composables/useManualNodes.js';
import { useProfiles } from '../composables/useProfiles.js';

// --- Component Imports ---
import RightPanel from './RightPanel.vue';
import ProfilePanel from './ProfilePanel.vue';
import SubscriptionPanel from './SubscriptionPanel.vue';
import ManualNodePanel from './ManualNodePanel.vue';
import Modal from './Modal.vue';

const SettingsModal = defineAsyncComponent(() => import('./SettingsModal.vue'));
const BulkImportModal = defineAsyncComponent(() => import('./BulkImportModal.vue'));
const ProfileModal = defineAsyncComponent(() => import('./ProfileModal.vue'));
const SubscriptionImportModal = defineAsyncComponent(() => import('./SubscriptionImportModal.vue'));

// --- 基礎 Props 和狀態 ---
const props = defineProps({ data: Object });
const { showToast } = useToastStore();
const uiStore = useUIStore();
const isLoading = ref(true);
const dirty = ref(false);
const saveState = ref('idle');

// --- 將狀態和邏輯委託給 Composables ---
const markDirty = () => { dirty.value = true; saveState.value = 'idle'; };
const initialSubs = ref([]);
const initialNodes = ref([]);

const {
  subscriptions, subsCurrentPage, subsTotalPages, paginatedSubscriptions, totalRemainingTraffic,
  changeSubsPage, addSubscription, updateSubscription, deleteSubscription, deleteAllSubscriptions,
  addSubscriptionsFromBulk, handleUpdateNodeCount,
} = useSubscriptions(initialSubs, markDirty);

const {
  manualNodes, manualNodesCurrentPage, manualNodesTotalPages, paginatedManualNodes, searchTerm,
  changeManualNodesPage, addNode, updateNode, deleteNode, deleteAllNodes,
  addNodesFromBulk, autoSortNodes, deduplicateNodes,
} = useManualNodes(initialNodes, markDirty);

// --- 訂閱組 (Profile) 相關狀態 ---
const config = ref({});
const initialProfiles = ref([]);
const {
  profiles, editingProfile, isNewProfile, showProfileModal, showDeleteProfilesModal,
  initializeProfiles, handleProfileToggle, handleAddProfile, handleEditProfile,
  handleSaveProfile, handleDeleteProfile, handleDeleteAllProfiles, copyProfileLink,
  cleanupSubscriptions, cleanupNodes, cleanupAllSubscriptions, cleanupAllNodes,
} = useProfiles(initialProfiles, markDirty, config);

// --- UI State ---
const isSortingSubs = ref(false);
const isSortingNodes = ref(false);
const manualNodeViewMode = ref('card');
const editingSubscription = ref(null);
const isNewSubscription = ref(false);
const showSubModal = ref(false);
const editingNode = ref(null);
const isNewNode = ref(false);
const showNodeModal = ref(false);
const showBulkImportModal = ref(false);
const showDeleteSubsModal = ref(false);
const showDeleteNodesModal = ref(false);
const showSubscriptionImportModal = ref(false);
// --- 初始化與生命週期 ---
const initializeState = () => {
  isLoading.value = true;
  if (props.data) {
    const subsData = props.data.misubs || [];
    initialSubs.value = subsData.filter(item => item.url && /^https?:\/\//.test(item.url));
    initialNodes.value = subsData.filter(item => !item.url || !/^https?:\/\//.test(item.url));
    initialProfiles.value = props.data.profiles || [];
    config.value = props.data.config || {};
    initializeProfiles();
  }
  isLoading.value = false;
  dirty.value = false;
};

const handleBeforeUnload = (event) => {
  if (dirty.value) {
    event.preventDefault();
    event.returnValue = '您有未保存的更改，確定要离开嗎？';
  }
};

onMounted(() => {
  initializeState();
  window.addEventListener('beforeunload', handleBeforeUnload);
  const savedViewMode = localStorage.getItem('manualNodeViewMode');
  if (savedViewMode) {
    manualNodeViewMode.value = savedViewMode;
  }
});

onUnmounted(() => {
  window.removeEventListener('beforeunload', handleBeforeUnload);
});

const setViewMode = (mode) => {
  manualNodeViewMode.value = mode;
  localStorage.setItem('manualNodeViewMode', mode);
};

// --- 其他 JS 逻辑 (省略) ---
const handleDiscard = () => {
  initializeState();
  showToast('已放弃所有未保存的更改');
};
const handleSave = async () => {
  saveState.value = 'saving';
  const combinedMisubs = [
      ...subscriptions.value.map(sub => ({ ...sub, isUpdating: undefined })),
      ...manualNodes.value.map(node => ({ ...node, isUpdating: undefined }))
  ];

  try {
    // 数据验证
    if (!Array.isArray(combinedMisubs) || !Array.isArray(profiles.value)) {
      throw new Error('数据格式错误，请刷新页面后重试');
    }

    const result = await saveMisubs(combinedMisubs, profiles.value);

    if (result.success) {
        saveState.value = 'success';
        showToast('保存成功！', 'success');
        setTimeout(() => { dirty.value = false; saveState.value = 'idle'; }, 1500);
    } else {
        // 显示服务器返回的具体错误信息
        const errorMessage = result.message || result.error || '保存失败，请稍后重试';
        throw new Error(errorMessage);
    }
  } catch (error) {
    console.error('保存数据时发生错误:', error);

    // 根据错误类型提供不同的用户提示
    let userMessage = error.message;
    if (error.message.includes('网络')) {
      userMessage = '网络连接异常，请检查网络后重试';
    } else if (error.message.includes('格式')) {
      userMessage = '数据格式异常，请刷新页面后重试';
    } else if (error.message.includes('存储')) {
      userMessage = '存储服务暂时不可用，请稍后重试';
    }

    showToast(userMessage, 'error');
    saveState.value = 'idle';
  }
};
const handleDeleteSubscriptionWithCleanup = (subId) => {
  deleteSubscription(subId);
  cleanupSubscriptions(subId);
};
const handleDeleteNodeWithCleanup = (nodeId) => {
  deleteNode(nodeId);
  cleanupNodes(nodeId);
};
const handleDeleteAllSubscriptionsWithCleanup = () => {
  deleteAllSubscriptions();
  cleanupAllSubscriptions();
  showDeleteSubsModal.value = false;
};
const handleDeleteAllNodesWithCleanup = () => {
  deleteAllNodes();
  cleanupAllNodes();
  showDeleteNodesModal.value = false;
};
const handleAutoSortNodes = () => {
  autoSortNodes();
  showToast('已按地区排序！正在为您自动保存...', 'success');
  handleSave();
};

const handleDeduplicateNodes = () => {
    deduplicateNodes();
    showToast('已完成去重，请手动保存', 'success');
};
const handleBulkImport = (importText) => {
  if (!importText) return;
  const lines = importText.split('\n').map(line => line.trim()).filter(Boolean);
  const newSubs = [], newNodes = [];
  for (const line of lines) {
      const newItem = { id: crypto.randomUUID(), name: extractNodeName(line) || '未命名', url: line, enabled: true, status: 'unchecked' };
      if (/^https?:\/\//.test(line)) {
          newSubs.push(newItem);
      } else if (/^(ss|ssr|vmess|vless|trojan|hysteria2?|hy|hy2|tuic|anytls|socks5):\/\//.test(line)) {
          newNodes.push(newItem);
      }
  }
  if (newSubs.length > 0) addSubscriptionsFromBulk(newSubs);
  if (newNodes.length > 0) addNodesFromBulk(newNodes);
  showToast(`成功导入 ${newSubs.length} 条订阅和 ${newNodes.length} 个手动节点，请点击保存`, 'success');
};
const handleAddSubscription = () => {
  isNewSubscription.value = true;
  editingSubscription.value = { name: '', url: '', enabled: true, exclude: '' }; // 新增 exclude
  showSubModal.value = true;
};
const handleEditSubscription = (subId) => {
  const sub = subscriptions.value.find(s => s.id === subId);
  if (sub) {
    isNewSubscription.value = false;
    editingSubscription.value = { ...sub };
    showSubModal.value = true;
  }
};
const handleSaveSubscription = () => {
  if (!editingSubscription.value || !editingSubscription.value.url) { showToast('订阅链接不能为空', 'error'); return; }
  if (!/^https?:\/\//.test(editingSubscription.value.url)) { showToast('请输入有效的 http:// 或 https:// 订阅链接', 'error'); return; }
  
  if (isNewSubscription.value) {
    addSubscription({ ...editingSubscription.value, id: crypto.randomUUID() });
  } else {
    updateSubscription(editingSubscription.value);
  }
  showSubModal.value = false;
};
const handleAddNode = () => {
  isNewNode.value = true;
  editingNode.value = { id: crypto.randomUUID(), name: '', url: '', enabled: true };
  showNodeModal.value = true;
};
const handleEditNode = (nodeId) => {
  const node = manualNodes.value.find(n => n.id === nodeId);
  if (node) {
    isNewNode.value = false;
    editingNode.value = { ...node };
    showNodeModal.value = true;
  }
};
const handleNodeUrlInput = (event) => {
  if (!editingNode.value) return;
  const newUrl = event.target.value;
  if (newUrl && !editingNode.value.name) {
    editingNode.value.name = extractNodeName(newUrl);
  }
};
const handleSaveNode = () => {
    if (!editingNode.value || !editingNode.value.url) { showToast('节点链接不能为空', 'error'); return; }
    if (isNewNode.value) {
        addNode(editingNode.value);
    } else {
        updateNode(editingNode.value);
    }
    showNodeModal.value = false;
};

const formatBytes = (bytes, decimals = 2) => {
  if (!+bytes || bytes < 0) return '0 B';
  const k = 1024;
  const dm = decimals < 0 ? 0 : decimals;
  const sizes = ['B', 'KB', 'MB', 'GB', 'TB', 'PB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return `${parseFloat((bytes / Math.pow(k, i)).toFixed(dm))} ${sizes[i]}`;
};
const formattedTotalRemainingTraffic = computed(() => formatBytes(totalRemainingTraffic.value));

</script>

<template>
  <div v-if="isLoading" class="text-center py-16 text-gray-500">
    正在加载...
  </div>
  <div v-else class="w-full max-w-screen-xl mx-auto p-4 sm:p-6 lg:p-8">
    <!-- Header -->
    <div class="flex justify-between items-center mb-8">
      <div class="flex items-center gap-4">
        <h1 class="text-2xl font-bold text-gray-800 dark:text-white">仪表盘</h1>
        <span 
          v-if="formattedTotalRemainingTraffic !== '0 B'"
          class="px-3 py-1 text-sm font-semibold text-green-700 bg-green-100 dark:text-green-300 dark:bg-green-500/20 rounded-full"
        >
          剩余总流量: {{ formattedTotalRemainingTraffic }}
        </span>
      </div>
      <div class="flex items-center gap-2">
        <button @click="showBulkImportModal = true" class="text-sm font-semibold px-4 py-2 rounded-lg text-indigo-600 dark:text-indigo-400 border-2 border-indigo-500/50 hover:bg-indigo-500/10 transition-colors">批量导入</button>
      </div>
    </div>

    <!-- Dirty State Banner -->
    <Transition name="slide-fade">
      <div v-if="dirty" class="p-3 mb-6 rounded-lg bg-indigo-600/10 dark:bg-indigo-500/20 ring-1 ring-inset ring-indigo-600/20 flex items-center justify-between">
        <p class="text-sm font-medium text-indigo-800 dark:text-indigo-200">您有未保存的更改</p>
        <div class="flex items-center gap-3">
          <button @click="handleDiscard" class="text-sm font-medium text-gray-700 dark:text-gray-300 hover:text-gray-900 dark:hover:text-white transition-colors">放弃更改</button>
          <button @click="handleSave" :disabled="saveState !== 'idle'" class="px-5 py-2 text-sm text-white font-semibold rounded-lg shadow-sm flex items-center justify-center transition-all duration-300 w-28" :class="{'bg-green-600 hover:bg-green-700': saveState === 'idle', 'bg-gray-500 cursor-not-allowed': saveState === 'saving', 'bg-teal-500 cursor-not-allowed': saveState === 'success' }">
            <div v-if="saveState === 'saving'" class="flex items-center"><svg class="animate-spin h-5 w-5 mr-2" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg><span>保存中...</span></div>
            <div v-else-if="saveState === 'success'" class="flex items-center"><svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="3"><path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" /></svg><span>已保存</span></div>
            <span v-else>保存更改</span>
          </button>
        </div>
      </div>
    </Transition>

    <!-- Main Grid -->
    <div class="grid grid-cols-1 lg:grid-cols-3 gap-8 items-start">
      <div class="lg:col-span-2 space-y-12">
        <!-- Subscription Panel -->
        <SubscriptionPanel
          :subscriptions="subscriptions"
          :paginated-subscriptions="paginatedSubscriptions"
          :current-page="subsCurrentPage"
          :total-pages="subsTotalPages"
          :is-sorting="isSortingSubs"
          @add="handleAddSubscription"
          @delete="handleDeleteSubscriptionWithCleanup"
          @change-page="changeSubsPage"
          @update-node-count="handleUpdateNodeCount"
          @edit="handleEditSubscription"
          @toggle-sort="isSortingSubs = !isSortingSubs"
          @mark-dirty="markDirty"
          @delete-all="showDeleteSubsModal = true"
        />

        <!-- Manual Node Panel -->
        <ManualNodePanel
          :manual-nodes="manualNodes"
          :paginated-manual-nodes="paginatedManualNodes"
          :current-page="manualNodesCurrentPage"
          :total-pages="manualNodesTotalPages"
          :is-sorting="isSortingNodes"
          :search-term="searchTerm"
          :view-mode="manualNodeViewMode"
          @add="handleAddNode"
          @delete="handleDeleteNodeWithCleanup"
          @edit="handleEditNode"
          @change-page="changeManualNodesPage"
          @update:search-term="newVal => searchTerm.value = newVal"
          @update:view-mode="setViewMode"
          @toggle-sort="isSortingNodes = !isSortingNodes"
          @mark-dirty="markDirty"
          @auto-sort="handleAutoSortNodes"
          @deduplicate="handleDeduplicateNodes"
          @import="showSubscriptionImportModal = true"
          @delete-all="showDeleteNodesModal = true"
        />
      </div>
      
      <!-- Right Column -->
      <div class="lg:col-span-1 space-y-8">
        <RightPanel :config="config" :profiles="profiles" />
        <ProfilePanel 
          :profiles="profiles"
          @add="handleAddProfile"
          @edit="handleEditProfile"
          @delete="handleDeleteProfile"
          @deleteAll="showDeleteProfilesModal = true"
          @toggle="handleProfileToggle"
          @copyLink="copyProfileLink"
        />
      </div>
    </div>
  </div>

  <BulkImportModal v-model:show="showBulkImportModal" @import="handleBulkImport" />
  <Modal v-model:show="showDeleteSubsModal" @confirm="handleDeleteAllSubscriptionsWithCleanup"><template #title><h3 class="text-lg font-bold text-red-500">确认清空订阅</h3></template><template #body><p class="text-sm text-gray-400">您确定要删除所有**订阅**吗？此操作将标记为待保存，不会影响手动节点。</p></template></Modal>
  <Modal v-model:show="showDeleteNodesModal" @confirm="handleDeleteAllNodesWithCleanup"><template #title><h3 class="text-lg font-bold text-red-500">确认清空节点</h3></template><template #body><p class="text-sm text-gray-400">您确定要删除所有**手动节点**吗？此操作将标记为待保存，不会影响订阅。</p></template></Modal>
  <Modal v-model:show="showDeleteProfilesModal" @confirm="handleDeleteAllProfiles"><template #title><h3 class="text-lg font-bold text-red-500">确认清空订阅组</h3></template><template #body><p class="text-sm text-gray-400">您确定要删除所有**订阅组**吗？此操作不可逆。</p></template></Modal>
  
  <ProfileModal v-if="showProfileModal" v-model:show="showProfileModal" :profile="editingProfile" :is-new="isNewProfile" :all-subscriptions="subscriptions" :all-manual-nodes="manualNodes" @save="handleSaveProfile" size="2xl" />
  
  <Modal v-if="editingNode" v-model:show="showNodeModal" @confirm="handleSaveNode">
    <template #title><h3 class="text-lg font-bold text-gray-800 dark:text-white">{{ isNewNode ? '新增手动节点' : '编辑手动节点' }}</h3></template>
    <template #body>
      <div class="space-y-4">
        <div><label for="node-name" class="block text-sm font-medium text-gray-700 dark:text-gray-300">节点名称</label><input type="text" id="node-name" v-model="editingNode.name" placeholder="（可选）不填将自动获取" class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm dark:text-white"></div>
        <div><label for="node-url" class="block text-sm font-medium text-gray-700 dark:text-gray-300">节点链接</label><textarea id="node-url" v-model="editingNode.url" @input="handleNodeUrlInput" rows="4" class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm font-mono dark:text-white"></textarea></div>
      </div>
    </template>
  </Modal>

  <Modal v-if="editingSubscription" v-model:show="showSubModal" @confirm="handleSaveSubscription">
    <template #title><h3 class="text-lg font-bold text-gray-800 dark:text-white">{{ isNewSubscription ? '新增订阅' : '编辑订阅' }}</h3></template>
    <template #body>
      <div class="space-y-4">
        <div><label for="sub-edit-name" class="block text-sm font-medium text-gray-700 dark:text-gray-300">订阅名称</label><input type="text" id="sub-edit-name" v-model="editingSubscription.name" placeholder="（可选）不填将自动获取" class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm dark:text-white"></div>
        <div><label for="sub-edit-url" class="block text-sm font-medium text-gray-700 dark:text-gray-300">订阅链接</label><input type="text" id="sub-edit-url" v-model="editingSubscription.url" placeholder="https://..." class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm font-mono dark:text-white"></div>
        <div>
          <label for="sub-edit-exclude" class="block text-sm font-medium text-gray-700 dark:text-gray-300">包含/排除节点</label>
          <textarea 
            id="sub-edit-exclude" 
            v-model="editingSubscription.exclude"
            placeholder="[排除模式 (默认)]&#10;proto:vless,trojan&#10;(过期|官网)&#10;---&#10;[包含模式 (只保留匹配项)]&#10;keep:(香港|HK)&#10;keep:proto:ss"
            rows="5" 
            class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm font-mono dark:text-white">
          </textarea>
          <p class="text-xs text-gray-400 mt-1">每行一条规则。使用 `keep:` 切换为白名单模式。</p>
        </div>
      </div>
    </template>
  </Modal>
  
  <SettingsModal v-model:show="uiStore.isSettingsModalVisible" />
  <SubscriptionImportModal :show="showSubscriptionImportModal" @update:show="showSubscriptionImportModal = $event" :add-nodes-from-bulk="addNodesFromBulk" />
</template>

<style scoped>
.slide-fade-enter-active, .slide-fade-leave-active { transition: all 0.3s ease-out; }
.slide-fade-enter-from,
.slide-fade-leave-to {
  transform: translateY(-20px);
  opacity: 0;
}
.cursor-move {
  cursor: move;
}

.slide-fade-sm-enter-active,
.slide-fade-sm-leave-active {
  transition: all 0.2s ease-out;
}
.slide-fade-sm-enter-from,
.slide-fade-sm-leave-to {
  transform: translateY(-10px);
  opacity: 0;
}
</style>
