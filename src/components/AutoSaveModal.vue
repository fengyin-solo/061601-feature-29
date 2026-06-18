<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useSaveStore } from '../stores/saveStore'
import type { AutoSaveSummary } from '../stores/saveStore'

const emit = defineEmits<{
  (e: 'restored'): void
  (e: 'abandoned'): void
}>()

const saveStore = useSaveStore()

type ModalState = 'choose' | 'confirm-abandon' | 'abandon-done'

const modalState = ref<ModalState>('choose')
const autoSaveSummary = ref<AutoSaveSummary | null>(null)
const initialSummary = ref<AutoSaveSummary | null>(null)

onMounted(() => {
  autoSaveSummary.value = saveStore.getAutoSaveSummary()
  initialSummary.value = saveStore.getInitialSummary()
})

interface DiffItem {
  label: string
  saveValue: string | number
  initialValue: string | number
  saveBetter: boolean | null
}

const diffItems = computed<DiffItem[]>(() => {
  if (!autoSaveSummary.value || !initialSummary.value) return []
  const s = autoSaveSummary.value
  const i = initialSummary.value

  return [
    {
      label: '游戏天数',
      saveValue: `第 ${s.day} 天`,
      initialValue: `第 ${i.day} 天`,
      saveBetter: s.day > i.day
    },
    {
      label: '当前时段',
      saveValue: s.timeSlot,
      initialValue: i.timeSlot,
      saveBetter: null
    },
    {
      label: '代币',
      saveValue: s.resources,
      initialValue: i.resources,
      saveBetter: s.resources > i.resources
    },
    {
      label: '解锁角色',
      saveValue: `${s.unlockedCount} 人`,
      initialValue: `${i.unlockedCount} 人`,
      saveBetter: s.unlockedCount > i.unlockedCount
    },
    {
      label: '收集卡牌',
      saveValue: `${s.cardCount} 张`,
      initialValue: `${i.cardCount} 张`,
      saveBetter: s.cardCount > i.cardCount
    },
    {
      label: '触发事件',
      saveValue: `${s.eventCount} 次`,
      initialValue: `${i.eventCount} 次`,
      saveBetter: s.eventCount > i.eventCount
    }
  ]
})

function getAffinityBarWidth(affinity: number): string {
  return `${Math.max(0, Math.min(100, affinity))}%`
}

function getMoodEmoji(mood: number): string {
  if (mood >= 80) return '😄'
  if (mood >= 60) return '🙂'
  if (mood >= 40) return '😐'
  if (mood >= 20) return '😕'
  return '😢'
}

function restoreSave() {
  saveStore.loadAutoSave()
  emit('restored')
}

function requestAbandon() {
  modalState.value = 'confirm-abandon'
}

function cancelAbandon() {
  modalState.value = 'choose'
}

function confirmAbandon() {
  saveStore.clearAutoSave()
  saveStore.toggleAutoSave(false)
  modalState.value = 'abandon-done'
}

function finishAbandon() {
  emit('abandoned')
}
</script>

<template>
  <Teleport to="body">
    <div class="modal-overlay">
      <div class="modal-content autosave-modal">
        <div v-if="modalState === 'choose'" class="choose-state">
          <div class="modal-header">
            <h2>📂 检测到自动存档</h2>
          </div>

          <div v-if="autoSaveSummary" class="save-summary">
            <div class="summary-header">
              <span class="summary-badge">上次进度</span>
              <span class="summary-time">{{ autoSaveSummary.formattedTime }}</span>
            </div>

            <div class="summary-overview">
              <div class="overview-item">
                <span class="overview-icon">📅</span>
                <div class="overview-text">
                  <span class="overview-value">第 {{ autoSaveSummary.day }} 天</span>
                  <span class="overview-label">{{ autoSaveSummary.timeSlot }}</span>
                </div>
              </div>
              <div class="overview-item">
                <span class="overview-icon">💰</span>
                <div class="overview-text">
                  <span class="overview-value">{{ autoSaveSummary.resources }}</span>
                  <span class="overview-label">代币</span>
                </div>
              </div>
              <div class="overview-item">
                <span class="overview-icon">👥</span>
                <div class="overview-text">
                  <span class="overview-value">{{ autoSaveSummary.unlockedCount }}</span>
                  <span class="overview-label">解锁角色</span>
                </div>
              </div>
              <div class="overview-item">
                <span class="overview-icon">🎴</span>
                <div class="overview-text">
                  <span class="overview-value">{{ autoSaveSummary.cardCount }}</span>
                  <span class="overview-label">收集卡牌</span>
                </div>
              </div>
            </div>

            <div v-if="autoSaveSummary.topCharacters.length > 0" class="character-section">
              <div class="section-title">角色好感度 TOP 3</div>
              <div class="character-list">
                <div 
                  v-for="(char, idx) in autoSaveSummary.topCharacters" 
                  :key="idx"
                  class="character-row"
                >
                  <span class="char-rank">{{ idx + 1 }}</span>
                  <span class="char-name">{{ char.name }}</span>
                  <div class="char-affinity-bar">
                    <div 
                      class="affinity-fill"
                      :style="{ width: getAffinityBarWidth(char.affinity) }"
                    ></div>
                  </div>
                  <span class="char-affinity-num">{{ char.affinity }}</span>
                  <span class="char-mood" :title="`心情值: ${char.mood}`">
                    {{ getMoodEmoji(char.mood) }}
                  </span>
                </div>
              </div>
            </div>
          </div>

          <div class="diff-section">
            <div class="section-title">📊 对比：恢复存档 vs 重新开始</div>
            <div class="diff-table">
              <div class="diff-header">
                <span></span>
                <span class="diff-col-save">💾 恢复存档</span>
                <span class="diff-col-init">🆕 重新开始</span>
              </div>
              <div 
                v-for="(item, idx) in diffItems" 
                :key="idx"
                class="diff-row"
              >
                <span class="diff-label">{{ item.label }}</span>
                <span 
                  class="diff-value diff-col-save"
                  :class="{ 'value-better': item.saveBetter === true }"
                >
                  {{ item.saveValue }}
                </span>
                <span 
                  class="diff-value diff-col-init"
                  :class="{ 'value-better': item.saveBetter === false }"
                >
                  {{ item.initialValue }}
                </span>
              </div>
            </div>
          </div>

          <div class="action-buttons">
            <button class="btn btn-restore" @click="restoreSave">
              <span>📂</span> 恢复存档继续游戏
            </button>
            <button class="btn btn-abandon" @click="requestAbandon">
              <span>🗑️</span> 放弃存档重新开始
            </button>
          </div>
        </div>

        <div v-else-if="modalState === 'confirm-abandon'" class="confirm-state">
          <div class="confirm-icon">⚠️</div>
          <h2 class="confirm-title">确认放弃存档？</h2>
          <div class="confirm-message">
            <p>放弃后，以下内容将被<strong>永久删除</strong>且无法恢复：</p>
            <ul v-if="autoSaveSummary">
              <li>第 {{ autoSaveSummary.day }} 天的游戏进度</li>
              <li>{{ autoSaveSummary.resources }} 代币</li>
              <li>{{ autoSaveSummary.unlockedCount }} 个解锁角色的好感度</li>
              <li>{{ autoSaveSummary.cardCount }} 张收藏卡牌</li>
              <li>{{ autoSaveSummary.eventCount }} 次事件记录</li>
            </ul>
            <p class="confirm-note">自动存档功能也将被关闭，可在设置中重新开启。</p>
          </div>
          <div class="action-buttons">
            <button class="btn btn-restore" @click="cancelAbandon">
              取消，保留存档
            </button>
            <button class="btn btn-danger" @click="confirmAbandon">
              <span>💥</span> 确认放弃
            </button>
          </div>
        </div>

        <div v-else-if="modalState === 'abandon-done'" class="done-state">
          <div class="done-icon">✅</div>
          <h2 class="done-title">存档已清理</h2>
          <div class="done-message">
            <p>自动存档已成功删除，自动存档功能已关闭。</p>
            <p>即将开始全新的游戏旅程...</p>
          </div>
          <div class="action-buttons">
            <button class="btn btn-restore" @click="finishAbandon">
              <span>🎮</span> 开始新游戏
            </button>
          </div>
        </div>
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
.autosave-modal {
  padding: 28px;
  max-width: 560px;
  width: 90%;
}

.modal-header {
  margin-bottom: 20px;
}

.modal-header h2 {
  font-size: 22px;
  font-weight: 700;
  text-align: center;
}

.save-summary {
  background: linear-gradient(135deg, var(--accent-light), var(--bg-tertiary));
  border-radius: var(--radius-lg);
  padding: 18px;
  margin-bottom: 20px;
  border: 1px solid var(--accent-primary);
}

.summary-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 14px;
}

.summary-badge {
  background: var(--accent-primary);
  color: white;
  padding: 4px 12px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 600;
}

.summary-time {
  font-size: 13px;
  color: var(--text-secondary);
}

.summary-overview {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  margin-bottom: 16px;
}

.overview-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 10px 6px;
  background: var(--bg-secondary);
  border-radius: var(--radius-md);
}

.overview-icon {
  font-size: 22px;
}

.overview-text {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.overview-value {
  font-size: 16px;
  font-weight: 700;
  color: var(--text-primary);
}

.overview-label {
  font-size: 11px;
  color: var(--text-muted);
}

.section-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-secondary);
  margin-bottom: 10px;
}

.character-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.character-row {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 12px;
  background: var(--bg-secondary);
  border-radius: var(--radius-sm);
}

.char-rank {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: var(--accent-primary);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  font-weight: 700;
  flex-shrink: 0;
}

.char-name {
  font-size: 13px;
  font-weight: 600;
  width: 60px;
  flex-shrink: 0;
}

.char-affinity-bar {
  flex: 1;
  height: 8px;
  background: var(--bg-tertiary);
  border-radius: 999px;
  overflow: hidden;
}

.affinity-fill {
  height: 100%;
  background: linear-gradient(90deg, #fbbf24, #f97316, #ef4444);
  border-radius: 999px;
  transition: width 0.3s;
}

.char-affinity-num {
  font-size: 12px;
  font-weight: 700;
  color: var(--accent-primary);
  width: 30px;
  text-align: right;
}

.char-mood {
  font-size: 16px;
}

.diff-section {
  margin-bottom: 24px;
}

.diff-table {
  border: 1px solid var(--bg-tertiary);
  border-radius: var(--radius-md);
  overflow: hidden;
}

.diff-header {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  padding: 10px 12px;
  background: var(--bg-tertiary);
  font-size: 12px;
  font-weight: 600;
  color: var(--text-secondary);
}

.diff-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  padding: 10px 12px;
  border-top: 1px solid var(--bg-tertiary);
  font-size: 13px;
  align-items: center;
}

.diff-label {
  color: var(--text-secondary);
  font-weight: 500;
}

.diff-value {
  text-align: center;
  font-weight: 600;
}

.diff-col-save {
  color: var(--accent-primary);
}

.diff-col-init {
  color: var(--text-muted);
}

.value-better {
  position: relative;
}

.value-better::after {
  content: '✓';
  position: absolute;
  right: -6px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 10px;
  color: #22c55e;
}

.action-buttons {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 14px 20px;
  border-radius: var(--radius-md);
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  border: none;
}

.btn-restore {
  background: var(--accent-primary);
  color: white;
}

.btn-restore:hover {
  background: var(--accent-dark);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.btn-abandon {
  background: var(--bg-tertiary);
  color: var(--text-secondary);
}

.btn-abandon:hover {
  background: #fee2e2;
  color: #dc2626;
}

.btn-danger {
  background: #ef4444;
  color: white;
}

.btn-danger:hover {
  background: #dc2626;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(239, 68, 68, 0.3);
}

.confirm-state,
.done-state {
  text-align: center;
  padding: 10px 0;
}

.confirm-icon,
.done-icon {
  font-size: 56px;
  margin-bottom: 16px;
}

.confirm-title,
.done-title {
  font-size: 22px;
  font-weight: 700;
  margin-bottom: 16px;
}

.confirm-title {
  color: #dc2626;
}

.done-title {
  color: #16a34a;
}

.confirm-message,
.done-message {
  text-align: left;
  background: var(--bg-tertiary);
  padding: 16px 20px;
  border-radius: var(--radius-md);
  margin-bottom: 24px;
  font-size: 14px;
  line-height: 1.7;
}

.confirm-message ul {
  margin: 10px 0;
  padding-left: 20px;
}

.confirm-message li {
  margin: 4px 0;
}

.confirm-note {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px dashed var(--bg-secondary);
  color: var(--text-muted);
  font-size: 13px;
}

.done-message {
  text-align: center;
}

@media (max-width: 520px) {
  .summary-overview {
    grid-template-columns: repeat(2, 1fr);
  }
}
</style>
