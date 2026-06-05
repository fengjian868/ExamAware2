<template>
  <div class="exam-container" ref="rootRef">
    <!-- 背景渐变椭圆 -->
    <div class="background-ellipse"></div>

    <!-- 主要内容 -->
    <div class="content-wrapper">
      <!-- 顶部标题栏 -->
      <div class="top-header">
        <div class="header-left">
          <h1 ref="mainTitleRef" class="main-title">
            {{ playerExamConfig?.examName || '考试' }}
          </h1>
          <p ref="subtitleRef" class="subtitle">
            {{ playerExamConfig?.message || '请遵守考场纪律' }}
          </p>
        </div>
        <div class="header-right">
          <ExamRoomNumber :room-number="effectiveRoomNumber" @click="handleRoomNumberClick" />
        </div>
      </div>

      <!-- 中间区域：北京时间 + 倒计时 -->
      <div class="middle-section">
        <div class="clock-area">
          <div class="clock-label">北京时间</div>
          <div class="time-display">{{ formattedCurrentTime }}</div>
          <div class="countdown-display">
            <span v-if="showPreExamCountdown">考前倒计时 {{ preExamCountdownText }}</span>
            <span v-else-if="examStatus?.status === 'pending'">考试未开始</span>
            <span v-else-if="examStatus?.status === 'inProgress'"
              >考试倒计时 {{ remainingTime }}</span
            >
            <span v-else>考试已结束</span>
          </div>
        </div>
      </div>

      <!-- 底部区域：左右分栏 -->
      <div class="bottom-section">
        <!-- 左侧：当前考试信息 -->
        <div class="bottom-left">
          <BaseCard custom-class="current-exam-card">
            <div class="current-exam-content">
              <div class="info-row">
                <span class="info-label">当前科目：</span>
                <span class="info-value">{{ currentExamName }}</span>
              </div>
              <div class="info-row">
                <span class="info-label">考试时间：</span>
                <span class="info-value">{{ currentExamTimeRange }}</span>
              </div>
              <div class="info-row">
                <span class="info-label">考试状态：</span>
                <span class="info-value" :class="statusClass">{{ examStatusText }}</span>
              </div>

              <!-- 考前倒计时（在考前指定时间内显示） -->
              <div v-if="showPreExamCountdown" class="info-row">
                <span class="info-label">考前倒计时：</span>
                <span class="info-value pre-exam-countdown">{{ preExamCountdownText }}</span>
              </div>

              <!-- 考试材料数量控制 -->
              <div v-if="currentExam?.materials?.length" class="materials-section">
                <div
                  v-for="(material, idx) in currentExam.materials"
                  :key="material.name"
                  class="material-row"
                >
                  <span class="info-label">{{ material.name }}：</span>
                  <div class="material-control">
                    <span class="material-text">共</span>
                    <div class="number-box">
                      <button class="num-btn" @click="decreaseMaterial(idx, 'pages')">-</button>
                      <span class="num-value">{{ materialPages[idx] ?? 1 }}</span>
                      <button class="num-btn" @click="increaseMaterial(idx, 'pages')">+</button>
                    </div>
                    <span class="material-text">页</span>
                    <span class="material-text">共</span>
                    <div class="number-box">
                      <button class="num-btn" @click="decreaseMaterial(idx, 'quantity')">-</button>
                      <span class="num-value">{{
                        materialQuantities[idx] ?? material.quantity
                      }}</span>
                      <button class="num-btn" @click="increaseMaterial(idx, 'quantity')">+</button>
                    </div>
                    <span class="material-text">张</span>
                  </div>
                </div>
              </div>
            </div>
          </BaseCard>
        </div>

        <!-- 右侧：考试列表表格 -->
        <div class="bottom-right">
          <BaseCard custom-class="exam-list-card">
            <div class="exam-table-header">
              <span>日期</span>
              <span>科目</span>
              <span>开始时间</span>
              <span>结束时间</span>
              <span>考试状态</span>
            </div>
            <div class="exam-table-body">
              <div
                v-for="exam in displayFormattedExamInfos"
                :key="exam.name"
                class="exam-table-row"
                :class="{ 'exam-active': exam.status === 'inProgress' }"
              >
                <span>{{ exam.date }}</span>
                <span>{{ exam.name }}</span>
                <span>{{ exam.timeRange.split(' ~ ')[0] }}</span>
                <span>{{ exam.timeRange.split(' ~ ')[1] }}</span>
                <span :class="`status-${exam.status}`">{{ exam.statusText }}</span>
              </div>
              <div v-if="!displayFormattedExamInfos?.length" class="empty-state">暂无考试安排</div>
            </div>
          </BaseCard>
        </div>
      </div>
    </div>

    <!-- 底部按钮栏 -->
    <ActionButtonBar
      v-if="showActionBar"
      :initial-scale="props.uiScale"
      :initial-density="densityState"
      :initial-large-clock-enabled="largeClockState"
      :initial-large-clock-scale="largeClockScaleState"
      :initial-exam-info-large-font="examInfoLargeFontState"
      :extra-tools="toolbarTools"
      @exit="emit('exit')"
      @scale-change="emit('scaleChange', $event)"
      @density-change="handleDensityChange"
      @large-clock-toggle="handleLargeClockToggle"
      @clock-scale-change="handleLargeClockScaleChange"
      @exam-info-large-font-toggle="handleExamInfoLargeFontToggle"
      @dev-reminder-test="handleDevReminderTest"
      @dev-reminder-hide="handleDevReminderHide"
    />

    <!-- 彩色提醒 -->
    <transition name="fade-soft">
      <div
        v-if="colorfulVisible"
        class="overlay colorful-overlay"
        :class="{ 'hdr-highlight': colorfulHdrActive }"
        :style="colorfulOverlayStyle"
      >
        <div class="colorful-title">{{ colorfulTitle }}</div>
      </div>
    </transition>

    <!-- 普通提醒 -->
    <transition name="fade-soft">
      <div v-if="currentNotice" class="overlay notice-overlay">
        <div class="notice-card">
          <div class="notice-content" v-html="renderedMarkdown"></div>
          <t-button theme="primary" size="large" @click="handleCloseNotice">
            关闭（{{ currentNotice?.remainingSec }}s）
          </t-button>
        </div>
      </div>
    </transition>

    <!-- 考场号设置弹窗 -->
    <t-dialog
      header="设置考场号"
      v-model:visible="showRoomNumberModal"
      :footer="true"
      @cancel="handleRoomNumberCancel"
      @esc-keydown="handleRoomNumberCancel"
      @close-btn-click="handleRoomNumberCancel"
      @close="handleRoomNumberCancel"
      @confirm="handleRoomNumberConfirm"
    >
      <template #body>
        <t-input v-model="tempRoomNumber" type="text" placeholder="请输入考场号" maxlength="10" />
        <div class="keyboard-container">
          <div ref="keyboardRef" class="virtual-keyboard"></div>
        </div>
      </template>
    </t-dialog>

    <slot name="extra"></slot>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, onMounted, onUnmounted, watch, watchEffect, provide } from 'vue';
type ExamConfig = {
  examName: string;
  message: string;
  examInfos: any[];
};
import { useExamPlayer, type TimeProvider } from '../useExamPlayer';
import type { PlayerConfig, PlayerEventHandlers } from '../types';
import 'simple-keyboard/build/css/index.css';
import BaseCard from './BaseCard.vue';
import ExamRoomNumber from './ExamRoomNumber.vue';
import ActionButtonBar from './ActionButtonBar.vue';
import { providePlayerToolbar } from '../composables/usePlayerToolbar';
import { Dialog as TDialog, Input as TInput, Button as TButton } from 'tdesign-vue-next';
import { useReminderService, ReminderUtils } from '../utils/reminderService';

const renderMarkdownLight = (md: string): string => {
  let html = md;
  html = html.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
  html = html.replace(/^##\s+(.+)$/gm, '<h2>$1</h2>');
  html = html.replace(/^#\s+(.+)$/gm, '<h1>$1</h1>');
  html = html.replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>');
  html = html.replace(/\*(.+?)\*/g, '<em>$1</em>');
  html = html.replace(/`([^`]+)`/g, '<code>$1</code>');
  html = html.replace(/\n/g, '<br/>');
  return html;
};

const rootRef = ref<HTMLElement | null>(null);

const toolbarRegistry = providePlayerToolbar();
const toolbarTools = toolbarRegistry.tools;

type UIDensity = 'comfortable' | 'cozy' | 'compact';

interface Props {
  examConfig: ExamConfig | null;
  config?: PlayerConfig;
  uiScale?: number;
  uiDensity?: UIDensity;
  examInfoLargeFont?: boolean;
  timeProvider?: TimeProvider;
  timeSyncStatus?: string;
  roomNumber?: string;
  showActionBar?: boolean;
  hdrHighlight?: boolean;
  largeClock?: boolean;
  largeClockScale?: number;
  allowEditRoomNumber?: boolean;
  eventHandlers?: PlayerEventHandlers;
}

interface Emits {
  (e: 'roomNumberClick'): void;
  (e: 'roomNumberChange', roomNumber: string): void;
  (e: 'update:roomNumber', roomNumber: string): void;
  (e: 'update:largeClock', enabled: boolean): void;
  (e: 'update:examInfoLargeFont', enabled: boolean): void;
  (e: 'exit'): void;
  (e: 'scaleChange', scale: number): void;
  (e: 'largeClockToggle', enabled: boolean): void;
  (e: 'largeClockScaleChange', scale: number): void;
  (e: 'examInfoLargeFontToggle', enabled: boolean): void;
  (e: 'densityChange', density: UIDensity): void;
  (e: 'examStart', exam: any): void;
  (e: 'examEnd', exam: any): void;
  (e: 'examAlert', exam: any, alertTime: number): void;
  (e: 'examSwitch', fromExam: any, toExam: any): void;
  (e: 'error', error: string): void;
}

const props = withDefaults(defineProps<Props>(), {
  config: () => ({ roomNumber: '01' }),
  uiScale: undefined,
  largeClockScale: undefined,
  examInfoLargeFont: false,
  timeProvider: () => ({ getCurrentTime: () => Date.now() }),
  timeSyncStatus: '电脑时间',
  roomNumber: '01',
  showActionBar: true,
  hdrHighlight: false,
  largeClock: false,
  allowEditRoomNumber: true,
  eventHandlers: () => ({}),
  uiDensity: 'comfortable'
});

const emit = defineEmits<Emits>();
const reminder = useReminderService();

const reminderShown = new Set<string>();

const getExamKey = (exam: any): string => {
  const raw = exam?.id ?? exam?.name;
  if (raw === undefined || raw === null) return '';
  return String(raw);
};

const showColorfulOnce = (
  key: string,
  options: { title: string; themeBaseColor: string; forceWhiteText?: boolean }
) => {
  if (!key || reminderShown.has(key)) return;
  reminderShown.add(key);
  reminder.showColorfulAlert(options);
};

const showExamReminder = (
  kind: 'start' | 'end' | 'alert',
  exam: any,
  options: { title: string; themeBaseColor: string; forceWhiteText?: boolean }
) => {
  const examKey = getExamKey(exam);
  if (!examKey) return;
  showColorfulOnce(`${kind}:${examKey}`, options);
};

const mergedEventHandlers: PlayerEventHandlers = {
  ...props.eventHandlers,
  onExamStart: (exam: any) => {
    props.eventHandlers?.onExamStart?.(exam);
    emit('examStart', exam);
    showExamReminder('start', exam, { title: '考试开始', themeBaseColor: '#2ecc71' });
  },
  onExamEnd: (exam: any) => {
    props.eventHandlers?.onExamEnd?.(exam);
    emit('examEnd', exam);
    showExamReminder('end', exam, { title: '考试结束', themeBaseColor: '#ff3b30' });
  },
  onExamAlert: (exam: any, alertTime: number) => {
    props.eventHandlers?.onExamAlert?.(exam, alertTime);
    emit('examAlert', exam, alertTime);
    showExamReminder('alert', exam, {
      title: '考试即将结束',
      themeBaseColor: '#f1c40f',
      forceWhiteText: true
    });
  },
  onExamSwitch: (fromExam: any, toExam: any) => {
    props.eventHandlers?.onExamSwitch?.(fromExam, toExam);
    emit('examSwitch', fromExam, toExam);
  },
  onError: (error: string) => {
    props.eventHandlers?.onError?.(error);
    emit('error', error);
  }
};

const densityState = ref<UIDensity>(props.uiDensity ?? 'comfortable');
const densityFactorMap: Record<UIDensity, number> = {
  comfortable: 1,
  cozy: 0.85,
  compact: 0.7
};
const densityFactor = computed(() => densityFactorMap[densityState.value] ?? 1);

const clampLargeClockScale = (value: unknown) => {
  const num = Number(value);
  if (!Number.isFinite(num)) return 1;
  return Math.min(1.8, Math.max(0.5, num));
};

const largeClockState = ref<boolean>(Boolean(props.largeClock));
const resolveInitialLargeClockScale = () => {
  if (props.largeClockScale !== undefined && props.largeClockScale !== null) {
    return clampLargeClockScale(props.largeClockScale);
  }
  return 1;
};
const largeClockScaleState = ref<number>(resolveInitialLargeClockScale());

const examInfoLargeFontState = ref<boolean>(Boolean(props.examInfoLargeFont));

watch(
  () => props.largeClockScale,
  (value) => {
    if (value === undefined || value === null) return;
    const safe = clampLargeClockScale(value);
    if (safe !== largeClockScaleState.value) {
      largeClockScaleState.value = safe;
    }
  }
);

const examPlayer = useExamPlayer(
  props.examConfig,
  props.config || { roomNumber: props.roomNumber || '01' },
  props.timeProvider || { getCurrentTime: () => Date.now() },
  mergedEventHandlers
);

watch(
  () => props.examConfig,
  (newConfig) => {
    console.log('ExamPlayer: 配置变化', newConfig);
    examPlayer.updateConfig(newConfig);
  },
  { immediate: false, deep: true }
);

watch(
  () => props.uiDensity,
  (val) => {
    if (!val) return;
    densityState.value = val;
  }
);

watch(
  () => props.largeClock,
  (next) => {
    largeClockState.value = Boolean(next);
  }
);

const setLargeClockScaleVar = (scale: number) => {
  const safe = clampLargeClockScale(scale);
  if (typeof document !== 'undefined') {
    document.documentElement.style.setProperty('--large-clock-scale', String(safe));
  }
  const root = rootRef.value;
  if (root) {
    root.style.setProperty('--large-clock-scale', String(safe));
  }
};

watch(
  largeClockScaleState,
  (value) => {
    const safe = clampLargeClockScale(value);
    setLargeClockScaleVar(safe);
    emit('largeClockScaleChange', safe);
  },
  { immediate: true }
);

watchEffect(() => {
  if (typeof window === 'undefined') return;
  const factor = densityFactor.value;
  document.documentElement.style.setProperty('--density-scale', String(factor));
  const root = rootRef.value;
  if (root) {
    root.style.setProperty('--density-scale', String(factor));
    root.dataset.density = densityState.value;
  }
});

watchEffect(() => {
  const root = rootRef.value;
  if (!root) return;
  root.dataset.largeClock = largeClockState.value ? 'true' : 'false';
});

watch(
  () => props.config,
  (newConfig) => {
    // 当 config 变化时，需要重新初始化 examPlayer
  },
  { deep: true }
);

watch(
  () => props.timeProvider,
  (newTimeProvider) => {
    if (newTimeProvider) {
      examPlayer.taskQueue.stop();
      examPlayer.taskQueue.start();
    }
  },
  { deep: true }
);

const {
  state,
  examConfig: playerExamConfig,
  currentExam,
  sortedExamInfos,
  formattedExamInfos,
  examStatus,
  currentExamName,
  currentExamTimeRange,
  remainingTime,
  formattedCurrentTime,
  switchToExam,
  updateConfig
} = examPlayer;

const handleDensityChange = (next: UIDensity) => {
  densityState.value = next;
  emit('densityChange', next);
};

const handleLargeClockToggle = (enabled: boolean) => {
  largeClockState.value = enabled;
  emit('update:largeClock', enabled);
  emit('largeClockToggle', enabled);
};

const handleLargeClockScaleChange = (scale: number) => {
  largeClockScaleState.value = scale;
};

const handleExamInfoLargeFontToggle = (enabled: boolean) => {
  const flag = Boolean(enabled);
  examInfoLargeFontState.value = flag;
  emit('update:examInfoLargeFont', flag);
  emit('examInfoLargeFontToggle', flag);
};

watch(
  () => props.examInfoLargeFont,
  (next) => {
    if (typeof next !== 'boolean') return;
    if (examInfoLargeFontState.value === next) return;
    examInfoLargeFontState.value = next;
  }
);

type DevReminderPreset = 'start' | 'warning' | 'end';
type DevReminderPayload = { title: string; themeBaseColor: string; forceWhiteText?: boolean };

const devReminderPresets: Record<DevReminderPreset, DevReminderPayload> = {
  start: { title: '考试开始', themeBaseColor: '#2ecc71' },
  warning: { title: '考试即将结束', themeBaseColor: '#f1c40f', forceWhiteText: true },
  end: { title: '考试结束', themeBaseColor: '#ff3b30' }
};

const resolveDevReminderPayload = (payload: DevReminderPreset | DevReminderPayload) => {
  if (typeof payload === 'string') {
    return devReminderPresets[payload] ?? devReminderPresets.start;
  }
  return payload;
};

const handleDevReminderTest = (payload: DevReminderPreset | DevReminderPayload) => {
  const resolved = resolveDevReminderPayload(payload);
  reminder.showColorfulAlert(resolved);
};

const handleDevReminderHide = () => {
  reminder.hideColorfulAlert();
};

const colorfulVisible = reminder.isColorfulVisible;
const colorfulTitle = computed(() => reminder._colorfulReminder.value?.title || '提示');
const colorfulOverlayStyle = computed(() => {
  const base = reminder._colorfulReminder.value?.themeBaseColor || '#ff3b30';
  const forceWhite = Boolean(reminder._colorfulReminder.value?.forceWhiteText);
  const text = forceWhite ? '#ffffff' : ReminderUtils.getContrastingTextColor(base);
  const shadow = forceWhite
    ? '0 10px 32px rgba(255, 255, 255, 0.55), 0 0 18px rgba(255, 255, 255, 0.45)'
    : '0 6px 24px rgba(0, 0, 0, 0.35)';
  return {
    '--colorful-bg': base,
    '--colorful-text': text,
    '--colorful-shadow': shadow
  } as Record<string, string>;
});
const colorfulHdrActive = computed(() => {
  if (!props.hdrHighlight) return false;
  return Boolean(reminder._colorfulReminder.value?.forceWhiteText);
});

const currentNotice = computed(() => reminder.currentNotice.value);
const renderedMarkdown = computed(() =>
  currentNotice.value ? renderMarkdownLight(currentNotice.value.markdown) : ''
);
const handleCloseNotice = () => reminder.closeCurrentNotice('manual');

defineExpose({
  showColorfulAlert: reminder.showColorfulAlert,
  hideColorfulAlert: reminder.hideColorfulAlert,
  showEndingAlert: reminder.showEndingAlert,
  hideEndingAlert: reminder.hideEndingAlert,
  notify: reminder.notify,
  closeCurrentNotice: reminder.closeCurrentNotice,
  clearAllNotices: reminder.clearAllNotices,
  toolbar: {
    register: toolbarRegistry.register,
    unregister: toolbarRegistry.unregister,
    clear: toolbarRegistry.clear
  }
});

let hasShownEndingForExamId: string | null = null;
watch(
  () => examStatus.value?.timeRemaining,
  (remainingMs) => {
    if (!currentExam.value) return;
    if (typeof remainingMs !== 'number') return;

    const alertMinutes = Number(currentExam.value.alertTime);
    if (!Number.isFinite(alertMinutes) || alertMinutes <= 0) return;

    const examId = currentExam.value?.id || currentExam.value?.name;
    const minutesLeft = remainingMs / (1000 * 60);
    if (
      examStatus.value?.status === 'inProgress' &&
      minutesLeft <= alertMinutes &&
      examId &&
      hasShownEndingForExamId !== examId
    ) {
      hasShownEndingForExamId = examId;
      showExamReminder('alert', currentExam.value, {
        title: '考试即将结束',
        themeBaseColor: '#f1c40f',
        forceWhiteText: true
      });
    }
  }
);

const lastStatusRef = ref<string | null>(null);
const lastExamKeyRef = ref<string | null>(null);

watch(
  () => [currentExam.value, examStatus.value?.status] as const,
  ([exam, status]) => {
    const examKey = getExamKey(exam);
    if (!examKey || !status) {
      lastExamKeyRef.value = examKey || null;
      lastStatusRef.value = status ?? null;
      return;
    }

    if (lastExamKeyRef.value !== examKey) {
      lastExamKeyRef.value = examKey;
      lastStatusRef.value = status;
      return;
    }

    if (status === 'inProgress' && lastStatusRef.value !== 'inProgress') {
      showExamReminder('start', exam, { title: '考试开始', themeBaseColor: '#2ecc71' });
    } else if (status === 'completed' && lastStatusRef.value !== 'completed') {
      showExamReminder('end', exam, { title: '考试结束', themeBaseColor: '#ff3b30' });
    }

    lastStatusRef.value = status;
  },
  { immediate: true }
);

// === 考前倒计时逻辑 ===
const DEFAULT_PRE_EXAM_MINUTES = 15;
const preExamCountdownMinutes = computed(() =>
  Math.max(1, Math.min(120, props.config?.preExamCountdownMinutes ?? DEFAULT_PRE_EXAM_MINUTES))
);

const preExamCountdownMs = computed(() => preExamCountdownMinutes.value * 60 * 1000);

const showPreExamCountdown = computed(() => {
  if (!currentExam.value || examStatus.value?.status !== 'pending') return false;
  const startTime = new Date(currentExam.value.start).getTime();
  const now = examPlayer.currentTime.value;
  const diff = startTime - now;
  return diff > 0 && diff <= preExamCountdownMs.value;
});

const preExamCountdownText = computed(() => {
  if (!currentExam.value) return '';
  const startTime = new Date(currentExam.value.start).getTime();
  const now = examPlayer.currentTime.value;
  const diff = Math.max(0, startTime - now);
  const hours = Math.floor(diff / (1000 * 60 * 60));
  const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));
  const seconds = Math.floor((diff % (1000 * 60)) / 1000);
  if (hours > 0) {
    return `${hours}:${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
  }
  return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
});

// === 考试状态文本 ===
const examStatusText = computed(() => {
  switch (examStatus.value?.status) {
    case 'pending':
      return '未开始';
    case 'inProgress':
      return '进行中';
    case 'completed':
      return '已结束';
    default:
      return '暂无安排';
  }
});

const statusClass = computed(() => {
  switch (examStatus.value?.status) {
    case 'pending':
      return 'status-pending';
    case 'inProgress':
      return 'status-ongoing';
    case 'completed':
      return 'status-finished';
    default:
      return '';
  }
});

// === 材料数量控制 ===
const materialPages = ref<Record<number, number>>({});
const materialQuantities = ref<Record<number, number>>({});

const increaseMaterial = (idx: number, type: 'pages' | 'quantity') => {
  if (type === 'pages') {
    materialPages.value[idx] = (materialPages.value[idx] ?? 1) + 1;
  } else {
    const defaultVal = currentExam.value?.materials?.[idx]?.quantity ?? 1;
    materialQuantities.value[idx] = (materialQuantities.value[idx] ?? defaultVal) + 1;
  }
};

const decreaseMaterial = (idx: number, type: 'pages' | 'quantity') => {
  if (type === 'pages') {
    const current = materialPages.value[idx] ?? 1;
    if (current > 0) materialPages.value[idx] = current - 1;
  } else {
    const defaultVal = currentExam.value?.materials?.[idx]?.quantity ?? 1;
    const current = materialQuantities.value[idx] ?? defaultVal;
    if (current > 0) materialQuantities.value[idx] = current - 1;
  }
};

// === 考场号设置 ===
const showRoomNumberModal = ref(false);
const STORAGE_KEY = 'examaware:roomNumber';

const loadStoredRoomNumber = (): string | null => {
  if (typeof window === 'undefined') return null;
  try {
    const v = window.localStorage.getItem(STORAGE_KEY);
    return v && v.trim() ? v.trim() : null;
  } catch {
    return null;
  }
};

const saveStoredRoomNumber = (val: string) => {
  if (typeof window === 'undefined') return;
  try {
    window.localStorage.setItem(STORAGE_KEY, val);
  } catch {}
};

const localRoomNumber = ref<string>(props.roomNumber || loadStoredRoomNumber() || '01');
const effectiveRoomNumber = computed<string>(() => props.roomNumber ?? localRoomNumber.value);

const tempRoomNumber = ref(effectiveRoomNumber.value);
const keyboardRef = ref<HTMLElement>();
let keyboardInstance: any = null;

const handleRoomNumberClick = () => {
  if (!props.allowEditRoomNumber) {
    emit('roomNumberClick');
    return;
  }

  tempRoomNumber.value = effectiveRoomNumber.value || '01';
  showRoomNumberModal.value = true;

  setTimeout(() => {
    initKeyboard();
  }, 100);
};

const onKeyPress = (button: string) => {
  if (button === '{clear}') {
    tempRoomNumber.value = '';
  } else if (button === '{bksp}') {
    tempRoomNumber.value = tempRoomNumber.value.slice(0, -1);
  } else {
    if (/^[0-9a-zA-Z]$/.test(button) && tempRoomNumber.value.length < 10) {
      tempRoomNumber.value += button;
    }
  }
};

const initKeyboard = () => {
  import('simple-keyboard')
    .then(({ default: Keyboard }) => {
      if (keyboardRef.value && !keyboardInstance) {
        keyboardInstance = new Keyboard(keyboardRef.value, {
          layout: {
            default: ['1 2 3', '4 5 6', '7 8 9', '{clear} 0 {bksp}']
          },
          display: {
            '{clear}': '清空',
            '{bksp}': '⌫ 删除'
          },
          theme: 'hg-theme-default hg-layout-numeric numeric-keyboard-dark',
          physicalKeyboardHighlight: false,
          syncInstanceInputs: false,
          mergeDisplay: true,
          onKeyPress: (button: string) => onKeyPress(button)
        });
      }
    })
    .catch((error) => {
      console.warn('Failed to load simple-keyboard:', error);
    });
};

const destroyKeyboard = () => {
  if (keyboardInstance) {
    keyboardInstance.destroy();
    keyboardInstance = null;
  }
};

const handleRoomNumberConfirm = () => {
  if (tempRoomNumber.value && tempRoomNumber.value.trim()) {
    const next = tempRoomNumber.value.trim();
    localRoomNumber.value = next;
    saveStoredRoomNumber(next);
    emit('update:roomNumber', next);
    emit('roomNumberChange', next);
    showRoomNumberModal.value = false;
    destroyKeyboard();
  } else {
    emit('error', '考场号不能为空');
  }
};

const handleRoomNumberCancel = () => {
  showRoomNumberModal.value = false;
  tempRoomNumber.value = effectiveRoomNumber.value || '01';
  destroyKeyboard();
};

const displayFormattedExamInfos = computed(() => {
  const formatted = formattedExamInfos.value || [];
  return formatted;
});

const displayedRemainingTime = computed(() => {
  return examStatus.value?.status === 'pending' ? '' : remainingTime.value || '';
});

onMounted(() => {
  console.log('ExamPlayer: mounted, props.examConfig:', props.examConfig);
  console.log('ExamPlayer: examPlayer state:', examPlayer.state.value);
  console.log('ExamPlayer: formattedExamInfos:', formattedExamInfos.value);
  const stored = loadStoredRoomNumber();
  if (stored && stored !== props.roomNumber) {
    localRoomNumber.value = stored;
    emit('update:roomNumber', stored);
    emit('roomNumberChange', stored);
  }

  setTimeout(() => {
    const status = examStatus.value?.status;
    if (status === 'inProgress') {
      reminder.showColorfulAlert({ title: '考试进行中', themeBaseColor: '#2ecc71' });
    } else if (status === 'pending') {
    } else if (status === 'completed') {
      reminder.showColorfulAlert({ title: '考试已结束', themeBaseColor: '#ff3b30' });
    }
  }, 0);
});

// === UI 自动缩放逻辑 ===
let autoScaleAnimationId: number | null = null;
let currentAutoScale = 1;

const calculateAutoScale = () => {
  const w = window.innerWidth;
  if (w >= 1920) return 1.2;
  if (w >= 1440) return 1.0;
  if (w >= 1024) return 0.85;
  return 0.7;
};

const easeOutCubic = (t: number): number => {
  return 1 - Math.pow(1 - t, 3);
};

const setAutoRootScale = (scale: number) => {
  document.documentElement.style.setProperty('--ui-scale', String(scale));
  if (rootRef.value) {
    rootRef.value.style.setProperty('--ui-scale', String(scale));
  }
  console.log('Auto-scale set to:', scale);
};

const animateToAutoScale = (target: number) => {
  if (autoScaleAnimationId) {
    cancelAnimationFrame(autoScaleAnimationId);
  }

  const startScale = currentAutoScale;
  const startTime = performance.now();
  const duration = 400;

  const animate = (currentTime: number) => {
    const elapsed = currentTime - startTime;
    const progress = Math.min(elapsed / duration, 1);
    const easedProgress = easeOutCubic(progress);
    const scale = startScale + (target - startScale) * easedProgress;
    currentAutoScale = scale;
    setAutoRootScale(scale);

    if (progress < 1) {
      autoScaleAnimationId = requestAnimationFrame(animate);
      (window as any).autoScaleAnimationId = autoScaleAnimationId;
    } else {
      autoScaleAnimationId = null;
      (window as any).autoScaleAnimationId = null;
    }
  };

  autoScaleAnimationId = requestAnimationFrame(animate);
  (window as any).autoScaleAnimationId = autoScaleAnimationId;
};

const handleAutoScaleResize = () => {
  const targetScale = calculateAutoScale();
  animateToAutoScale(targetScale);
};

// 标题大小调整
const mainTitleRef = ref<HTMLElement>();
const subtitleRef = ref<HTMLElement>();

const adjustTitleSize = () => {
  if (!mainTitleRef.value || !subtitleRef.value) return;

  const container = mainTitleRef.value.parentElement;
  if (!container) return;

  setTimeout(() => {
    const containerWidth = container.clientWidth;

    let fontSize = 50;
    mainTitleRef.value!.style.fontSize = `${fontSize}px`;
    void mainTitleRef.value!.offsetHeight;

    let scrollWidth = mainTitleRef.value!.scrollWidth;

    while (scrollWidth > containerWidth && fontSize > 12) {
      fontSize -= 0.5;
      mainTitleRef.value!.style.fontSize = `${fontSize}px`;
      void mainTitleRef.value!.offsetHeight;
      scrollWidth = mainTitleRef.value!.scrollWidth;
    }

    fontSize = Math.max(12, fontSize - 5);
    mainTitleRef.value!.style.fontSize = `${fontSize}px`;

    const subtitleFontSize = fontSize * 0.4;
    subtitleRef.value!.style.fontSize = `${subtitleFontSize}px`;
  }, 10);
};

onMounted(() => {
  adjustTitleSize();
  window.addEventListener('resize', adjustTitleSize);

  currentAutoScale = calculateAutoScale();
  setAutoRootScale(currentAutoScale);
  window.addEventListener('resize', handleAutoScaleResize);

  const observer = new MutationObserver(() => {
    adjustTitleSize();
  });
  observer.observe(document.documentElement, {
    attributes: true,
    attributeFilter: ['style']
  });

  window.addEventListener('beforeunload', () => {
    observer.disconnect();
  });
});

onUnmounted(() => {
  window.removeEventListener('resize', adjustTitleSize);
  window.removeEventListener('resize', handleAutoScaleResize);

  if (autoScaleAnimationId) {
    cancelAnimationFrame(autoScaleAnimationId);
  }
});

watch(
  () => playerExamConfig?.value?.examName,
  () => adjustTitleSize()
);
watch(
  () => playerExamConfig?.value?.message,
  () => adjustTitleSize()
);

watch(
  () => props.roomNumber,
  (val) => {
    if (val != null) {
      localRoomNumber.value = val;
      tempRoomNumber.value = val;
    }
  }
);

// 监听考试变化，重置材料数量
watch(
  () => currentExam.value?.name,
  () => {
    materialPages.value = {};
    materialQuantities.value = {};
  }
);
</script>

<style scoped>
* {
  font-family: 'MiSans';
}

.exam-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: #02080d;
  --ui-scale: 1;
  --density-scale: 1;
  --large-clock-scale: 1;
}

.background-ellipse {
  position: absolute;
  top: 0;
  left: 50%;
  width: 100%;
  height: 45%;
  background: radial-gradient(
    50% 50% at 50% 50%,
    rgba(55, 88, 255, 0.3) 0%,
    rgba(70, 82, 255, 0) 100%
  );
  border-radius: 50%;
  transform: translateX(-50%) translateY(-50%);
  z-index: 0;
}

.content-wrapper {
  position: relative;
  z-index: 10;
  height: 100vh;
  display: flex;
  flex-direction: column;
  padding: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 1.5rem)
    calc(var(--ui-scale, 1) * var(--density-scale, 1) * 2rem)
    calc(var(--ui-scale, 1) * var(--density-scale, 1) * 6rem)
    calc(var(--ui-scale, 1) * var(--density-scale, 1) * 2rem);
  gap: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 1.5rem);
}

/* 顶部标题栏 */
.top-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  flex-shrink: 0;
}

.header-left {
  flex: 1;
  min-width: 0;
}

.header-right {
  flex-shrink: 0;
  margin-left: calc(var(--ui-scale, 1) * 2rem);
}

.main-title {
  color: #ffffff;
  font-weight: 700;
  line-height: 1.2;
  margin: 0 0 calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.5rem) 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  width: 100%;
}

.subtitle {
  color: rgba(255, 255, 255, 0.7);
  font-weight: 400;
  line-height: 1.4;
  margin: 0;
}

/* 中间时钟区域 */
.middle-section {
  flex-shrink: 0;
  display: flex;
  justify-content: center;
}

.clock-area {
  background: rgba(4, 14, 21, 0.8);
  border: 1px solid rgba(36, 47, 56, 0.6);
  border-radius: calc(var(--ui-scale, 1) * 20px);
  padding: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 2rem)
    calc(var(--ui-scale, 1) * var(--density-scale, 1) * 4rem);
  text-align: center;
  width: 66.67%;
}

.clock-label {
  color: rgba(255, 255, 255, 0.6);
  font-size: calc(var(--ui-scale, 1) * 1.5rem);
  margin-bottom: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.5rem);
}

.time-display {
  font-size: calc(var(--ui-scale, 1) * 5rem);
  line-height: 1;
  color: #fff;
  text-shadow: 0 calc(var(--ui-scale, 1) * 0.167rem) calc(var(--ui-scale, 1) * 1.458rem)
    rgba(255, 255, 255, 0.3);
  font-family: 'TCloudNumber', 'MiSans', monospace;
  font-weight: 600;
  margin-bottom: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.75rem);
}

.countdown-display {
  color: rgba(255, 255, 255, 0.85);
  font-size: calc(var(--ui-scale, 1) * 1.8rem);
  font-weight: 500;
}

/* 底部区域 */
.bottom-section {
  flex: 1;
  display: flex;
  gap: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 2rem);
  min-height: 0;
  overflow: hidden;
}

.bottom-left {
  width: 40%;
  min-width: 0;
  overflow: auto;
}

.bottom-right {
  width: 60%;
  min-width: 0;
  overflow: auto;
}

/* 当前考试信息卡片 */
.current-exam-card :deep(.card-content) {
  padding: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 1.5rem);
}

.current-exam-content {
  display: flex;
  flex-direction: column;
  gap: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 1rem);
}

.info-row {
  display: flex;
  align-items: center;
  gap: calc(var(--ui-scale, 1) * 0.5rem);
}

.info-label {
  color: rgba(255, 255, 255, 0.8);
  font-size: calc(var(--ui-scale, 1) * 1.4rem);
  font-weight: 500;
  white-space: nowrap;
  flex-shrink: 0;
}

.info-value {
  color: #fff;
  font-size: calc(var(--ui-scale, 1) * 1.4rem);
  font-weight: 500;
}

.status-pending {
  color: #888888;
}

.status-ongoing {
  color: #45a452;
}

.status-finished {
  color: #ff3b30;
}

.pre-exam-countdown {
  color: #f1c40f;
  font-weight: 700;
  animation: pre-exam-pulse 1s ease-in-out infinite;
}

@keyframes pre-exam-pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.6;
  }
}

/* 材料控制 */
.materials-section {
  display: flex;
  flex-direction: column;
  gap: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.75rem);
  margin-top: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.5rem);
}

.material-row {
  display: flex;
  align-items: center;
  gap: calc(var(--ui-scale, 1) * 0.5rem);
  flex-wrap: wrap;
}

.material-control {
  display: flex;
  align-items: center;
  gap: calc(var(--ui-scale, 1) * 0.35rem);
}

.material-text {
  color: rgba(255, 255, 255, 0.8);
  font-size: calc(var(--ui-scale, 1) * 1.2rem);
}

.number-box {
  display: flex;
  align-items: center;
  gap: calc(var(--ui-scale, 1) * 0.25rem);
  background: rgba(255, 255, 255, 0.1);
  border-radius: calc(var(--ui-scale, 1) * 6px);
  padding: calc(var(--ui-scale, 1) * 0.25rem) calc(var(--ui-scale, 1) * 0.5rem);
}

.num-btn {
  width: calc(var(--ui-scale, 1) * 1.5rem);
  height: calc(var(--ui-scale, 1) * 1.5rem);
  border: none;
  border-radius: calc(var(--ui-scale, 1) * 4px);
  background: rgba(255, 255, 255, 0.15);
  color: #fff;
  font-size: calc(var(--ui-scale, 1) * 1rem);
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s ease;
}

.num-btn:hover {
  background: rgba(255, 255, 255, 0.25);
}

.num-btn:active {
  background: rgba(255, 255, 255, 0.1);
}

.num-value {
  color: #fff;
  font-size: calc(var(--ui-scale, 1) * 1.2rem);
  font-weight: 600;
  min-width: calc(var(--ui-scale, 1) * 1.5rem);
  text-align: center;
}

/* 考试列表表格 */
.exam-list-card :deep(.card-content) {
  padding: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 1.5rem);
  height: 100%;
  display: flex;
  flex-direction: column;
}

.exam-table-header {
  display: grid;
  grid-template-columns: 1fr 2fr 1.5fr 1.5fr 1fr;
  gap: calc(var(--ui-scale, 1) * 1rem);
  padding-bottom: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.75rem);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  margin-bottom: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.75rem);
}

.exam-table-header span {
  color: rgba(255, 255, 255, 0.6);
  font-size: calc(var(--ui-scale, 1) * 1.2rem);
  font-weight: 500;
  text-align: center;
}

.exam-table-body {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.5rem);
}

.exam-table-row {
  display: grid;
  grid-template-columns: 1fr 2fr 1.5fr 1.5fr 1fr;
  gap: calc(var(--ui-scale, 1) * 1rem);
  padding: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 0.5rem) 0;
  align-items: center;
}

.exam-table-row span {
  color: rgba(255, 255, 255, 0.85);
  font-size: calc(var(--ui-scale, 1) * 1.2rem);
  text-align: center;
}

.exam-active {
  background: rgba(69, 164, 82, 0.15);
  border-radius: calc(var(--ui-scale, 1) * 8px);
}

.status-completed {
  color: #ff3b30;
}

.status-inProgress {
  color: #45a452;
  font-weight: 600;
}

.status-pending {
  color: #888888;
}

.empty-state {
  text-align: center;
  color: rgba(255, 255, 255, 0.5);
  font-size: calc(var(--ui-scale, 1) * 1.2rem);
  padding: calc(var(--ui-scale, 1) * var(--density-scale, 1) * 2rem) 0;
}

/* 弹窗样式 */
.keyboard-container {
  margin-top: 16px;
}

.virtual-keyboard {
  max-width: 340px;
  margin: 0 auto;
  background: transparent;
}

:deep(.numeric-keyboard-dark) {
  background: #1a1a1a !important;
  border-radius: 8px;
  padding: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

:deep(.numeric-keyboard-dark .hg-button) {
  background: #2d2d2d !important;
  color: #ffffff !important;
  border: 1px solid #404040 !important;
  border-radius: 6px !important;
  height: 50px !important;
  margin: 3px !important;
  font-size: 18px !important;
  font-weight: 500 !important;
  transition: all 0.2s ease !important;
  display: flex !important;
  align-items: center !important;
  justify-content: center !important;
}

:deep(.numeric-keyboard-dark .hg-button:hover) {
  background: #3d3d3d !important;
  border-color: #505050 !important;
  transform: translateY(-1px) !important;
}

:deep(.numeric-keyboard-dark .hg-button:active) {
  background: #1d1d1d !important;
  transform: translateY(0) !important;
}

:deep(.numeric-keyboard-dark .hg-button.hg-functionBtn) {
  background: #0052d9 !important;
  color: #ffffff !important;
  border-color: #0052d9 !important;
}

:deep(.numeric-keyboard-dark .hg-button.hg-functionBtn:hover) {
  background: #1668dc !important;
  border-color: #1668dc !important;
}

:deep(.numeric-keyboard-dark .hg-row) {
  display: flex !important;
  justify-content: center !important;
}

/* 覆盖层与动画 */
.overlay {
  position: fixed;
  inset: 0;
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
}

.fade-soft-enter-active,
.fade-soft-leave-active {
  transition:
    opacity 320ms ease,
    transform 320ms ease;
}
.fade-soft-enter-from,
.fade-soft-leave-to {
  opacity: 0;
  transform: scale(1.02);
}

.colorful-overlay {
  background: var(--colorful-bg, #ff3b30);
  background: color-mix(in srgb, var(--colorful-bg, #ff3b30) 85%, transparent);
  backdrop-filter: blur(2px);
}
.colorful-title {
  color: var(--colorful-text, #fff);
  font-size: calc(var(--ui-scale, 1) * 5rem);
  font-weight: 800;
  letter-spacing: 0.05em;
  text-shadow: var(--colorful-shadow, 0 6px 24px rgba(0, 0, 0, 0.35));
  text-align: center;
}

@media (dynamic-range: high) {
  .colorful-overlay.hdr-highlight .colorful-title {
    color: color(display-p3 1 1 1);
  }
}

.notice-overlay {
  backdrop-filter: blur(12px) saturate(1.1);
  background: rgba(0, 0, 0, 0.35);
  padding: 24px;
}
.notice-card {
  background: rgba(16, 22, 33, 0.9);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 16px;
  max-width: min(960px, 92vw);
  padding: 28px;
  color: #fff;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.45);
}
.notice-content :is(h1, h2, h3) {
  margin: 0 0 12px 0;
}
.notice-content h1 {
  font-size: 2rem;
}
.notice-content h2 {
  font-size: 1.5rem;
}
.notice-content p,
.notice-content br {
  line-height: 1.6;
}
.notice-content code {
  background: rgba(255, 255, 255, 0.08);
  padding: 0 6px;
  border-radius: 4px;
}
.notice-card :deep(.t-button) {
  margin-top: 18px;
}
</style>
