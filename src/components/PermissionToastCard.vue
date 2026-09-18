<script setup lang="ts">
import { computed, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { openAgentSession, resolvePermission } from '../api/tauri'

const { t } = useI18n()

export interface PermissionItem {
  requestId: number
  toolName: string
  toolInput?: unknown
  sessionId?: string
  cwd?: string
}

const props = defineProps<{
  item: PermissionItem
}>()

const emit = defineEmits<{
  /** 卡片生命周期结束（已决策/已超时/前往终端），从通知栈移除 */
  (e: 'close'): void
}>()

// 决策状态机：pending → deciding(防重复) → decided / timedout
const state = ref<'pending' | 'deciding' | 'decided' | 'timedout'>('pending')
const decidedLabel = ref('')

function projectName(cwd?: string): string {
  if (!cwd) return ''
  const parts = cwd.replace(/\\/g, '/').split('/').filter(Boolean)
  return parts[parts.length - 1] || ''
}

/** 把 tool_input 提炼成一行人类可读摘要：bash 取 command，文件类取路径，其余 JSON 截断。 */
const toolSummary = computed(() => {
  const input = props.item.toolInput
  if (input == null) return ''
  if (typeof input === 'string') return truncate(input, 120)
  if (typeof input === 'object') {
    const obj = input as Record<string, unknown>
    for (const key of ['command', 'file_path', 'path', 'pattern', 'url', 'query']) {
      const v = obj[key]
      if (typeof v === 'string' && v) return truncate(v, 120)
    }
    try {
      return truncate(JSON.stringify(input), 120)
    } catch {
      return ''
    }
  }
  return truncate(String(input), 120)
})

function truncate(s: string, max: number): string {
  const first = s.split('\n')[0].trim()
  return first.length > max ? `${first.slice(0, max)}…` : first
}

async function decide(decision: 'allow' | 'deny') {
  if (state.value !== 'pending') return
  state.value = 'deciding'
  try {
    const accepted = await resolvePermission(props.item.requestId, decision)
    if (!accepted) {
      // 后端已超时/不存在：提示用户回终端，短暂展示后关闭
      state.value = 'timedout'
      setTimeout(() => emit('close'), 1800)
      return
    }
    decidedLabel.value = decision === 'allow' ? t('agent.permissionAllow') : t('agent.permissionDeny')
    state.value = 'decided'
    setTimeout(() => emit('close'), 700)
  } catch {
    state.value = 'pending' // invoke 失败允许重试
  }
}

async function gotoTerminal() {
  if (state.value === 'deciding') return
  state.value = 'deciding'
  // 通知后端放弃这次审批（timeout 决策 → Claude 回退终端），再打开终端让用户处理
  try {
    await resolvePermission(props.item.requestId, 'timeout')
  } catch {
    // 已超时也无妨，终端仍开
  }
  try {
    if (props.item.sessionId && props.item.sessionId !== 'unknown') {
      await openAgentSession(props.item.cwd || '', props.item.sessionId)
    }
  } catch {
    // 终端打开失败保留卡片
    state.value = 'pending'
    return
  }
  emit('close')
}
</script>

<template>
  <div class="perm-card">
    <div class="header">
      <div class="header-left">
        <div class="pulse-dot" />
        <h2 class="title">{{ t('agent.permissionTitle') }}</h2>
      </div>
      <span v-if="projectName(item.cwd)" class="project">{{ projectName(item.cwd) }}</span>
    </div>

    <div class="tool-block">
      <span class="tool-name">{{ item.toolName || 'tool' }}</span>
      <p v-if="toolSummary" class="tool-summary">{{ toolSummary }}</p>
    </div>

    <!-- 决策按钮 -->
    <div v-if="state === 'pending' || state === 'deciding'" class="actions">
      <button class="btn btn-allow" :disabled="state === 'deciding'" @click="decide('allow')">
        {{ t('agent.permissionAllow') }}
      </button>
      <button class="btn btn-deny" :disabled="state === 'deciding'" @click="decide('deny')">
        {{ t('agent.permissionDeny') }}
      </button>
      <button class="btn btn-goto" :disabled="state === 'deciding'" @click="gotoTerminal">
        {{ t('agent.permissionGoto') }}
      </button>
    </div>

    <!-- 结果态 -->
    <div v-else class="result" :class="{ timedout: state === 'timedout' }">
      {{ state === 'timedout' ? t('agent.permissionTimedOut') : `${t('agent.permissionDecided')} · ${decidedLabel}` }}
    </div>
  </div>
</template>

<style scoped>
.perm-card {
  display: flex;
  flex-direction: column;
  width: 100%;
  min-height: 0;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 0.375rem;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  min-width: 0;
}

.pulse-dot {
  width: 0.5rem;
  height: 0.5rem;
  border-radius: 50%;
  background: var(--ct-warning);
  animation: pulse 1.2s ease-in-out infinite;
  flex-shrink: 0;
}

@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(1.3); }
}

.title {
  font-size: 0.875rem;
  font-weight: 700;
  color: var(--ct-warning);
  margin: 0;
}

.project {
  font-size: 0.6875rem;
  color: var(--ct-warning-strong);
  opacity: 0.8;
}

.tool-block {
  background: var(--ct-warning-soft);
  border: 0.0625rem solid var(--ct-warning-soft);
  border-radius: 0.375rem;
  padding: 0.5rem 0.625rem;
  margin-bottom: 0.625rem;
}

.tool-name {
  display: inline-block;
  font-size: 0.75rem;
  font-weight: 700;
  color: var(--ct-warning);
  background: var(--ct-warning-soft);
  border-radius: 0.25rem;
  padding: 0.0625rem 0.375rem;
  margin-bottom: 0.25rem;
}

.tool-summary {
  font-size: 0.75rem;
  color: var(--ct-warning-strong);
  font-family: monospace;
  word-break: break-all;
  margin: 0;
  line-height: 1.4;
}

.actions {
  display: flex;
  gap: 0.375rem;
}

.btn {
  flex: 1;
  height: 1.75rem;
  border-radius: 0.375rem;
  font-size: 0.75rem;
  font-weight: 600;
  cursor: pointer;
  border: none;
  transition: all 0.2s ease;
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-allow {
  background: var(--ct-success);
  color: var(--ct-on-accent);
}
.btn-allow:hover:not(:disabled) {
  background: var(--ct-success);
}

.btn-deny {
  background: var(--ct-error);
  color: var(--ct-on-accent);
}
.btn-deny:hover:not(:disabled) {
  background: var(--ct-error);
}

.btn-goto {
  background: var(--ct-surface-2);
  color: var(--ct-warning-strong);
  border: 0.0625rem solid var(--ct-warning-soft);
}
.btn-goto:hover:not(:disabled) {
  background: var(--ct-warning-soft);
}

.result {
  font-size: 0.75rem;
  font-weight: 600;
  color: var(--ct-success);
  text-align: center;
  padding: 0.375rem 0;
}

.result.timedout {
  color: var(--ct-warning-strong);
}
</style>
