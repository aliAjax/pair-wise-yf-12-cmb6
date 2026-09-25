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
  station: string;
  shift: string;
  operator: string;
  beforeStock: number;
  afterStock: number;
  note: string;
  version: number;
  createdAt: string;
};

const project = {
  "number": 21,
  "folder": "hxwl/frontend/hxwlfront-21",
  "framework": "vue",
  "title": "油站网点地图管理",
  "subtitle": "维护油站位置、营业状态，并按班次记录库存流水。",
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
      "capacity": 50000,
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
const shifts = ["早班", "中班", "晚班"];

function currentShift() {
  const hour = new Date().getHours();
  if (hour >= 6 && hour < 14) return shifts[0];
  if (hour >= 14 && hour < 22) return shifts[1];
  return shifts[2];
}

// 警戒线为容量两成：低于则转库存紧张，补回警戒线以上恢复营业中；
// 暂停营业站不自动流转，只记录流水，是否恢复由站长手动处理。
function resolveStatus(status: string, stock: number, capacity: number) {
  if (status === "暂停营业") return status;
  return stock < capacity * 0.2 ? "库存紧张" : "营业中";
}

function createBlank() {
  return Object.fromEntries(fields.map((field) => [field.key, field.type === "number" ? 0 : ""]));
}

function normalize(list: RecordItem[]): RecordItem[] {
  return list.map((record) => ({
    ...record,
    capacity: Number(record.capacity) > 0 ? Number(record.capacity) : 50000,
    version: Number(record.version) > 0 ? Number(record.version) : 1
  }));
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
    return normalize(JSON.parse(raw) as RecordItem[]);
  } catch {
    return [];
  }
}

function loadLedger(): LedgerEntry[] {
  const raw = localStorage.getItem(ledgerKey);
  if (raw) {
    try {
      return JSON.parse(raw) as LedgerEntry[];
    } catch {
      return [];
    }
  }
  if (!localStorage.getItem(project.storageKey)) {
    return project.records.map((record, index) => ({
      id: `seed-ledger-${index + 1}`,
      stationId: `seed-${index + 1}`,
      station: record.station,
      shift: currentShift(),
      operator: record.manager,
      beforeStock: 0,
      afterStock: record.stock,
      note: "建站初始库存",
      version: 1,
      createdAt: new Date(Date.now() - index * 86400000).toISOString()
    }));
  }
  return [];
}

const records = ref<RecordItem[]>(loadRecords());
const ledger = ref<LedgerEntry[]>(loadLedger());
const form = reactive<Record<string, string | number>>(createBlank());
const note = ref("");
const filter = ref(project.filters[0]);

const editingId = ref<string | null>(null);
const editVersion = ref(1);
const editSnapshot = reactive({ stock: 0, capacity: 0, status: "" });
const editForm = reactive({ stock: 0 as number | string, shift: currentShift(), operator: "", note: "" });
const editError = ref("");

const ledgerShift = ref("全部班次");
const ledgerStation = ref("全部油站");

// 其他标签页保存后同步本地视图，版本校验以 localStorage 最新值为准
window.addEventListener("storage", (event) => {
  if (event.key === project.storageKey) records.value = loadRecords();
  if (event.key === ledgerKey) ledger.value = loadLedger();
});

const filteredRecords = computed(() => {
  if (filter.value.startsWith("全部")) return records.value;
  return records.value.filter((record) => Object.values(record).includes(filter.value));
});

const filteredLedger = computed(() =>
  ledger.value.filter((entry) =>
    (ledgerShift.value === "全部班次" || entry.shift === ledgerShift.value) &&
    (ledgerStation.value === "全部油站" || entry.stationId === ledgerStation.value)
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

function appendLedger(entry: Omit<LedgerEntry, "id" | "createdAt">) {
  ledger.value = [
    { ...entry, id: crypto.randomUUID(), createdAt: new Date().toISOString() },
    ...ledger.value
  ];
  persistLedger();
}

function submit() {
  const stock = Number(form.stock) || 0;
  const capacity = Number(form.capacity) || 0;
  const record = {
    ...form,
    stock,
    capacity,
    id: crypto.randomUUID(),
    status: resolveStatus("营业中", stock, capacity),
    version: 1,
    notes: note.value || "暂无备注",
    createdAt: new Date().toISOString()
  } as RecordItem;
  records.value = [record, ...records.value];
  appendLedger({
    stationId: record.id,
    station: String(record.station),
    shift: currentShift(),
    operator: String(form.manager) || "值班员",
    beforeStock: 0,
    afterStock: stock,
    note: "建站初始库存",
    version: 1
  });
  Object.assign(form, createBlank());
  note.value = "";
  persist();
}

function syncEditor(record: RecordItem) {
  editVersion.value = Number(record.version) || 1;
  editSnapshot.stock = Number(record.stock) || 0;
  editSnapshot.capacity = Number(record.capacity) || 0;
  editSnapshot.status = record.status;
}

function openStockEditor(record: RecordItem) {
  editingId.value = record.id;
  syncEditor(record);
  editForm.stock = Number(record.stock) || 0;
  editForm.shift = currentShift();
  editForm.operator = "";
  editForm.note = "";
  editError.value = "";
}

function submitStock() {
  const id = editingId.value;
  if (!id) return;
  // 以最新持久化数据校验版本：若打开页面后有人先保存，则本次不写入
  const latest = loadRecords();
  const current = latest.find((record) => record.id === id);
  if (!current) {
    records.value = latest;
    editingId.value = null;
    editError.value = "";
    return;
  }
  const currentVersion = Number(current.version) || 1;
  if (currentVersion !== editVersion.value) {
    records.value = latest;
    syncEditor(current);
    editError.value = `库存已被他人先保存（当前版本 v${currentVersion}），本次未写入，请按最新库存重新核对后再提交。`;
    return;
  }
  const stock = Number(editForm.stock);
  if (editForm.stock === "" || !Number.isFinite(stock) || stock < 0) {
    editError.value = "库存需为不小于 0 的数字。";
    return;
  }
  const capacity = Number(current.capacity) || 0;
  const updated: RecordItem = {
    ...current,
    stock,
    status: resolveStatus(current.status, stock, capacity),
    version: currentVersion + 1
  };
  records.value = latest.map((record) => (record.id === id ? updated : record));
  persist();
  appendLedger({
    stationId: id,
    station: String(current.station),
    shift: editForm.shift,
    operator: editForm.operator || "值班员",
    beforeStock: Number(current.stock) || 0,
    afterStock: stock,
    note: editForm.note || (current.status === "暂停营业" ? "暂停营业期间登记，仅记录流水" : "班次库存登记"),
    version: updated.version
  });
  editingId.value = null;
  editError.value = "";
}

function flow(record: RecordItem) {
  if (record.status === "暂停营业") {
    // 站长手动恢复：按当前库存与警戒线落到营业中/库存紧张
    record.status = resolveStatus("营业中", Number(record.stock) || 0, Number(record.capacity) || 0);
  } else {
    record.status = nextStatus(record.status);
  }
  persist();
}

function remove(id: string) {
  records.value = records.value.filter((record) => record.id !== id);
  ledger.value = ledger.value.filter((entry) => entry.stationId !== id);
  if (editingId.value === id) editingId.value = null;
  persist();
  persistLedger();
}

function formatTime(iso: string) {
  return new Date(iso).toLocaleString("zh-CN", { hour12: false });
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
              <input v-else-if="field.type === 'number'" v-model.number="form[field.key]" type="number" min="0" required />
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
                <div class="badges">
                  <span class="version">v{{ record.version }}</span>
                  <span class="status">{{ record.status }}</span>
                </div>
              </div>
              <div class="details">
                <span v-for="field in fields" :key="field.key">{{ field.label }}: {{ record[field.key] }}</span>
              </div>
              <p class="note">{{ record.notes }}</p>
              <div class="actions">
                <button type="button" @click="openStockEditor(record)">登记库存</button>
                <button class="secondary" type="button" @click="flow(record)">
                  {{ record.status === "暂停营业" ? "恢复营业" : "流转状态" }}
                </button>
                <button class="secondary" type="button" @click="navigator.clipboard?.writeText(primaryText(record))">复制摘要</button>
                <button class="danger" type="button" @click="remove(record.id)">删除</button>
              </div>

              <form v-if="editingId === record.id" class="stock-editor" @submit.prevent="submitStock">
                <div class="editor-meta">
                  <span>当前库存 {{ editSnapshot.stock }} L</span>
                  <span>容量 {{ editSnapshot.capacity }} L</span>
                  <span>警戒线 {{ Math.round(editSnapshot.capacity * 0.2) }} L（容量两成）</span>
                  <span>提交版本 v{{ editVersion }}</span>
                </div>
                <p v-if="editSnapshot.status === '暂停营业'" class="editor-hint">
                  该站暂停营业，提交仅记录流水，是否恢复营业由站长手动处理。
                </p>
                <div class="editor-grid">
                  <label>
                    班次
                    <select v-model="editForm.shift">
                      <option v-for="shift in shifts" :key="shift">{{ shift }}</option>
                    </select>
                  </label>
                  <label>
                    值班员
                    <input v-model="editForm.operator" placeholder="值班员姓名" required />
                  </label>
                  <label>
                    新库存L
                    <input v-model.number="editForm.stock" type="number" min="0" required />
                  </label>
                  <label>
                    交接备注
                    <input v-model="editForm.note" placeholder="本班交接说明" />
                  </label>
                </div>
                <p v-if="editError" class="editor-error">{{ editError }}</p>
                <div class="actions">
                  <button type="submit">提交库存</button>
                  <button class="secondary" type="button" @click="editingId = null">取消</button>
                </div>
              </form>
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

      <section class="panel ledger-panel">
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
          <article v-for="entry in filteredLedger" :key="entry.id" class="ledger-row">
            <span class="ledger-time">{{ formatTime(entry.createdAt) }}</span>
            <strong>{{ entry.station }}</strong>
            <span class="tag">{{ entry.shift }}</span>
            <span>{{ entry.operator }}</span>
            <span class="ledger-delta" :class="entry.afterStock >= entry.beforeStock ? 'up' : 'down'">
              {{ entry.beforeStock }} → {{ entry.afterStock }} L（{{ entry.afterStock - entry.beforeStock >= 0 ? "+" : "" }}{{ entry.afterStock - entry.beforeStock }}）
            </span>
            <span class="version">v{{ entry.version }}</span>
            <span class="ledger-note">{{ entry.note }}</span>
          </article>
        </div>
      </section>
    </div>
  </main>
</template>
