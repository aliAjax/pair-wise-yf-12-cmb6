<script setup lang="ts">
import { computed, reactive, ref } from "vue";

type Field = {
  key: string;
  label: string;
  type?: "number" | "date" | "select";
  options?: readonly string[];
};

type RecordItem = {
  id: string;
  status: string;
  notes: string;
  createdAt: string;
  version: number;
  [key: string]: string | number;
};

type LedgerEntry = {
  id: string;
  stationId: string;
  stationName: string;
  shift: string;
  operator: string;
  beforeStock: number;
  afterStock: number;
  capacity: number;
  fromVersion: number;
  toVersion: number;
  note: string;
  createdAt: string;
};

const project = {
  "number": 21,
  "folder": "hxwl/frontend/hxwlfront-21",
  "framework": "vue",
  "title": "油站网点地图管理",
  "subtitle": "维护油站位置、营业状态和库存摘要。",
  "industry": "石油",
  "stack": [
    "Vue3",
    "Vite",
    "TypeScript",
    "Element Plus",
    "Leaflet"
  ],
  "storageKey": "hxwlfront-21-station-map",
  "formTitle": "新增油站",
  "primaryAction": "保存油站",
  "entityLabel": "油站",
  "statuses": [
    "营业中",
    "暂停营业",
    "库存紧张"
  ],
  "filters": [
    "全部区域",
    "东区",
    "西区",
    "机场线"
  ],
  "fields": [
    {
      "key": "station",
      "label": "油站名称"
    },
    {
      "key": "area",
      "label": "区域",
      "type": "select",
      "options": [
        "东区",
        "西区",
        "机场线"
      ]
    },
    {
      "key": "stock",
      "label": "库存摘要L",
      "type": "number"
    },
    {
      "key": "capacity",
      "label": "油罐容量L",
      "type": "number"
    },
    {
      "key": "manager",
      "label": "负责人"
    }
  ],
  "records": [
    {
      "station": "东区一站",
      "area": "东区",
      "stock": 36000,
      "capacity": 60000,
      "manager": "刘站长",
      "status": "营业中",
      "notes": "库存正常"
    },
    {
      "station": "机场快线站",
      "area": "机场线",
      "stock": 9000,
      "capacity": 50000,
      "manager": "王站长",
      "status": "库存紧张",
      "notes": "柴油待补"
    }
  ],
  "metricLabels": [
    "油站数",
    "营业中",
    "库存紧张"
  ]
} as const;

const fields = project.fields as readonly Field[];
const statuses = [...project.statuses];
const ledgerKey = `${project.storageKey}-ledger`;

// 警戒线为容量的两成：低于转库存紧张，补回线以上恢复营业中
const LOW_STOCK_RATIO = 0.2;
const DEFAULT_CAPACITY = 50000;
const shifts = ["早班", "中班", "晚班"];

function createBlank() {
  return Object.fromEntries(fields.map((field) => [field.key, field.type === "number" ? 0 : ""]));
}

function shiftOf(date: Date) {
  const hour = date.getHours();
  if (hour >= 6 && hour < 14) return shifts[0];
  if (hour >= 14 && hour < 22) return shifts[1];
  return shifts[2];
}

function isLow(stock: number, capacity: number) {
  return capacity > 0 && stock < capacity * LOW_STOCK_RATIO;
}

function normalize(record: RecordItem): RecordItem {
  return {
    ...record,
    stock: Number(record.stock) || 0,
    capacity: Number(record.capacity) || DEFAULT_CAPACITY,
    version: Number(record.version) || 1
  };
}

function loadRecords(): RecordItem[] {
  const raw = localStorage.getItem(project.storageKey);
  if (!raw) {
    return project.records.map((record, index) => ({
      ...record,
      id: `seed-${index + 1}`,
      version: 1,
      createdAt: new Date(Date.now() - index * 86400000).toISOString()
    })) as RecordItem[];
  }
  try {
    return (JSON.parse(raw) as RecordItem[]).map(normalize);
  } catch {
    return [];
  }
}

const records = ref<RecordItem[]>(loadRecords());

function loadLedger(): LedgerEntry[] {
  const raw = localStorage.getItem(ledgerKey);
  if (raw) {
    try {
      return JSON.parse(raw) as LedgerEntry[];
    } catch {
      return [];
    }
  }
  // 老数据升级不伪造历史；仅首次运行时为种子油站补期初流水
  if (localStorage.getItem(project.storageKey)) return [];
  return records.value.map((record) => ({
    id: crypto.randomUUID(),
    stationId: record.id,
    stationName: String(record.station),
    shift: shiftOf(new Date(record.createdAt)),
    operator: String(record.manager || "值班员"),
    beforeStock: 0,
    afterStock: Number(record.stock),
    capacity: Number(record.capacity),
    fromVersion: 0,
    toVersion: Number(record.version),
    note: "期初库存建档",
    createdAt: record.createdAt
  }));
}

const ledger = ref<LedgerEntry[]>(loadLedger());
const form = reactive<Record<string, string | number>>(createBlank());
const note = ref("");
const filter = ref(project.filters[0]);
const ledgerStation = ref("全部油站");
const ledgerShift = ref("全部班次");

const stockDialog = reactive({
  visible: false,
  stationId: "",
  stationName: "",
  baseVersion: 0,
  currentStock: 0,
  capacity: 0,
  shift: "",
  stock: 0,
  operator: "",
  note: "",
  conflict: ""
});

const filteredRecords = computed(() => {
  if (filter.value.startsWith("全部")) return records.value;
  return records.value.filter((record) => Object.values(record).includes(filter.value));
});

const filteredLedger = computed(() =>
  ledger.value.filter((entry) =>
    (ledgerStation.value === "全部油站" || entry.stationId === ledgerStation.value) &&
    (ledgerShift.value === "全部班次" || entry.shift === ledgerShift.value)
  )
);

const metrics = computed(() => [
  records.value.length,
  records.value.filter((record) => record.status === statuses[0]).length,
  records.value.filter((record) => record.status === statuses[2]).length
]);

const chartRows = computed(() => statuses.map((status) => ({
  status,
  value: records.value.filter((record) => record.status === status).length
})));

const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

function persist() {
  localStorage.setItem(project.storageKey, JSON.stringify(records.value));
}

function persistLedger() {
  localStorage.setItem(ledgerKey, JSON.stringify(ledger.value));
}

function nextStatus(status: string) {
  const index = statuses.indexOf(status);
  return statuses[(index + 1) % statuses.length];
}

function primaryText(record: RecordItem) {
  const first = fields[0];
  const second = fields[1];
  return [record[first.key], record[second.key]].filter(Boolean).join(" / ") || project.entityLabel;
}

function stockPercent(record: RecordItem) {
  const capacity = Number(record.capacity);
  if (!capacity) return 0;
  return Math.round((Number(record.stock) / capacity) * 100);
}

function formatTime(iso: string) {
  return new Date(iso).toLocaleString("zh-CN", { hour12: false });
}

// 暂停营业站只记流水，是否恢复营业由站长手动流转，不自动改状态
function autoStatus(record: RecordItem) {
  if (record.status === statuses[1]) return;
  record.status = isLow(Number(record.stock), Number(record.capacity)) ? statuses[2] : statuses[0];
}

function submit() {
  const record = {
    ...form,
    id: crypto.randomUUID(),
    status: statuses[0],
    notes: note.value || "暂无备注",
    createdAt: new Date().toISOString(),
    version: 1
  } as RecordItem;
  fields
    .filter((field) => field.type === "number")
    .forEach((field) => {
      record[field.key] = Number(record[field.key]) || 0;
    });
  autoStatus(record);
  records.value = [record, ...records.value];
  Object.assign(form, createBlank());
  note.value = "";
  persist();
}

function flow(record: RecordItem) {
  record.status = nextStatus(record.status);
  persist();
}

function remove(id: string) {
  records.value = records.value.filter((record) => record.id !== id);
  // 油站清掉时，它的班次流水一并清掉
  ledger.value = ledger.value.filter((entry) => entry.stationId !== id);
  persist();
  persistLedger();
}

function openStockDialog(record: RecordItem) {
  // 以本地存储里的最新记录为准，记下打开页面时的版本
  const latest = loadRecords().find((item) => item.id === record.id) || record;
  Object.assign(stockDialog, {
    visible: true,
    stationId: latest.id,
    stationName: String(latest.station),
    baseVersion: Number(latest.version),
    currentStock: Number(latest.stock),
    capacity: Number(latest.capacity),
    shift: shiftOf(new Date()),
    stock: Number(latest.stock),
    operator: "",
    note: "",
    conflict: ""
  });
}

function closeStockDialog() {
  stockDialog.visible = false;
}

function submitStock() {
  const latest = loadRecords();
  const target = latest.find((item) => item.id === stockDialog.stationId);
  if (!target) {
    records.value = latest;
    stockDialog.conflict = "该油站已被删除，本次库存未写入。";
    return;
  }
  if (Number(target.version) !== stockDialog.baseVersion) {
    // 期间有人先保存：这次不写入，提示按最新库存重新核对
    records.value = latest;
    stockDialog.conflict = `提交未写入：库存已被他人更新到 v${target.version}（你按 v${stockDialog.baseVersion} 核对），请按最新库存 ${target.stock}L 重新核对后再提交。`;
    stockDialog.baseVersion = Number(target.version);
    stockDialog.currentStock = Number(target.stock);
    stockDialog.stock = Number(target.stock);
    return;
  }
  const before = Number(target.stock);
  target.stock = Math.max(0, Number(stockDialog.stock) || 0);
  target.version = stockDialog.baseVersion + 1;
  autoStatus(target);
  records.value = latest;
  ledger.value = [
    {
      id: crypto.randomUUID(),
      stationId: target.id,
      stationName: String(target.station),
      shift: shiftOf(new Date()),
      operator: stockDialog.operator || "未署名值班员",
      beforeStock: before,
      afterStock: Number(target.stock),
      capacity: Number(target.capacity),
      fromVersion: stockDialog.baseVersion,
      toVersion: Number(target.version),
      note: stockDialog.note || "班次库存交接",
      createdAt: new Date().toISOString()
    },
    ...ledger.value
  ];
  persist();
  persistLedger();
  closeStockDialog();
}
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">{{ project.industry }}行业前端最小闭环</p>
          <h1>{{ project.title }}</h1>
          <p class="subtitle">{{ project.subtitle }}</p>
        </div>
        <div class="stack">
          <span v-for="item in project.stack" :key="item" class="tag">{{ item }}</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="(label, index) in project.metricLabels" :key="label" class="metric">
          <span>{{ label }}</span>
          <strong>{{ metrics[index] }}</strong>
        </article>
      </section>

      <section class="workspace">
        <form class="panel" @submit.prevent="submit">
          <h2>{{ project.formTitle }}</h2>
          <div class="form-grid">
            <label v-for="field in fields" :key="field.key">
              {{ field.label }}
              <select v-if="field.type === 'select'" v-model="form[field.key]" required>
                <option value="">请选择</option>
                <option v-for="option in field.options" :key="option">{{ option }}</option>
              </select>
              <input v-else v-model="form[field.key]" :type="field.type || 'text'" required />
            </label>
            <label>
              备注
              <textarea v-model="note" placeholder="填写处理说明或现场备注" />
            </label>
            <button type="submit">{{ project.primaryAction }}</button>
          </div>
        </form>

        <section class="list-panel">
          <div class="toolbar">
            <h2>{{ project.entityLabel }}列表</h2>
            <select v-model="filter">
              <option v-for="item in project.filters" :key="item">{{ item }}</option>
            </select>
          </div>

          <div class="record-grid">
            <div v-if="filteredRecords.length === 0" class="empty">暂无匹配数据</div>
            <article v-for="record in filteredRecords" :key="record.id" class="record">
              <div class="record-head">
                <p class="record-title">{{ primaryText(record) }}</p>
                <span class="version">v{{ record.version }}</span>
                <span class="status">{{ record.status }}</span>
              </div>
              <div class="details">
                <span v-for="field in fields" :key="field.key">{{ field.label }}: {{ record[field.key] }}</span>
                <span>库存占比: {{ stockPercent(record) }}%（警戒线 20%）</span>
              </div>
              <p class="note">{{ record.notes }}</p>
              <div class="actions">
                <button type="button" @click="openStockDialog(record)">登记库存</button>
                <button class="secondary" type="button" @click="flow(record)">流转状态</button>
                <button class="secondary" type="button" @click="navigator.clipboard?.writeText(primaryText(record))">复制摘要</button>
                <button class="danger" type="button" @click="remove(record.id)">删除</button>
              </div>
            </article>
          </div>

          <div class="mini-chart">
            <div v-for="row in chartRows" :key="row.status" class="bar">
              <span>{{ row.status }}</span>
              <div class="bar-track"><div class="bar-fill" :style="{ width: `${(row.value / maxChart) * 100}%` }" /></div>
              <strong>{{ row.value }}</strong>
            </div>
          </div>
        </section>
      </section>

      <section class="ledger-panel">
        <div class="toolbar">
          <h2>班次库存流水</h2>
          <div class="ledger-filters">
            <select v-model="ledgerStation">
              <option>全部油站</option>
              <option v-for="record in records" :key="record.id" :value="record.id">{{ record.station }}</option>
            </select>
            <select v-model="ledgerShift">
              <option>全部班次</option>
              <option v-for="shift in shifts" :key="shift">{{ shift }}</option>
            </select>
          </div>
        </div>

        <div class="ledger-list">
          <div v-if="filteredLedger.length === 0" class="empty">暂无流水记录</div>
          <article v-for="entry in filteredLedger" :key="entry.id" class="ledger-item">
            <div class="ledger-main">
              <strong>{{ entry.stationName }}</strong>
              <span class="tag">{{ entry.shift }}</span>
              <span v-if="isLow(entry.afterStock, entry.capacity)" class="low-tag">低于警戒线</span>
            </div>
            <div class="ledger-sub">
              {{ formatTime(entry.createdAt) }} · {{ entry.operator }} · 库存 {{ entry.beforeStock }}L → {{ entry.afterStock }}L · 版本 v{{ entry.fromVersion }} → v{{ entry.toVersion }}
            </div>
            <p class="ledger-note">{{ entry.note }}</p>
          </article>
        </div>
      </section>
    </div>

    <div v-if="stockDialog.visible" class="modal-mask" @click.self="closeStockDialog">
      <form class="modal" @submit.prevent="submitStock">
        <h3>登记库存 · {{ stockDialog.stationName }}</h3>
        <p class="modal-meta">
          当前库存 {{ stockDialog.currentStock }}L / 容量 {{ stockDialog.capacity }}L · 版本 v{{ stockDialog.baseVersion }} · {{ stockDialog.shift }}
        </p>
        <label>
          新库存（L）
          <input v-model.number="stockDialog.stock" type="number" min="0" required />
        </label>
        <label>
          值班员
          <input v-model="stockDialog.operator" placeholder="填写值班员姓名" />
        </label>
        <label>
          交接备注
          <textarea v-model="stockDialog.note" placeholder="填写班次交接说明" />
        </label>
        <p v-if="stockDialog.conflict" class="conflict">{{ stockDialog.conflict }}</p>
        <div class="actions">
          <button type="submit">提交库存</button>
          <button class="secondary" type="button" @click="closeStockDialog">取消</button>
        </div>
      </form>
    </div>
  </main>
</template>
