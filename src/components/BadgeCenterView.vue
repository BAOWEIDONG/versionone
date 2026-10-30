<script setup lang="ts">
import { computed, onMounted } from 'vue';
import { useAppStore } from '../store/app';
import { NavBar, Card } from './ui';
import { Award, Medal, Trophy, Sparkles, Info } from 'lucide-vue-next';
import type { BadgeCadence } from '../types';

const store = useAppStore();

const availableCamps = computed(() => (store.user ? store.getStudentCamps(store.user.id) : []));
const campId = computed(() => {
  if (store.selectedCampId && availableCamps.value.some((c) => c.id === store.selectedCampId)) return store.selectedCampId;
  return store.pickLatestCamp(availableCamps.value)?.id || availableCamps.value[0]?.id || null;
});
const camp = computed(() => availableCamps.value.find((c) => c.id === campId.value) || null);

const configs = computed(() => (campId.value ? store.getBadgeConfigs(campId.value) : []));
const myBadges = computed(() => (store.user && campId.value ? store.getStudentBadges(store.user.id, campId.value) : []));

const counts = computed(() => {
  const by = (c: BadgeCadence) => myBadges.value.filter((b) => b.cadence === c).length;
  return { day: by('day'), week: by('week'), month: by('month'), total: myBadges.value.length };
});

const configStats = computed(() => configs.value.map((cfg) => {
  const list = myBadges.value.filter((b) => b.badgeId === cfg.id);
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
          <Medal class="w-4 h-4" /> {{ camp?.name || '健康训练营' }} · 徽章记录
        </div>
        <div class="mt-3 flex items-end gap-1">
          <span class="text-4xl font-black">{{ counts.total }}</span>
          <span class="text-sm mb-1 opacity-90">枚徽章</span>
        </div>
        <div class="mt-3 grid grid-cols-3 gap-2 text-center">
          <div class="bg-white/15 rounded-xl py-2">
            <div class="text-lg font-bold">{{ counts.day }}</div>
            <div class="text-[10px] opacity-85">每日</div>
          </div>
          <div class="bg-white/15 rounded-xl py-2">
            <div class="text-lg font-bold">{{ counts.week }}</div>
            <div class="text-[10px] opacity-85">每周</div>
          </div>
          <div class="bg-white/15 rounded-xl py-2">
            <div class="text-lg font-bold">{{ counts.month }}</div>
            <div class="text-[10px] opacity-85">每月</div>
          </div>
        </div>
      </div>

      <!-- 活动进度（计数形式） -->
      <Card>
        <h3 class="text-sm font-bold text-gray-900 mb-3 flex items-center gap-2">
          <Sparkles class="w-4 h-4 text-[#FF976A]" /> 趣味活动
        </h3>
        <div class="space-y-3">
          <div v-for="c in configStats" :key="c.id" class="flex items-center gap-3 p-3 rounded-xl bg-gray-50">
            <div class="w-11 h-11 rounded-full bg-white flex items-center justify-center text-2xl shadow-sm shrink-0">{{ c.icon }}</div>
            <div class="flex-1 min-w-0">
              <div class="text-sm font-bold text-gray-900 truncate">{{ c.name }}</div>
              <div class="text-[11px] text-gray-500 truncate">{{ c.description || (c.cadence === 'day' ? '每日' : c.cadence === 'week' ? '每周' : '每月') + '达成即可获得' }}</div>
            </div>
            <div class="text-right shrink-0">
              <div class="text-sm font-black text-[#07C160]">{{ c.earned }}<span class="text-[10px] text-gray-400 font-normal"> 枚</span></div>
              <div class="text-[10px] text-gray-400">{{ c.latest || '未获得' }}</div>
            </div>
          </div>
          <div v-if="configStats.length === 0" class="text-center text-xs text-gray-400 py-4">当前营期暂无徽章活动</div>
        </div>
      </Card>

      <!-- 徽章墙 -->
      <Card>
        <h3 class="text-sm font-bold text-gray-900 mb-3 flex items-center gap-2">
          <Trophy class="w-4 h-4 text-[#FF976A]" /> 徽章明细
        </h3>
        <div v-if="myBadges.length > 0" class="space-y-2">
          <div v-for="b in myBadges" :key="b.id" class="flex items-center gap-3 py-2 border-b border-gray-50 last:border-0">
            <div class="w-9 h-9 rounded-full bg-[#07C160]/10 flex items-center justify-center text-xl shrink-0">{{ b.icon }}</div>
            <div class="flex-1 min-w-0">
              <div class="text-xs font-bold text-gray-900">{{ b.badgeName }}</div>
              <div class="text-[10px] text-gray-400">{{ cadenceLabel(b.cadence) }} · 周期 {{ b.periodKey }} · 达成 {{ b.value }} 天</div>
            </div>
            <div class="text-[10px] text-gray-500 shrink-0">{{ b.earnedAt }}</div>
          </div>
        </div>
        <div v-else class="py-6 text-center">
          <Award class="w-8 h-8 text-gray-300 mx-auto mb-2" />
          <div class="text-xs text-gray-400">完成打卡即可获得第一枚徽章</div>
        </div>
      </Card>

      <div class="flex items-start gap-2 text-[11px] text-gray-400 px-1">
        <Info class="w-3.5 h-3.5 shrink-0 mt-0.5" />
        <p>系统仅记录活动达成与徽章；实物奖励由你所在公司根据徽章清单发放。</p>
      </div>
    </div>
  </div>
</template>
