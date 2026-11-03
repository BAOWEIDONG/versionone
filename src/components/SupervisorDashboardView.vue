<script setup lang="ts">
import { computed, ref } from 'vue';
import { format, subDays } from 'date-fns';
import { useAppStore } from '../store/app';
import { Card } from './ui';
import { LogOut, ChevronDown, Download, Users, MessageSquare, CheckCircle2, ChevronRight, UserCheck } from 'lucide-vue-next';
import { Popup as VanPopup, showToast } from 'vant';

const store = useAppStore();

type RangeKey = 'today' | 'yesterday' | 'last7' | 'camp';
const range = ref<RangeKey>('yesterday');
const ranges: { key: RangeKey; label: string }[] = [
  { key: 'today', label: '今天' },
  { key: 'yesterday', label: '昨天' },
  { key: 'last7', label: '近7天' },
  { key: 'camp', label: '营期累计' },
];

// ─── 营期 ───
const showCampPicker = ref(false);
const activeCampId = computed(() => {
  if (store.selectedCampId) return store.selectedCampId;
  const opened = store.camps.filter((c) => c.status !== 'upcoming');
  return store.pickLatestCamp(opened)?.id || store.camps[0]?.id || null;
});
const camp = computed(() => store.camps.find((c) => c.id === activeCampId.value) || null);

const todayStr = format(new Date(), 'yyyy-MM-dd');
const yesterdayStr = format(subDays(new Date(), 1), 'yyyy-MM-dd');

// 时间范围 → [from, to]
const dateRange = computed(() => {
  if (range.value === 'today') return { from: todayStr, to: todayStr };
  if (range.value === 'yesterday') return { from: yesterdayStr, to: yesterdayStr };
  if (range.value === 'last7') return { from: format(subDays(new Date(), 6), 'yyyy-MM-dd'), to: todayStr };
  const from = camp.value?.startDate?.substring(0, 10) || '0000-01-01';
  const end = camp.value?.endDate?.substring(0, 10) || todayStr;
  const to = end > todayStr ? todayStr : end;
  return { from, to };
});

const inRange = (dateStr: string) => {
  const d = dateStr.substring(0, 10);
  return d >= dateRange.value.from && d <= dateRange.value.to;
};

// ─── 记录与批注 ───
interface Annotation { name: string; date: string }
interface Entry {
  id: string;
  type: 'diet' | 'weight' | 'exercise';
  typeLabel: string;
  studentId: string;
  studentName: string;
  date: string;
  summary: string;
  ownerRole: 'dietitian' | 'coach';
  annotations: Annotation[];
}

const annotationsOf = (rec: any, role: 'dietitian' | 'coach'): Annotation[] => {
  const list: Annotation[] = [];
  const arr = Array.isArray(rec.comments) ? rec.comments : [];
  arr.filter((c: any) => c.role === role).forEach((c: any) => list.push({ name: c.name, date: c.date }));
  if (list.length === 0) {
    if (role === 'dietitian' && rec.dietitianComment) {
      list.push({ name: rec.dietitianName || '营养师', date: rec.dietitianCommentDate || '' });
    }
    if (role === 'coach' && rec.coachComment) {
      list.push({ name: rec.coachName || '教练', date: rec.coachCommentDate || '' });
    }
  }
  return list;
};

const campStudents = computed(() => (activeCampId.value ? store.getStudentsByCamp(activeCampId.value) : []));
const studentName = (id: string) => campStudents.value.find((s) => s.id === id)?.name || id;

const entries = computed<Entry[]>(() => {
  if (!activeCampId.value) return [];
  const list: Entry[] = [];
  for (const r of store.getCampDietRecords(activeCampId.value)) {
    if (!inRange(r.date)) continue;
    list.push({
      id: r.id, type: 'diet', typeLabel: '饮食', studentId: r.studentId || '', studentName: studentName(r.studentId || ''),
      date: r.date, summary: r.description || '', ownerRole: 'dietitian', annotations: annotationsOf(r, 'dietitian'),
    });
  }
  for (const r of store.getCampWeightRecords(activeCampId.value)) {
    if (!inRange(r.date)) continue;
    list.push({
      id: r.id, type: 'weight', typeLabel: '体重', studentId: r.studentId || '', studentName: studentName(r.studentId || ''),
      date: r.date, summary: `${r.weight} kg`, ownerRole: 'dietitian', annotations: annotationsOf(r, 'dietitian'),
    });
  }
  for (const r of store.getCampExerciseRecords(activeCampId.value)) {
    if (!inRange(r.date)) continue;
    list.push({
      id: r.id, type: 'exercise', typeLabel: '运动', studentId: r.studentId || '', studentName: studentName(r.studentId || ''),
      date: r.date, summary: `${r.type} · ${r.duration} 分钟`, ownerRole: 'coach', annotations: annotationsOf(r, 'coach'),
    });
  }
  return list;
});

// ─── 概览（按记录条数） ───
const overview = computed(() => {
  const required = entries.value.length;
  const annotated = entries.value.filter((e) => e.annotations.length > 0).length;
  return { required, annotated, pending: required - annotated, rate: required ? Math.round((annotated / required) * 100) : 0 };
});

// ─── 团队工作量（按实际批注人统计） ───
const staffStats = computed(() => {
  const staff = store.accounts.filter((a) => a.active && (a.role === 'dietitian' || a.role === 'coach') && !a.supervisor);
  return staff.map((a) => {
    let count = 0;
    let hours = 0;
    let timed = 0;
    for (const e of entries.value) {
      for (const ann of e.annotations) {
        if (ann.name !== a.name) continue;
        count++;
        if (ann.date) {
          const diff = (new Date(ann.date).getTime() - new Date(e.date).getTime()) / 3600000;
          if (Number.isFinite(diff) && diff >= 0) { hours += diff; timed++; }
        }
      }
    }
    return {
      id: a.id, name: a.name, roleLabel: a.role === 'dietitian' ? '营养师' : '教练',
      count, avgHours: timed ? Math.round((hours / timed) * 10) / 10 : null,
    };
  }).sort((x, y) => y.count - x.count);
});

// ─── 学员批注覆盖 ───
const studentCoverage = computed(() => campStudents.value.map((s) => {
  const list = entries.value.filter((e) => e.studentId === s.id);
  const by = (t: Entry['type']) => {
    const sub = list.filter((e) => e.type === t);
    return { required: sub.length, annotated: sub.filter((e) => e.annotations.length > 0).length };
  };
  return { ...s, diet: by('diet'), weight: by('weight'), exercise: by('exercise'), total: list.length, unannotated: list.filter((e) => e.annotations.length === 0).length };
}).sort((a, b) => b.unannotated - a.unannotated));

const pendingEntries = computed(() => entries.value
  .filter((e) => e.annotations.length === 0)
  .sort((a, b) => a.date.localeCompare(b.date))
  .slice(0, 50));

const pct = (a: number, b: number) => (b > 0 ? Math.round((a / b) * 100) : 0);

const openStudent = (id: string) => {
  store.setSelectedStudentId(id);
  store.setCurrentView('dietitian-student-detail');
};

// ─── 导出 CSV ───
const exportCsv = () => {
  const lines: string[][] = [];
  lines.push([`督导看板`, camp.value?.name || '', `${dateRange.value.from} ~ ${dateRange.value.to}`]);
  lines.push([]);
  lines.push(['团队批注工作量（按实际批注人）']);
  lines.push(['姓名', '角色', '批注条数', '平均响应时长(小时)']);
  staffStats.value.forEach((s) => lines.push([s.name, s.roleLabel, String(s.count), s.avgHours === null ? '' : String(s.avgHours)]));
  lines.push([]);
  lines.push(['学员批注覆盖']);
  lines.push(['学员', '饮食已批注/应批注', '体重已批注/应批注', '运动已批注/应批注', '未批注条数']);
  studentCoverage.value.forEach((s) => lines.push([
    s.name,
    `${s.diet.annotated}/${s.diet.required}`,
    `${s.weight.annotated}/${s.weight.required}`,
    `${s.exercise.annotated}/${s.exercise.required}`,
    String(s.unannotated),
  ]));
  const csv = lines.map((row) => row.map((v) => `"${String(v).replace(/"/g, '""')}"`).join(',')).join('\n');
  const blob = new Blob(['﻿' + csv], { type: 'text/csv;charset=utf-8;' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = `督导看板_${camp.value?.name || ''}_${dateRange.value.from}_${dateRange.value.to}.csv`;
  a.click();
  URL.revokeObjectURL(a.href);
  showToast({ message: '已导出 CSV', duration: 1500 });
};
</script>

<template>
  <div class="flex min-h-[100dvh] flex-col pb-8 font-sans bg-[#F7F8FA]">
    <!-- 顶部 -->
    <div class="pt-[calc(env(safe-area-inset-top)+2rem)] px-5 pb-4 bg-gradient-to-b from-[#1F4E79]/10 to-[#F7F8FA]">
      <div class="flex items-center justify-between">
        <div>
          <h2 class="text-xl font-black text-gray-900">督导看板</h2>
          <p class="text-[11px] text-gray-500 mt-0.5">学员打卡 + 营养师/教练批注情况</p>
        </div>
        <div class="flex items-center gap-1">
          <button @click="exportCsv" class="p-2 rounded-full bg-white/70 text-[#1F4E79] active:bg-white"><Download class="w-5 h-5" /></button>
          <button @click="store.logout()" class="p-2 rounded-full bg-white/70 text-gray-500 active:bg-white"><LogOut class="w-5 h-5" /></button>
        </div>
      </div>
      <div class="flex items-center gap-2 mt-3">
        <button @click="showCampPicker = true" class="flex items-center gap-1.5 px-3 py-2 bg-white rounded-xl border border-gray-200 text-sm font-medium text-gray-700 shadow-sm">
          {{ camp?.name || '未选择营期' }}<ChevronDown class="w-4 h-4 text-gray-400" />
        </button>
        <span class="text-[11px] text-gray-400">{{ dateRange.from }} ~ {{ dateRange.to }}</span>
      </div>
      <div class="flex gap-1.5 mt-3 overflow-x-auto no-scrollbar">
        <button v-for="r in ranges" :key="r.key" @click="range = r.key"
          :class="['px-3 py-1.5 rounded-full text-xs font-bold shrink-0', range === r.key ? 'bg-[#1F4E79] text-white' : 'bg-white text-gray-600 border border-gray-200']">
          {{ r.label }}
        </button>
      </div>
    </div>

    <div class="px-5 space-y-4">
      <!-- 概览 -->
      <div class="grid grid-cols-2 gap-3">
        <Card class="p-4">
          <div class="text-2xl font-black text-gray-900">{{ overview.required }}</div>
          <div class="text-[11px] text-gray-500 mt-1">应批注记录</div>
        </Card>
        <Card class="p-4">
          <div class="text-2xl font-black text-[#07C160]">{{ overview.annotated }}</div>
          <div class="text-[11px] text-gray-500 mt-1">已批注 · {{ overview.rate }}%</div>
        </Card>
        <Card class="p-4">
          <div class="text-2xl font-black text-[#FF976A]">{{ overview.pending }}</div>
          <div class="text-[11px] text-gray-500 mt-1">待批注</div>
        </Card>
        <Card class="p-4">
          <div class="text-2xl font-black text-[#1F4E79]">{{ campStudents.length }}</div>
          <div class="text-[11px] text-gray-500 mt-1">在营学员</div>
        </Card>
      </div>

      <!-- 团队工作量 -->
      <Card class="p-4">
        <h3 class="text-sm font-bold text-gray-900 mb-3 flex items-center gap-2">
          <UserCheck class="w-4 h-4 text-[#1F4E79]" /> 团队批注工作量
          <span class="ml-auto text-[10px] text-gray-400 font-normal">按实际批注人统计</span>
        </h3>
        <div class="space-y-2">
          <div v-for="s in staffStats" :key="s.id" class="flex items-center gap-3 py-2 border-b border-gray-50 last:border-0">
            <div class="w-8 h-8 rounded-full bg-[#1F4E79]/10 text-[#1F4E79] flex items-center justify-center text-xs font-bold shrink-0">{{ s.name.substring(0, 1) }}</div>
            <div class="flex-1 min-w-0">
              <div class="text-xs font-bold text-gray-900">{{ s.name }} <span class="text-[9px] text-gray-400 font-normal">{{ s.roleLabel }}</span></div>
              <div class="text-[10px] text-gray-400">平均响应 {{ s.avgHours === null ? '--' : s.avgHours + ' 小时' }}</div>
            </div>
            <div class="text-sm font-black text-gray-900 shrink-0">{{ s.count }}<span class="text-[10px] text-gray-400 font-normal"> 条</span></div>
          </div>
          <div v-if="staffStats.length === 0" class="text-center text-xs text-gray-400 py-4">暂无营养师/教练账号</div>
        </div>
      </Card>

      <!-- 学员批注覆盖 -->
      <Card class="p-4">
        <h3 class="text-sm font-bold text-gray-900 mb-3 flex items-center gap-2">
          <Users class="w-4 h-4 text-[#FF976A]" /> 学员批注覆盖
          <span class="ml-auto text-[10px] text-gray-400 font-normal">点击看学员详情</span>
        </h3>
        <div class="space-y-2">
          <div v-for="s in studentCoverage" :key="s.id" class="p-3 rounded-xl bg-gray-50 active:bg-gray-100 cursor-pointer" @click="openStudent(s.id)">
            <div class="flex items-center gap-2">
              <span class="text-sm font-bold text-gray-900">{{ s.name }}</span>
              <span v-if="s.unannotated > 0" class="text-[9px] px-1.5 py-0.5 rounded bg-orange-50 text-[#FF976A] font-bold">{{ s.unannotated }} 条待批注</span>
              <span v-else class="text-[9px] px-1.5 py-0.5 rounded bg-[#07C160]/10 text-[#07C160] font-bold">已全部批注</span>
              <ChevronRight class="w-3.5 h-3.5 text-gray-300 ml-auto" />
            </div>
            <div class="grid grid-cols-3 gap-2 mt-2 text-[10px] text-gray-500">
              <div>饮食 <span class="font-bold text-gray-700">{{ s.diet.annotated }}/{{ s.diet.required }}</span> <span class="text-gray-400">{{ pct(s.diet.annotated, s.diet.required) }}%</span></div>
              <div>体重 <span class="font-bold text-gray-700">{{ s.weight.annotated }}/{{ s.weight.required }}</span> <span class="text-gray-400">{{ pct(s.weight.annotated, s.weight.required) }}%</span></div>
              <div>运动 <span class="font-bold text-gray-700">{{ s.exercise.annotated }}/{{ s.exercise.required }}</span> <span class="text-gray-400">{{ pct(s.exercise.annotated, s.exercise.required) }}%</span></div>
            </div>
          </div>
          <div v-if="studentCoverage.length === 0" class="text-center text-xs text-gray-400 py-4">当前营期暂无学员</div>
        </div>
      </Card>

      <!-- 待批注明细 -->
      <Card class="p-4">
        <h3 class="text-sm font-bold text-gray-900 mb-3 flex items-center gap-2">
          <MessageSquare class="w-4 h-4 text-[#FF976A]" /> 待批注记录
          <span class="ml-auto text-[10px] text-gray-400 font-normal">{{ overview.pending }} 条</span>
        </h3>
        <div class="space-y-2">
          <div v-for="e in pendingEntries" :key="e.id" class="flex items-start gap-2 py-2 border-b border-gray-50 last:border-0">
            <span :class="['text-[9px] px-1.5 py-0.5 rounded font-bold shrink-0 mt-0.5', e.type === 'diet' ? 'bg-orange-50 text-[#FF976A]' : e.type === 'weight' ? 'bg-[#1677FF]/10 text-[#1677FF]' : 'bg-[#07C160]/10 text-[#07C160]']">{{ e.typeLabel }}</span>
            <div class="flex-1 min-w-0">
              <div class="text-xs font-medium text-gray-900">{{ e.studentName }} · {{ e.summary }}</div>
              <div class="text-[10px] text-gray-400">{{ e.date.substring(0, 16) }} · 归属{{ e.ownerRole === 'dietitian' ? '营养师' : '教练' }}</div>
            </div>
          </div>
          <div v-if="pendingEntries.length === 0" class="text-center text-xs text-gray-400 py-4 flex items-center justify-center gap-1">
            <CheckCircle2 class="w-3.5 h-3.5 text-[#07C160]" /> 当前范围内没有待批注记录
          </div>
        </div>
      </Card>
    </div>

    <!-- 营期选择 -->
    <VanPopup v-model:show="showCampPicker" position="bottom" round>
      <div class="p-4">
        <h3 class="font-bold text-gray-900 text-base mb-3 text-center">选择营期</h3>
        <div class="space-y-2">
          <button v-for="c in store.camps" :key="c.id" @click="store.selectedCampId = c.id; showCampPicker = false"
            :class="['w-full flex items-center justify-between px-4 py-3 rounded-xl border', activeCampId === c.id ? 'border-[#1F4E79] bg-[#1F4E79]/5 text-[#1F4E79]' : 'border-gray-200 bg-white text-gray-700']">
            <span class="font-medium">{{ c.name }}</span>
            <span class="text-[10px] text-gray-400">{{ c.status === 'active' ? '进行中' : c.status === 'ended' ? '已结束' : '未开始' }}</span>
          </button>
        </div>
      </div>
    </VanPopup>
  </div>
</template>

<style scoped>
.no-scrollbar::-webkit-scrollbar { display: none; }
.no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
</style>
