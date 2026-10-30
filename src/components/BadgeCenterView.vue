<script setup lang="ts">
import { computed, onMounted } from 'vue';
import { useAppStore } from '../store/app';
import { NavBar, Card } from './ui';
import { Award, Medal, Trophy, Sparkles, Info, Lock } from 'lucide-vue-next';
import type { BadgeCadence } from '../types';
import { generateStudentReport } from '../lib/campReport';
import { computeWeightMilestones } from '../lib/campActivities';
import { MOCK_STUDENT_METRIC_VALUES } from '../mock/data';

const store = useAppStore();

const availableCamps = computed(() => (store.user ? store.getStudentCamps(store.user.id) : []));
const campId = computed(() => {
  if (store.selectedCampId && availableCamps.value.some((c) => c.id === store.selectedCampId)) return store.selectedCampId;
  return store.pickLatestCamp(availableCamps.value)?.id || availableCamps.value[0]?.id || null;
});
const camp = computed(() => availableCamps.value.find((c) => c.id === campId.value) || null);
const uid = computed(() => store.user?.id || '');

// 活动徽章（周期达成，可重复）
const configs = computed(() => (campId.value ? store.getBadgeConfigs(campId.value) : []));
const activityBadges = computed(() => (uid.value && campId.value ? store.getStudentBadges(uid.value, campId.value) : []));

// 本营期打卡记录
const campDiet = computed(() => (campId.value ? store.getCampDietRecords(campId.value).filter((r) => r.studentId === uid.value) : []));
const campEx = computed(() => (campId.value ? store.getCampExerciseRecords(campId.value).filter((r) => r.studentId === uid.value) : []));
const campWt = computed(() => (campId.value ? store.getCampWeightRecords(campId.value).filter((r) => r.studentId === uid.value) : []));
const campDays = computed(() => {
  if (camp.value?.startDate && camp.value?.endDate) {
    const d = Math.round((new Date(camp.value.endDate).getTime() - new Date(camp.value.startDate).getTime()) / 86400000);
    return d > 0 ? d : 28;
  }
  return store.user ? store.getCampDays(store.user.id) : 28;
});

// 一次性徽章：里程碑（减重 %）+ 成就（连续打卡 / 运动行为等）
const report = computed(() => {
  if (!store.user || !campId.value) return null;
  return generateStudentReport(
    { id: store.user.id, name: store.user.name, gender: store.user.gender },
    store.metricConfigs,
    MOCK_STUDENT_METRIC_VALUES[store.user.id] || {},
    campDiet.value,
    campEx.value,
    campWt.value,
    campDays.value,
  );
});

const milestoneBadges = computed(() => {
  const weights = campWt.value.slice().sort((a, b) => a.date.localeCompare(b.date));
  const start = weights.length ? weights[0].weight : null;
  return computeWeightMilestones(weights, start).map((m) => ({
    id: m.id,
    name: `减重 ${Math.round(m.threshold * 100)}% 里程碑`,
    description: m.achieved ? '已达成' : `当前进度 ${Math.round(m.progress * 100)}%`,
    icon: '🎯',
    earned: m.achieved,
  }));
});

const behaviorBadges = computed(() => (report.value?.achievements || []).map((a) => ({
  id: a.id,
  name: a.title,
  description: a.description,
  icon: a.icon,
  earned: a.unlocked,
})));

const earnedCount = computed(() => milestoneBadges.value.filter((b) => b.earned).length + behaviorBadges.value.filter((b) => b.earned).length + activityBadges.value.length);
const activityCounts = computed(() => {
  const by = (c: BadgeCadence) => activityBadges.value.filter((b) => b.cadence === c).length;
  return { day: by('day'), week: by('week'), month: by('month') };
});
const configStats = computed(() => configs.value.map((cfg) => {
  const list = activityBadges.value.filter((b) => b.badgeId === cfg.id);
  return { ...cfg, earned: list.length, latest: list[0]?.earnedAt || '' };
}));

const cadenceLabel = (c: BadgeCadence) => (c === 'day' ? '每日' : c === 'week' ? '每周' : '每月');

onMounted(() => {
  if (campId.value) store.syncBadges(campId.value);
});
</script>

<template>
  <div class="flex min-h-full flex-col bg-[#F4F6F8] pb-10 font-sans">
    <NavBar title="我的徽章" :on-back="store.goBack" />

    <div class="px-5 pt-4 space-y-4">
      <!-- 顶部总览 -->
      <div class="rounded-2xl p-5 text-white bg-gradient-to-br from-[#07C160] to-[#04a551] shadow-lg">
        <div class="flex items-center gap-2 text-xs opacity-90">
          <Medal class="w-4 h-4" /> {{ camp?.name || '健康训练营' }} · 徽章收集
        </div>
        <div class="mt-3 flex items-end gap-1">
          <span class="text-4xl font-black">{{ earnedCount }}</span>
          <span class="text-sm mb-1 opacity-90">枚徽章</span>
        </div>
        <div class="mt-3 grid grid-cols-3 gap-2 text-center">
          <div class="bg-white/15 rounded-xl py-2">
            <div class="text-lg font-bold">{{ milestoneBadges.filter(b => b.earned).length }}</div>
            <div class="text-[10px] opacity-85">里程碑</div>
          </div>
          <div class="bg-white/15 rounded-xl py-2">
            <div class="text-lg font-bold">{{ behaviorBadges.filter(b => b.earned).length }}</div>
            <div class="text-[10px] opacity-85">成就</div>
          </div>
          <div class="bg-white/15 rounded-xl py-2">
            <div class="text-lg font-bold">{{ activityBadges.length }}</div>
            <div class="text-[10px] opacity-85">活动</div>
          </div>
        </div>
      </div>

      <!-- 里程碑徽章 -->
      <Card>
        <h3 class="text-sm font-bold text-gray-900 mb-3 flex items-center gap-2">
          <Trophy class="w-4 h-4 text-[#FF976A]" /> 里程碑徽章
        </h3>
        <div class="grid grid-cols-2 gap-2">
          <div v-for="b in milestoneBadges" :key="b.id" :class="['flex items-center gap-2 p-3 rounded-xl', b.earned ? 'bg-orange-50' : 'bg-gray-50']">
            <div :class="['w-10 h-10 rounded-full flex items-center justify-center text-xl shrink-0', b.earned ? 'bg-white shadow-sm' : 'bg-gray-200 grayscale opacity-60']">
              <span v-if="b.earned">{{ b.icon }}</span>
              <Lock v-else class="w-4 h-4 text-gray-400" />
            </div>
            <div class="min-w-0">
              <div :class="['text-xs font-bold truncate', b.earned ? 'text-gray-900' : 'text-gray-400']">{{ b.name }}</div>
              <div class="text-[10px] text-gray-400 truncate">{{ b.description }}</div>
            </div>
          </div>
        </div>
      </Card>

      <!-- 成就徽章（一次性） -->
      <Card>
        <h3 class="text-sm font-bold text-gray-900 mb-1 flex items-center gap-2">
          <Award class="w-4 h-4 text-[#1677FF]" /> 成就徽章
          <span class="ml-auto text-[11px] text-gray-400 font-normal">{{ behaviorBadges.filter(b => b.earned).length }}/{{ behaviorBadges.length }}</span>
        </h3>
        <p class="text-[10px] text-gray-400 mb-3">一次性达成，不重复获得</p>
        <div class="grid grid-cols-3 gap-2">
          <div v-for="b in behaviorBadges" :key="b.id" class="flex flex-col items-center text-center p-2 rounded-xl bg-gray-50">
            <div :class="['w-10 h-10 rounded-full flex items-center justify-center text-xl', b.earned ? 'bg-white shadow-sm' : 'bg-gray-200 grayscale opacity-50']">{{ b.icon }}</div>
            <div :class="['text-[11px] font-bold mt-1 leading-tight', b.earned ? 'text-gray-900' : 'text-gray-400']">{{ b.name }}</div>
            <div class="text-[9px] text-gray-400 mt-0.5 line-clamp-2">{{ b.description }}</div>
          </div>
        </div>
      </Card>

      <!-- 活动徽章（周期，可重复） -->
      <Card>
        <h3 class="text-sm font-bold text-gray-900 mb-3 flex items-center gap-2">
          <Sparkles class="w-4 h-4 text-[#FF976A]" /> 活动徽章
          <span class="ml-auto text-[11px] text-gray-400 font-normal">日 {{ activityCounts.day }} · 周 {{ activityCounts.week }} · 月 {{ activityCounts.month }}</span>
        </h3>
        <div class="space-y-3">
          <div v-for="c in configStats" :key="c.id" class="flex items-center gap-3 p-3 rounded-xl bg-gray-50">
            <div class="w-11 h-11 rounded-full bg-white flex items-center justify-center text-2xl shadow-sm shrink-0">{{ c.icon }}</div>
            <div class="flex-1 min-w-0">
              <div class="text-sm font-bold text-gray-900 truncate">{{ c.name }}</div>
              <div class="text-[11px] text-gray-500 truncate">{{ c.description || '达成即可获得' }}</div>
            </div>
            <div class="text-right shrink-0">
              <div class="text-sm font-black text-[#07C160]">{{ c.earned }}<span class="text-[10px] text-gray-400 font-normal"> 枚</span></div>
              <div class="text-[10px] text-gray-400">{{ c.latest || '未获得' }}</div>
            </div>
          </div>
          <div v-if="configStats.length === 0" class="text-center text-xs text-gray-400 py-4">当前营期暂无活动徽章</div>
        </div>
      </Card>

      <!-- 活动徽章明细 -->
      <Card v-if="activityBadges.length > 0">
        <h3 class="text-sm font-bold text-gray-900 mb-3">活动徽章明细</h3>
        <div class="space-y-2">
          <div v-for="b in activityBadges" :key="b.id" class="flex items-center gap-3 py-2 border-b border-gray-50 last:border-0">
            <div class="w-9 h-9 rounded-full bg-[#07C160]/10 flex items-center justify-center text-xl shrink-0">{{ b.icon }}</div>
            <div class="flex-1 min-w-0">
              <div class="text-xs font-bold text-gray-900">{{ b.badgeName }}</div>
              <div class="text-[10px] text-gray-400">{{ cadenceLabel(b.cadence) }} · 周期 {{ b.periodKey }} · 达成 {{ b.value }} 天</div>
            </div>
            <div class="text-[10px] text-gray-500 shrink-0">{{ b.earnedAt }}</div>
          </div>
        </div>
      </Card>

      <div class="flex items-start gap-2 text-[11px] text-gray-400 px-1">
        <Info class="w-3.5 h-3.5 shrink-0 mt-0.5" />
        <p>系统仅记录活动达成与徽章；实物奖励由你所在公司根据徽章清单发放。</p>
      </div>
    </div>
  </div>
</template>
