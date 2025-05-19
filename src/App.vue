<template>
  <el-container class="app-container">
    <el-aside :width="isSidebarCollapsed ? '64px' : '280px'" class="sidebar" :class="{ 'is-collapsed': isSidebarCollapsed }">
      <div class="logo">
        <img src="/favicon.jpg" alt="MultiAI Logo" style="width: 40px; height: 40px; vertical-align: middle;" :style="{ 'margin-right': isSidebarCollapsed ? '0px' : '8px' }">
        <span v-if="!isSidebarCollapsed">MultiAI</span>
      </div>
      <div class="toggle-sidebar-button" @click="toggleSidebar">
        <el-icon v-if="isSidebarCollapsed"><ArrowRightBold /></el-icon>
        <el-icon v-else><ArrowLeftBold /></el-icon>
      </div>
      <el-menu :default-active="activeTab" class="menu" @select="handleMenuSelect" :collapse="isSidebarCollapsed">
        <el-menu-item index="provider">
          <el-icon><Setting /></el-icon>
          <span>服务商配置</span>
        </el-menu-item>
        <el-menu-item index="system">
          <el-icon><Document /></el-icon>
          <span>系统提示词</span>
        </el-menu-item>
        <el-menu-item index="peers">
          <el-icon><User /></el-icon>
          <span>AI角色配置</span>
        </el-menu-item>
        <el-menu-item index="chat">
          <el-icon><ChatDotRound /></el-icon>
          <span>聊天</span>
        </el-menu-item>
      </el-menu>
    </el-aside>

    <el-container class="main-content-wrapper">
      <el-header class="header">
        <h2>{{ getTabTitle() }}</h2>
      </el-header>

      <el-main class="main-content-area">
        <div v-if="activeTab === 'provider'">
          <el-card class="config-card">
            <template #header>
              <div class="card-header">
                <span>服务商列表</span>
                <el-button type="primary" @click="showAddProviderDialog">添加服务商</el-button>
              </div>
            </template>
            <el-table :data="providers" style="width: 100%">
              <el-table-column prop="name" label="名称" width="150" />
              <el-table-column prop="url" label="URL" />
              <el-table-column label="模型" width="120">
                <template #default="scope">
                  <el-popover placement="top" :width="300" trigger="hover">
                    <template #default>
                      <div v-if="scope.row.models && scope.row.models.length" class="models-popover">
                        <el-tag v-for="(model, index) in scope.row.models" :key="index" size="small" style="margin: 2px;">{{ model }}</el-tag>
                      </div>
                      <div v-else>未配置模型</div>
                    </template>
                    <template #reference>
                      <el-button link type="primary">{{ scope.row.models ? scope.row.models.length : 0 }} 个模型</el-button>
                    </template>
                  </el-popover>
                </template>
              </el-table-column>
              <el-table-column label="操作" width="220">
                <template #default="scope">
                  <el-button size="small" @click="showEditProviderDialog(scope.row)">编辑</el-button>
                  <el-button size="small" @click="refreshProviderModels(scope.row)">刷新模型</el-button>
                  <el-button size="small" type="danger" @click="removeProvider(scope.$index)">删除</el-button>
                </template>
              </el-table-column>
            </el-table>
          </el-card>
        </div>

        <div v-if="activeTab === 'system'">
          <el-card class="config-card">
            <template #header>
              <div class="card-header">
                <span>公共系统提示词</span>
                <el-button type="primary" @click="addSystemPrompt">添加提示词</el-button>
              </div>
            </template>
            <el-table :data="publicSystemList" style="width: 100%">
              <el-table-column label="占位符" width="120">
                <template #default="scope">
                  <el-tag>${{ 'p' + (scope.$index + 1) }}</el-tag>
                </template>
              </el-table-column>
              <el-table-column label="提示词内容">
                <template #default="scope">
                  <el-input type="textarea" v-model="publicSystemList[scope.$index]" :autosize="{ minRows: 2, maxRows: 6 }" resize="none" />
                </template>
              </el-table-column>
              <el-table-column label="操作" width="100">
                <template #default="scope">
                  <el-button size="small" type="danger" @click="removeSystemPrompt(scope.$index)">删除</el-button>
                </template>
              </el-table-column>
            </el-table>
          </el-card>
        </div>

        <div v-if="activeTab === 'peers'">
          <el-card class="config-card">
            <template #header>
              <div class="card-header">
                <span>AI角色列表</span>
                <el-button type="primary" @click="showAddPeerDialog">添加角色</el-button>
              </div>
            </template>
            <el-table :data="peers" style="width: 100%">
              <el-table-column prop="id" label="ID" width="150" />
              <el-table-column label="名称 (聊天显示)" width="150">
                <template #default="scope">
                  {{ scope.row.name }}
                  <el-tag v-if="scope.row.isObserver" type="info" size="small" style="margin-left: 5px;">旁观者</el-tag>
                </template>
              </el-table-column>
              <el-table-column prop="provider" label="服务商" width="150" />
              <el-table-column prop="model" label="模型" width="180" />
              <el-table-column label="系统提示词" width="200">
                <template #default="scope">
                  <el-input type="textarea" v-model="scope.row.systemPrompt" :autosize="{ minRows: 1, maxRows: 4 }" resize="none" readonly />
                </template>
              </el-table-column>
               <el-table-column label="流式" width="70" align="center">
                <template #default="scope">
                  <el-tag :type="scope.row.stream ? 'success' : 'info'">{{ scope.row.stream ? '是' : '否' }}</el-tag>
                </template>
              </el-table-column>
              <el-table-column label="启用" width="70" align="center">
                <template #default="scope">
                  <el-switch v-model="scope.row.enable" />
                </template>
              </el-table-column>
               <el-table-column label="聆听" width="70" align="center">
                <template #default="scope">
                   <el-tag :type="scope.row.listening ? 'primary' : 'info'">{{ scope.row.listening ? '开启' : '关闭' }}</el-tag>
                </template>
              </el-table-column>
              <el-table-column label="操作" width="150" fixed="right">
                <template #default="scope">
                  <el-button size="small" @click="showEditPeerDialog(scope.row)">编辑</el-button>
                  <el-button size="small" type="danger" @click="removePeer(scope.row.id)">删除</el-button>
                </template>
              </el-table-column>
            </el-table>
            <div class="speaking-order" style="margin-top: 20px;">
              <h3>发言顺序</h3>
              <el-select v-model="speakingOrder" multiple placeholder="选择发言顺序 (仅启用角色)" style="width: 100%;">
                <el-option v-for="peer in enabledPeersForSpeakingOrder" :key="peer.name" :label="`${peer.name} (${peer.id})${peer.isObserver ? ' (旁观者)' : ''}`" :value="peer.name" />
              </el-select>
            </div>
          </el-card>
        </div>

         <div v-if="activeTab === 'chat'" class="chat-interface">
          <div class="chat-messages-container" ref="chatMessagesContainer_ref">
            <div v-for="(message, index) in chatMessages" :key="message.id || index"
                 class="message-wrapper"
                 :class="{ 'user-message-wrapper': message.role === '用户',
                           'ai-message-wrapper': isConfiguredPeerMessage(message.role) && !isObserverMessage(message.role),
                           'observer-message-wrapper': isObserverMessage(message.role),
                           'custom-role-message-wrapper': !isConfiguredPeerMessage(message.role) && message.role !== '用户' }">
              <div class="message-meta">
                  <span class="message-role-display">{{ message.role }}{{ isObserverMessage(message.role) ? ' (旁观)' : '' }}</span>
              </div>
              <div class="message-bubble" :class="{'observer-bubble': isObserverMessage(message.role), 'custom-role-bubble': !isConfiguredPeerMessage(message.role) && message.role !== '用户' }">
                <div class="message-content" v-html="formatMessageContent(message.content)"></div>
              </div>
            </div>
            <div v-if="isGenerating && liveStreamDisplayContent !== null"
                 class="message-wrapper"
                 :class="getGeneratingPeerIsObserver() ? 'observer-message-wrapper' : 'ai-message-wrapper'">
                <div class="message-meta">
                    <span class="message-role-display">{{ peerCurrentlyGenerating }} {{ getGeneratingPeerIsObserver() ? '(旁观者)' : '' }} 正在输入...</span>
                </div>
                <div class="message-bubble" :class="{'observer-bubble': getGeneratingPeerIsObserver() }">
                    <div class="message-content" v-if="liveStreamDisplayContent" v-html="formatMessageContent(liveStreamDisplayContent)"></div>
                     <div class="message-content" v-else><i>等待数据流...</i></div>
                </div>
            </div>
          </div>

          <div class="chat-input-area">
            <div class="chat-actions">
              <template v-for="peer in enabledPeersForChatActions" :key="peer.id">
                <el-button
                  @click="letPeerSpeak(peer.name)"
                  :disabled="isGenerating" size="small">
                  {{ peer.name }}{{ peer.isObserver ? ' (旁观)' : '' }}
                </el-button>
                <el-tooltip v-if="!peer.isObserver" :content="peer.listening ? '正在聆听其他AI对话' : '仅聆听用户和自身对话'" placement="top">
                  <el-button
                    :icon="peer.listening ? View : Hide"
                    @click="togglePeerListening(peer.id)"
                    size="small"
                    :type="peer.listening ? 'primary' : 'info'"
                    circle
                    plain
                    style="margin-left: -5px; margin-right: 5px;"
                  />
                </el-tooltip>
                 <el-tooltip v-if="peer.isObserver" :content="peer.listening ? '正在分析其他AI对话' : '仅分析用户对话'" placement="top">
                  <el-button
                    :icon="peer.listening ? View : Hide"
                    @click="togglePeerListening(peer.id)"
                    size="small"
                    :type="peer.listening ? 'primary' : 'info'"
                    circle
                    plain
                    style="margin-left: -5px; margin-right: 5px;"
                  />
                </el-tooltip>
              </template>
              <el-button type="success" @click="autoRound"
                :disabled="isGenerating || speakingOrder.length === 0" size="small">
                自动轮流
              </el-button>
              <el-button @click="deleteLastMessage" size="small" :disabled="isGenerating || chatMessages.length === 0" type="warning">删除最后一条</el-button>
              <el-button @click="clearChat" size="small" :disabled="isGenerating">清空对话</el-button>
            </div>
            <div class="user-input-row">
               <el-select
                v-model="userMessageRole"
                placeholder="角色"
                filterable
                allow-create
                default-first-option
                class="role-input"
              >
                <el-option label="用户" value="用户" />
                <el-option v-for="peer in enabledPeersForRoleDropdown" :key="peer.name" :label="peer.name + (peer.isObserver ? ' (旁观者)' : '')" :value="peer.name" />
              </el-select>
              <el-input
                v-model="userMessageContent"
                placeholder="输入你的消息..."
                @keyup.enter="sendUserMessage"
                :disabled="isGenerating"
                class="message-input"
                resize="none"
                type="textarea"
                :autosize="{ minRows: 1, maxRows: 5 }"
              />
              <el-button type="primary" @click="sendUserMessage" :disabled="isGenerating || !userMessageContent.trim()" class="send-button">
                <el-icon><Promotion /></el-icon>
              </el-button>
            </div>
          </div>
        </div>
      </el-main>
    </el-container>
  </el-container>

  <el-dialog v-model="providerDialogVisible" :title="editingProvider ? '编辑服务商' : '添加服务商'" width="600px" @close="editingProvider = null">
    <el-form :model="providerForm" label-width="120px" ref="providerFormRef">
      <el-form-item label="名称" prop="name" :rules="{ required: true, message: '名称不能为空', trigger: 'blur' }">
        <el-input v-model="providerForm.name" :disabled="!!editingProvider && providerForm.name === editingProvider.name" />
      </el-form-item>
      <el-form-item label="URL" prop="url" :rules="{ required: true, message: 'URL不能为空', trigger: 'blur' }">
        <el-input v-model="providerForm.url" placeholder="例如：https://api.openai.com/v1" />
      </el-form-item>
      <el-form-item label="API 密钥">
        <el-input v-model="providerForm.key" type="password" show-password />
      </el-form-item>
      <el-form-item label="模型">
        <div style="display: flex; align-items: center; gap: 10px; margin-bottom: 10px;">
            <el-button
                @click="fetchModelsForDialog"
                size="small"
                :disabled="!providerForm.url || !providerForm.key || isFetchingDialogModels"
                :loading="isFetchingDialogModels"
            >
            获取模型
            </el-button>
        </div>
        <el-tag v-for="(model, index) in providerForm.models" :key="model + index" closable @close="removeFormModel(index)" style="margin-right: 5px; margin-bottom: 5px;">
          {{ model }}
        </el-tag>
        <el-input v-if="formModelInputVisible" ref="formModelInput_ref" v-model="formModelValue" size="small" style="width: 200px; margin-top: 5px;" @keyup.enter="addFormModel" @blur="addFormModel" />
        <el-button v-else size="small" @click="showFormModelInput" :icon="Plus" style="margin-top: 5px;">手动添加模型</el-button>
      </el-form-item>
    </el-form>
    <template #footer>
      <el-button @click="providerDialogVisible = false">取消</el-button>
      <el-button type="primary" @click="saveProvider">确认</el-button>
    </template>
  </el-dialog>

  <el-dialog v-model="peerDialogVisible" :title="editingPeer ? '编辑 AI 角色' : '添加 AI 角色'" width="600px" @close="editingPeer = null">
    <el-form :model="peerForm" label-width="120px" ref="peerFormRef">
      <el-form-item label="ID (唯一标识)" prop="id" :rules="{ required: true, message: 'ID不能为空', trigger: 'blur' }">
        <el-input v-model="peerForm.id" :disabled="!!editingPeer" />
      </el-form-item>
      <el-form-item label="名称 (聊天显示)" prop="name" :rules="{ required: true, message: '名称不能为空', trigger: 'blur' }">
        <el-input v-model="peerForm.name" />
      </el-form-item>
       <el-form-item label="设为旁观者">
        <el-switch v-model="peerForm.isObserver" />
         <el-tooltip content="旁观者角色会生成内容，但其发言不会被其他AI角色学习。旁观者在形成自己观点时，也不会参考自己之前的发言。" placement="top-start">
            <el-icon style="margin-left: 8px; color: #909399;"><InfoFilled /></el-icon>
         </el-tooltip>
      </el-form-item>
      <el-form-item label="服务商" prop="provider" :rules="{ required: true, message: '服务商不能为空', trigger: 'change' }">
        <el-select v-model="peerForm.provider" placeholder="选择服务商" style="width: 100%" @change="updatePeerFormModels">
          <el-option v-for="p in providers" :key="p.name" :label="p.name" :value="p.name" />
        </el-select>
      </el-form-item>
      <el-form-item label="模型" prop="model" :rules="{ required: true, message: '模型不能为空', trigger: 'change' }">
        <el-select v-model="peerForm.model" placeholder="选择模型" style="width: 100%">
          <el-option v-for="m in availablePeerModels" :key="m" :label="m" :value="m" />
        </el-select>
      </el-form-item>
      <el-form-item label="系统提示词">
        <el-input type="textarea" v-model="peerForm.systemPrompt" :autosize="{ minRows: 3, maxRows: 8 }" placeholder="使用 $p1, $p2 等引用公共提示词" />
      </el-form-item>
      <el-form-item label="流式输出">
        <el-switch v-model="peerForm.stream" />
      </el-form-item>
      <el-form-item label="启用角色">
        <el-switch v-model="peerForm.enable" />
      </el-form-item>
       <el-form-item label="默认聆听对话">
        <el-switch v-model="peerForm.listening" />
         <el-tooltip content="开启后，此角色会参考其他AI的发言。旁观者开启此项，会分析其他AI发言；关闭则仅分析用户发言。" placement="top-start">
            <el-icon style="margin-left: 8px; color: #909399;"><InfoFilled /></el-icon>
         </el-tooltip>
      </el-form-item>
    </el-form>
    <template #footer>
      <el-button @click="peerDialogVisible = false">取消</el-button>
      <el-button type="primary" @click="savePeer">确认</el-button>
    </template>
  </el-dialog>
</template>

<script setup>
import { ref, reactive, computed, nextTick, onMounted, watch } from 'vue'
import { Setting, Document, User, ChatDotRound, Promotion, Plus, View, Hide, InfoFilled, ArrowLeftBold, ArrowRightBold } from '@element-plus/icons-vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import axios from 'axios'
import { marked } from 'marked';

const LS_KEYS = {
  PROVIDERS: 'multiAI_providers_v6_zh',
  SYSTEM_PROMPTS: 'multiAI_systemPrompts_v6_zh',
  PEERS: 'multiAI_peers_v6_zh',
  SPEAKING_ORDER: 'multiAI_speakingOrder_v6_zh',
  ACTIVE_TAB: 'multiAI_activeTab_v6_zh',
  CHAT_MESSAGES: 'multiAI_chatMessages_v6_zh',
  SIDEBAR_COLLAPSED: 'multiAI_sidebarCollapsed_v6_zh',
};

const activeTab = ref(localStorage.getItem(LS_KEYS.ACTIVE_TAB) || 'provider');
const isSidebarCollapsed = ref(localStorage.getItem(LS_KEYS.SIDEBAR_COLLAPSED) === 'true' || false);
const providers = ref([]);
const publicSystemList = ref([]);
const peers = ref([]);
const speakingOrder = ref([]);
const chatMessages = ref([]);
const isGenerating = ref(false);
const peerCurrentlyGenerating = ref('');
const liveStreamDisplayContent = ref(null);
const userMessageContent = ref('');
const userMessageRole = ref('用户');
const chatMessagesContainer_ref = ref(null);
const isFetchingDialogModels = ref(false);

const defaultPeer = () => ({
  id: '',
  name: '',
  provider: '',
  model: '',
  systemPrompt: '',
  stream: true,
  enable: true,
  listening: true,
  listeningStartIndex: null,
  isObserver: false,
});

const loadFromLocalStorage = () => {
  providers.value = JSON.parse(localStorage.getItem(LS_KEYS.PROVIDERS) || '[]');
  publicSystemList.value = JSON.parse(localStorage.getItem(LS_KEYS.SYSTEM_PROMPTS) || '[]');
  peers.value = JSON.parse(localStorage.getItem(LS_KEYS.PEERS) || '[]').map(p => ({ ...defaultPeer(), ...p, listeningStartIndex: null }));
  speakingOrder.value = JSON.parse(localStorage.getItem(LS_KEYS.SPEAKING_ORDER) || '[]');
  chatMessages.value = JSON.parse(localStorage.getItem(LS_KEYS.CHAT_MESSAGES) || '[]').map(m => ({...m, id: m.id || Date.now() + Math.random() }));
};

const saveToLocalStorage = (key, data) => {
  if (key === LS_KEYS.PEERS) {
    const peersToSave = data.map(p => {
      const { listeningStartIndex, ...peerToSave } = p;
      return peerToSave;
    });
    localStorage.setItem(key, JSON.stringify(peersToSave));
  } else {
    localStorage.setItem(key, JSON.stringify(data));
  }
};

const toggleSidebar = () => {
  isSidebarCollapsed.value = !isSidebarCollapsed.value;
  localStorage.setItem(LS_KEYS.SIDEBAR_COLLAPSED, isSidebarCollapsed.value);
};

watch(activeTab, (newTab) => localStorage.setItem(LS_KEYS.ACTIVE_TAB, newTab));
watch(providers, (newData) => saveToLocalStorage(LS_KEYS.PROVIDERS, newData), { deep: true });
watch(publicSystemList, (newData) => saveToLocalStorage(LS_KEYS.SYSTEM_PROMPTS, newData), { deep: true });
watch(peers, (newData) => saveToLocalStorage(LS_KEYS.PEERS, newData), { deep: true });
watch(speakingOrder, (newData) => saveToLocalStorage(LS_KEYS.SPEAKING_ORDER, newData), { deep: true });
watch(chatMessages, (newData) => saveToLocalStorage(LS_KEYS.CHAT_MESSAGES, newData), { deep: true });

const providerDialogVisible = ref(false);
const editingProvider = ref(null);
const providerFormRef = ref(null);
const providerForm = reactive({ name: '', url: '', key: '', models: [] });
const formModelInputVisible = ref(false);
const formModelValue = ref('');
const formModelInput_ref = ref(null);

const peerDialogVisible = ref(false);
const editingPeer = ref(null);
const peerFormRef = ref(null);
const peerForm = reactive(defaultPeer());
const availablePeerModels = ref([]);

const enabledPeersForRoleDropdown = computed(() => peers.value.filter(p => p.enable));
const enabledPeersForChatActions = computed(() => peers.value.filter(p => p.enable));
const enabledPeersForSpeakingOrder = computed(() => peers.value.filter(p => p.enable));

const getTabTitle = () => {
  const titles = { provider: '服务商配置', system: '系统提示词配置', peers: 'AI角色配置', chat: '聊天界面' }
  return titles[activeTab.value] || '多AI聊天'
};

const handleMenuSelect = (index) => {
  activeTab.value = index;
};

const showAddProviderDialog = () => {
  editingProvider.value = null;
  Object.assign(providerForm, { name: '', url: '', key: '', models: [] });
  providerDialogVisible.value = true;
  nextTick(() => providerFormRef.value?.clearValidate());
};

const showEditProviderDialog = (provider) => {
  editingProvider.value = JSON.parse(JSON.stringify(provider));
  Object.assign(providerForm, JSON.parse(JSON.stringify(provider)));
  providerDialogVisible.value = true;
  nextTick(() => providerFormRef.value?.clearValidate());
};

const saveProvider = async () => {
  if (!providerFormRef.value) return;
  await providerFormRef.value.validate((valid) => {
    if (valid) {
      const providerData = JSON.parse(JSON.stringify(providerForm));
      if (editingProvider.value) {
        const index = providers.value.findIndex(p => p.name === editingProvider.value.name);
        if (index !== -1) {
          if (providerData.name !== editingProvider.value.name && providers.value.some((p, i) => i !== index && p.name === providerData.name)) {
            ElMessage.error(`服务商名称 '${providerData.name}' 已存在.`);
            return;
          }
          providers.value.splice(index, 1, providerData);
          ElMessage.success(`服务商 '${providerData.name}' 已更新.`);
        }
      } else {
        if (providers.value.some(p => p.name === providerData.name)) {
          ElMessage.error(`服务商名称 '${providerData.name}' 已存在.`);
          return;
        }
        providers.value.push(providerData);
        ElMessage.success(`服务商 '${providerData.name}' 已添加.`);
      }
      providerDialogVisible.value = false;
      editingProvider.value = null;
    }
  });
};

const removeProvider = (index) => {
  ElMessageBox.confirm(`确定删除服务商 '${providers.value[index].name}' 吗? 这也会删除使用此服务商的AI角色。`, '确认删除', { type: 'warning', confirmButtonText: '确定', cancelButtonText: '取消' })
    .then(() => {
      const providerName = providers.value[index].name;
      providers.value.splice(index, 1);

      const peersToRemoveNames = peers.value.filter(peer => peer.provider === providerName).map(p => p.name);
      peers.value = peers.value.filter(peer => peer.provider !== providerName);
      speakingOrder.value = speakingOrder.value.filter(nameInOrder => !peersToRemoveNames.includes(nameInOrder));

      ElMessage.success('服务商及关联AI角色已删除.');
    }).catch(() => {});
};

const fetchModelsForDialog = async () => {
    if (!providerForm.url || !providerForm.key) {
        ElMessage.warning('需要填写URL和API密钥才能获取模型.');
        return;
    }
    isFetchingDialogModels.value = true;
    try {
        const headers = { 'Authorization': `Bearer ${providerForm.key}` };
        let url = providerForm.url;
        if (!url.endsWith('/')) url += '/';
        const response = await axios.get(`${url}models`, { headers });
        if (response.data && response.data.data) {
            const apiModels = response.data.data.map(model => model.id);
            providerForm.models = [...new Set([...(providerForm.models || []), ...apiModels])];
            ElMessage.success(`模型已获取并合并到 ${providerForm.name || '当前服务商'}.`);
        } else {
            ElMessage.warning('无法从API响应中解析模型.');
        }
    } catch (error) {
        console.error("Error fetching models for dialog:", error);
        ElMessage.error(`获取模型失败: ${error.response?.data?.error?.message || error.message || '未知错误'}`);
    } finally {
        isFetchingDialogModels.value = false;
    }
};

const refreshProviderModels = async (provider) => {
  if (!provider.key || !provider.url) {
    ElMessage.warning('服务商URL或API密钥缺失.');
    return;
  }
  const originalGeneratingState = isGenerating.value;
  isGenerating.value = true;
  peerCurrentlyGenerating.value = `正在刷新 ${provider.name}`;
  liveStreamDisplayContent.value = "";
  try {
    const headers = { 'Authorization': `Bearer ${provider.key}` };
    let url = provider.url;
    if (!url.endsWith('/')) url += '/';
    const response = await axios.get(`${url}models`, { headers });
    if (response.data && response.data.data) {
      const apiModels = response.data.data.map(model => model.id);
      const providerInList = providers.value.find(p => p.name === provider.name);
      if (providerInList) {
        providerInList.models = [...new Set([...(providerInList.models || []), ...apiModels])];
      }
      ElMessage.success(`服务商 '${provider.name}' 的模型已刷新.`);
    } else {
      ElMessage.warning('无法从API响应中解析模型.');
    }
  } catch (error) {
    console.error("Error refreshing provider models:", error);
    ElMessage.error(`刷新模型失败: ${error.response?.data?.error?.message || error.message || '未知错误'}`);
  } finally {
    isGenerating.value = originalGeneratingState;
    peerCurrentlyGenerating.value = '';
    liveStreamDisplayContent.value = null;
  }
};

const showFormModelInput = () => {
  formModelInputVisible.value = true;
  nextTick(() => formModelInput_ref.value?.focus());
};

const addFormModel = () => {
  if (formModelValue.value && !providerForm.models.includes(formModelValue.value)) {
    providerForm.models.push(formModelValue.value);
  }
  formModelValue.value = '';
  formModelInputVisible.value = false;
};

const removeFormModel = (index) => {
  providerForm.models.splice(index, 1);
};

const addSystemPrompt = () => publicSystemList.value.push('');
const removeSystemPrompt = (index) => publicSystemList.value.splice(index, 1);

const showAddPeerDialog = () => {
  if (providers.value.length === 0) {
    ElMessage.warning('请先添加服务商.');
    return;
  }
  editingPeer.value = null;
  Object.assign(peerForm, defaultPeer());
  availablePeerModels.value = [];
  if (providers.value.length > 0 && !peerForm.provider) {
      peerForm.provider = providers.value[0].name;
  }
  updatePeerFormModels();
  peerDialogVisible.value = true;
  nextTick(() => peerFormRef.value?.clearValidate());
};

const showEditPeerDialog = (peer) => {
  editingPeer.value = peer;
  const { listeningStartIndex, ...formValues } = JSON.parse(JSON.stringify(peer));
  Object.assign(peerForm, defaultPeer(), formValues);
  updatePeerFormModels();
  peerDialogVisible.value = true;
  nextTick(() => peerFormRef.value?.clearValidate());
};

const updatePeerFormModels = () => {
  const selectedProvider = providers.value.find(p => p.name === peerForm.provider);
  availablePeerModels.value = selectedProvider ? (selectedProvider.models || []) : [];
  if (availablePeerModels.value.length > 0 && !availablePeerModels.value.includes(peerForm.model)) {
    peerForm.model = availablePeerModels.value[0];
  } else if (availablePeerModels.value.length === 0) {
    peerForm.model = '';
  }
};

const savePeer = async () => {
  if (!peerFormRef.value) return;
  await peerFormRef.value.validate(async (valid) => {
    if (valid) {
      const { listeningStartIndex, ...peerDataFromForm } = JSON.parse(JSON.stringify(peerForm));
      const peerData = { ...defaultPeer(), ...peerDataFromForm, listeningStartIndex: null };

      if (editingPeer.value) {
        const index = peers.value.findIndex(p => p.id === editingPeer.value.id);
        if (index !== -1) {
          if (peerData.name !== peers.value[index].name && peers.value.some((p, i) => i !== index && p.name === peerData.name)) {
            ElMessage.error(`AI角色名称 '${peerData.name}' 已存在.`);
            return;
          }
          const oldPeerName = peers.value[index].name;
          peers.value.splice(index, 1, { ...peers.value[index], ...peerData });

          if (oldPeerName !== peerData.name) {
             speakingOrder.value = speakingOrder.value.map(nameInOrder => nameInOrder === oldPeerName ? peerData.name : nameInOrder);
          }
          ElMessage.success(`AI角色 '${peerData.name}' (ID: ${peerData.id}) 已更新.`);
        }
      } else {
        if (peers.value.some(p => p.id === peerData.id)) {
          ElMessage.error(`AI角色ID '${peerData.id}' 已存在.`);
          return;
        }
        if (peers.value.some(p => p.name === peerData.name)) {
          ElMessage.error(`AI角色名称 '${peerData.name}' 已存在.`);
          return;
        }
        peers.value.push(peerData);
        ElMessage.success(`AI角色 '${peerData.name}' (ID: ${peerData.id}) 已添加.`);
      }
      speakingOrder.value = speakingOrder.value.filter(name => {
        const p = peers.value.find(pr => pr.name === name);
        return p && p.enable;
      });

      peerDialogVisible.value = false;
      editingPeer.value = null;
    }
  });
};

const removePeer = (peerId) => {
 const peerToRemove = peers.value.find(p => p.id === peerId);
 if (!peerToRemove) return;

 ElMessageBox.confirm(`确定删除 AI 角色 '${peerToRemove.name}' (ID: ${peerToRemove.id}) 吗?`, '确认删除', { type: 'warning', confirmButtonText: '确定', cancelButtonText: '取消' })
    .then(() => {
      const index = peers.value.findIndex(p => p.id === peerId);
      if (index > -1) {
        const removedPeerName = peers.value[index].name;
        peers.value.splice(index, 1);
        speakingOrder.value = speakingOrder.value.filter(name => name !== removedPeerName);
        ElMessage.success('AI 角色已删除.');
      }
    }).catch(() => {});
};

const togglePeerListening = (peerId) => {
  const peer = peers.value.find(p => p.id === peerId);
  if (peer) {
    const oldListeningState = peer.listening;
    peer.listening = !peer.listening;

    if (!oldListeningState && peer.listening) {
        peer.listeningStartIndex = chatMessages.value.length;
    } else if (oldListeningState && !peer.listening) {
        // No change to listeningStartIndex needed here,
        // as the 'isWithinGeneralListeningWindow' check will fail if peer.listening is false.
    }
    const listeningTarget = peer.isObserver ? "其他AI对话进行分析" : "其他AI对话";
    const notListeningTarget = peer.isObserver ? "用户对话进行分析" : "用户和自身对话";

    ElMessage.info(`${peer.name} ${peer.listening ? `开始${peer.isObserver ? '' : '聆听'}${listeningTarget} (从当前位置)` : `仅${peer.isObserver ? '' : '聆听'}${notListeningTarget}`}`);
  }
};

const scrollToBottom = () => {
  nextTick(() => {
    if (chatMessagesContainer_ref.value) {
      chatMessagesContainer_ref.value.scrollTop = chatMessagesContainer_ref.value.scrollHeight;
    }
  });
};

const formatMessageContent = (content) => {
    if (typeof content !== 'string') return '';
    try {
        const parsedTurns = JSON.parse(content);
        if (Array.isArray(parsedTurns) && parsedTurns.every(turn => typeof turn === 'object' && Object.keys(turn).length === 1)) {
            return parsedTurns.map(turn => {
                const speaker = Object.keys(turn)[0];
                const markdownText = turn[speaker];
                const sanitizedSpeaker = String(speaker || '').replace(/</g, "<").replace(/>/g, ">");
                const htmlContent = marked.parse(String(markdownText || ''));
                return `<div><strong>${sanitizedSpeaker}:</strong> ${htmlContent}</div>`;
            }).join('<hr class="turn-separator">');
        }
    } catch (e) {
    }
    return marked.parse(String(content || ''));
};

const isConfiguredPeerMessage = (role) => peers.value.some(p => p.name === role);
const isObserverMessage = (role) => peers.value.find(p => p.name === role)?.isObserver || false;

const getGeneratingPeerIsObserver = () => {
    if (!peerCurrentlyGenerating.value) return false;
    const peer = peers.value.find(p => p.name === peerCurrentlyGenerating.value);
    return peer ? peer.isObserver : false;
};

const replaceSystemPromptPlaceholders = (promptText) => {
  if (!promptText) return "";
  let result = promptText;
  publicSystemList.value.forEach((pVal, i) => {
    result = result.replace(new RegExp(`\\$p${i + 1}`, 'g'), pVal);
  });
  return result;
};

const letPeerSpeak = async (peerName) => {
    if (isGenerating.value) return;

    const currentPeer = peers.value.find(p => p.name === peerName);
    if (!currentPeer || !currentPeer.enable) {
        ElMessage.error(`AI角色 '${peerName}' 未找到或未启用.`);
        return;
    }

    const provider = providers.value.find(p => p.name === currentPeer?.provider);
    if (!provider || !provider.url || !provider.key) {
        ElMessage.error(`AI角色 '${peerName}' 或其服务商未正确配置 (缺少URL/密钥).`);
        return;
    }

    isGenerating.value = true;
    peerCurrentlyGenerating.value = peerName;
    liveStreamDisplayContent.value = '';
    scrollToBottom();

    let accumulatedContentForFinalPush = "";

    try {
        const finalSystemPrompt = replaceSystemPromptPlaceholders(currentPeer.systemPrompt);
        const messagesForAPI = [{ role: 'system', content: finalSystemPrompt }];
        let currentMultiTurnUserContent = [];

        for (let i = 0; i < chatMessages.value.length; i++) {
            const msg = chatMessages.value[i];
            const msgIndex = i;
            const isOwnMessage = msg.role === currentPeer.name;

            let includeAsUserTurn = false;
            let includeAsAssistantTurn = false;

            if (isOwnMessage) {
                if (!currentPeer.isObserver) {
                    includeAsAssistantTurn = true;
                }
            } else {
                let isWithinGeneralListeningWindow = false;
                if (currentPeer.listening) {
                    if (currentPeer.listeningStartIndex === null || msgIndex >= currentPeer.listeningStartIndex) {
                        isWithinGeneralListeningWindow = true;
                    }
                }

                if (isWithinGeneralListeningWindow) {
                    if (currentPeer.isObserver) {
                        if (currentPeer.listening) { // Observer "listening to other AIs & User"
                            includeAsUserTurn = true;
                        } else { // Observer "listening to User only"
                            if (msg.role === '用户') {
                                includeAsUserTurn = true;
                            }
                        }
                    } else { // Current peer is NOT an observer
                        if (!isObserverMessage(msg.role)) { // Non-observer does not hear observer AIs
                            includeAsUserTurn = true;
                        }
                    }
                }
            }

            if (includeAsAssistantTurn) {
                 if (currentMultiTurnUserContent.length > 0) {
                    messagesForAPI.push({ role: 'user', content: JSON.stringify(currentMultiTurnUserContent) });
                    currentMultiTurnUserContent = [];
                }
                messagesForAPI.push({ role: 'assistant', content: msg.content });
            } else if (includeAsUserTurn) {
                currentMultiTurnUserContent.push({ [msg.role]: msg.content });
            }
        }

        if (currentMultiTurnUserContent.length > 0) {
            messagesForAPI.push({ role: 'user', content: JSON.stringify(currentMultiTurnUserContent) });
        }

        const headers = { 'Authorization': `Bearer ${provider.key}`, 'Content-Type': 'application/json' };
        let url = provider.url;
        if (!url.endsWith('/')) url += '/';

        const payload = { model: currentPeer.model, messages: messagesForAPI, stream: currentPeer.stream };

         if (messagesForAPI.length <= 1 && messagesForAPI[0]?.role === 'system' && !messagesForAPI[0]?.content.trim() && (messagesForAPI.length === 0 || !messagesForAPI.some(m => m.role === 'user' || m.role === 'assistant'))) {
             ElMessage.warning(`AI角色 '${peerName}' 的输入为空 (系统提示词为空且没有对话历史需要参考).`);
             isGenerating.value = false;
             peerCurrentlyGenerating.value = '';
             liveStreamDisplayContent.value = null;
             return;
        }


        if (currentPeer.stream) {
            const response = await fetch(`${url}chat/completions`, { method: 'POST', headers, body: JSON.stringify(payload) });
            if (!response.ok || !response.body) {
                const errorBody = response.ok ? "响应体为空" : await response.text();
                throw new Error(`API 错误 (${response.status}): ${errorBody}`);
            }

            const reader = response.body.getReader();
            const decoder = new TextDecoder();
            let buffer = '';

            while (true) {
                const { done, value } = await reader.read();
                if (done) {
                     if (buffer.trim()) {
                         const lines = buffer.split('\n').filter(line => line.trim() !== '');
                         for (const line of lines) {
                            if (line.startsWith('data: ')) {
                                const jsonStr = line.substring(5).trim();
                                if (jsonStr !== '[DONE]') {
                                    try {
                                        const data = JSON.parse(jsonStr);
                                        const deltaContent = data.choices?.[0]?.delta?.content;
                                        if (deltaContent) {
                                            accumulatedContentForFinalPush += deltaContent;
                                            liveStreamDisplayContent.value = accumulatedContentForFinalPush;
                                            scrollToBottom();
                                        }
                                    } catch (e) { console.warn("Stream parsing error (final):", e, jsonStr); }
                                }
                            }
                         }
                     }
                    break;
                }

                buffer += decoder.decode(value, { stream: true });
                let lines = buffer.split('\n');
                buffer = lines.pop();

                let shouldBreakOuterLoop = false;
                for (const line of lines) {
                    if (line.startsWith('data: ')) {
                        const jsonStr = line.substring(5).trim();
                        if (jsonStr === '[DONE]') {
                            shouldBreakOuterLoop = true;
                            break;
                        }
                        try {
                            const data = JSON.parse(jsonStr);
                            const deltaContent = data.choices?.[0]?.delta?.content;
                            if (deltaContent) {
                                accumulatedContentForFinalPush += deltaContent;
                                liveStreamDisplayContent.value = accumulatedContentForFinalPush;
                                scrollToBottom();
                            }
                            if (data.choices?.[0]?.finish_reason) {
                                shouldBreakOuterLoop = true;
                            }
                        } catch (e) { console.warn("Stream parsing error:", e, jsonStr); }
                    }
                }
                if (shouldBreakOuterLoop) break;
            }
        } else {
            const response = await axios.post(`${url}chat/completions`, payload, { headers });
            accumulatedContentForFinalPush = response.data?.choices?.[0]?.message?.content;
             if ((accumulatedContentForFinalPush === null || accumulatedContentForFinalPush === undefined) && response.data?.error) {
                throw new Error(`API 错误 (非流式): ${response.data.error.message || JSON.stringify(response.data.error)}`);
            }
        }

        if (accumulatedContentForFinalPush || accumulatedContentForFinalPush === "") {
            chatMessages.value.push({ role: peerName, content: accumulatedContentForFinalPush, id: Date.now() + Math.random() });
        } else if (!currentPeer.stream && (accumulatedContentForFinalPush === null || accumulatedContentForFinalPush === undefined)) {
            ElMessage.warning(`未从 ${peerName} 收到内容.`);
        }

    } catch (error) {
        console.error(`AI Error (${peerName}):`, error);
        ElMessage.error(`AI 错误 (${peerName}): ${error.message || '未知错误'}`);
    } finally {
        isGenerating.value = false;
        peerCurrentlyGenerating.value = '';
        liveStreamDisplayContent.value = null;
        scrollToBottom();
    }
};

const sendUserMessage = () => {
  if (!userMessageContent.value.trim() || isGenerating.value) return;
  const roleForMessage = userMessageRole.value || '用户';
  if (!roleForMessage.trim()) {
      ElMessage.warning("请为您的消息指定一个角色.");
      return;
  }
  chatMessages.value.push({ role: roleForMessage, content: userMessageContent.value, id: Date.now() + Math.random() });
  userMessageContent.value = '';
  scrollToBottom();
};

const deleteLastMessage = () => {
    if (chatMessages.value.length > 0 && !isGenerating.value) {
        chatMessages.value.pop();
        scrollToBottom();
    }
};

const autoRound = async () => {
  if (isGenerating.value || speakingOrder.value.length === 0) return;
  for (const peerName of speakingOrder.value) {
    if (isGenerating.value) {
        ElMessage.info("自动轮流已中断.");
        break;
    }
    const peerToSpeak = peers.value.find(p => p.name === peerName && p.enable);
    if (peerToSpeak) {
        await letPeerSpeak(peerToSpeak.name);
    } else {
        ElMessage.warning(`自动轮流：跳过角色 '${peerName}' (未找到或未启用).`);
    }
  }
};

const clearChat = () => {
  ElMessageBox.confirm('确定清空所有聊天记录吗?', '确认清空', { type: 'warning', confirmButtonText: '确定', cancelButtonText: '取消' })
    .then(() => {
      chatMessages.value = [];
      peers.value.forEach(p => {
          if (p.listening && p.listeningStartIndex !== null) { // only reset if it was actively listening and had an index
              p.listeningStartIndex = 0; // reset to start of (now empty) chat
          }
          // if !p.listening, listeningStartIndex should remain null or its previous state.
          // or more simply:
          // p.listeningStartIndex = p.listening ? 0 : null;
      });
      ElMessage.success('聊天记录已清空.');
    }).catch(() => {});
};

onMounted(() => {
  loadFromLocalStorage();
  if (providers.value.length === 0) {
     providers.value.push({ name: '示例 OpenAI', url: 'https://api.openai.com/v1', key: '', models: ['gpt-3.5-turbo', 'gpt-4'] });
  }
  if (publicSystemList.value.length === 0) {
    publicSystemList.value.push('你是一个有用的AI助理。');
  }
  speakingOrder.value = speakingOrder.value.filter(name => {
    const p = peers.value.find(pr => pr.name === name);
    return p && p.enable;
  });
  scrollToBottom();
});

</script>

<style>
body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol"; margin: 0; background-color: #f4f6f8; color: #333; }
.app-container { height: 100vh; display: flex; overflow: hidden; }
.sidebar { background-color: #ffffff; border-right: 1px solid #e0e0e0; display: flex; flex-direction: column; box-shadow: 2px 0 5px rgba(0,0,0,0.05); transition: width 0.3s ease; flex-shrink: 0; }
.sidebar.is-collapsed { width: 64px; }

.logo {
  padding: 20px;
  font-size: 1.4em;
  font-weight: 600;
  text-align: center;
  color: #2c3e50;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  align-items: center;
  justify-content: center;
  height: 60px;
  box-sizing: border-box;
  overflow: hidden;
  white-space: nowrap;
}

.logo img {
    transition: margin-right 0.3s ease;
}

.logo span {
    transition: opacity 0.3s ease;
}

.sidebar.is-collapsed .logo span {
    opacity: 0;
    width: 0;
}

.toggle-sidebar-button {
    position: absolute;
    top: 15px;
    right: 10px;
    cursor: pointer;
    font-size: 1.2em;
    color: #666;
    z-index: 100;
}


.menu { border-right: none !important; flex-grow: 1;}
.menu .el-menu-item { font-size: 0.95em; }
.menu .el-menu-item.is-active { background-color: #e6f7ff !important; color: #1a73e8 !important; }
.menu .el-menu-item .el-icon { margin-right: 10px; }

.menu.el-menu--collapse .el-menu-item {
    padding: 0px var(--el-menu-base-level-padding) !important;
    justify-content: center;
}
.menu.el-menu--collapse .el-menu-item .el-icon {
    margin-right: 0;
}


.main-content-wrapper { flex-grow: 1; display: flex; flex-direction: column; background-color: #f4f6f8; }
.header { background-color: #ffffff; border-bottom: 1px solid #e0e0e0; display: flex; align-items: center; padding: 0 20px; height: 60px; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
.header h2 { margin: 0; font-size: 1.3em; font-weight: 500; color: #333; }
.main-content-area { padding: 20px; overflow-y: auto; flex-grow: 1; }
.config-card { background-color: #ffffff; border-radius: 8px; border: 1px solid #e0e0e0; margin-bottom: 20px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06); }
.config-card .el-card__header { background-color: #fafafa; border-bottom: 1px solid #e0e0e0; }
.card-header { display: flex; justify-content: space-between; align-items: center; font-size: 1.1em; font-weight: 500; }
.models-popover .el-tag { margin: 3px;}
.chat-interface { display: flex; flex-direction: column; height: calc(100vh - 60px - 40px); background-color: #fff; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.1); overflow: hidden; }
.chat-messages-container { flex-grow: 1; overflow-y: auto; padding: 20px; background-color: #f9f9f9; }

.message-wrapper {
  display: flex;
  flex-direction: column;
  margin-bottom: 15px;
  max-width: 85%;
}
.ai-message-wrapper, .observer-message-wrapper, .custom-role-message-wrapper {
  align-items: flex-start;
  margin-right: auto;
}
.user-message-wrapper {
  align-items: flex-end;
  margin-left: auto;
}

.message-meta {
  width: 100%;
  margin-bottom: 4px;
}
.ai-message-wrapper .message-meta, .observer-message-wrapper .message-meta, .custom-role-message-wrapper .message-meta {
  text-align: left;
}
.user-message-wrapper .message-meta {
  text-align: right;
}

.message-role-display {
  font-size: 0.8em;
  font-weight: 600;
  color: #555;
  padding: 0 5px;
}

.message-bubble {
  padding: 10px 15px;
  border-radius: 18px;
  line-height: 1.5;
  word-break: break-word;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  max-width: 100%;
}
.user-message-wrapper .message-bubble {
  background-color: #DCF8C6;
  color: #333;
}
.ai-message-wrapper .message-bubble {
  background-color: #FFFFFF;
  color: #333;
  border: 1px solid #eee;
}
.observer-message-wrapper .message-bubble, .observer-bubble {
  background-color: #e6f7ff;
  color: #303133;
  border: 1px solid #d9ecff;
}
.custom-role-message-wrapper .message-bubble, .custom-role-bubble {
  background-color: #f0f2f5;
  color: #303133;
  border: 1px solid #e4e7ed;
}


.message-content > *:first-child { margin-top: 0; }
.message-content > *:last-child { margin-bottom: 0; }
.message-content p { margin-top: 0; margin-bottom: 0.75em; }
.message-content ul, .message-content ol { padding-left: 20px; margin-top: 0.5em; margin-bottom: 0.5em; }
.message-content li { margin-bottom: 0.25em; }
.message-content pre { background-color: #f0f0f0; padding: 10px; border-radius: 4px; overflow-x: auto; margin: 0.75em 0; font-size: 0.9em; line-height: 1.4; }
.message-content code { background-color: #f0f0f0; padding: 2px 5px; border-radius: 3px; font-size: 0.9em; }
.message-content pre code { background-color: transparent; padding: 0; border-radius: 0; font-size: inherit; }
.message-content blockquote { border-left: 3px solid #ccc; padding-left: 10px; margin-left: 0; margin-right: 0; margin-top: 0.75em; margin-bottom: 0.75em; color: #666; font-style: italic; }
.message-content table { border-collapse: collapse; margin: 1em 0; width: auto; font-size: 0.9em; }
.message-content th, .message-content td { border: 1px solid #ddd; padding: 8px; text-align: left; }
.message-content th { background-color: #f9f9f9; font-weight: bold; }
.message-content img { max-width: 100%; height: auto; border-radius: 4px; margin: 0.5em 0; }
.message-content a { color: #007bff; text-decoration: none; }
.message-content a:hover { text-decoration: underline; }
.message-content hr.turn-separator { border: 0; height: 1px; background-color: #ddd; margin: 10px 0; }

.chat-input-area { padding: 15px 20px; border-top: 1px solid #e0e0e0; background-color: #ffffff; }
.chat-actions { display: flex; flex-wrap: wrap; gap: 10px; margin-bottom: 10px; align-items: center;}
.user-input-row { display: flex; align-items: flex-end; gap: 10px; }
.role-input { width: 180px !important; flex-shrink: 0; }
.message-input .el-textarea__inner { border-radius: 18px; padding: 8px 15px; line-height: 1.5; }
.send-button { height: var(--el-input-height, 40px); border-radius: 18px !important; padding: 0 15px !important; }
.send-button .el-icon { font-size: 1.2em; }
.el-dialog__body { padding: 20px 25px; }
.el-form-item { margin-bottom: 18px; }
.el-button { border-radius: 6px; }
.el-table th { background-color: #fafafa !important; }
</style>