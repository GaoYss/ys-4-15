<template>
  <div class="page-stack">
    <section class="panel">
      <div class="panel-head">
        <h2>欠费账单</h2>
        <div class="toolbar">
          <select v-model="channel">
            <option value="sms">短信</option>
            <option value="phone">电话</option>
            <option value="wechat">微信</option>
            <option value="notice">纸质通知</option>
          </select>
          <button :disabled="generating" @click="previewReminders">{{ generating ? '处理中...' : '批量生成催缴' }}</button>
        </div>
      </div>
      <DataTable :columns="billColumns" :rows="debtBills">
        <template #cell-status="{ row }"><StatusBadge :status="row.status" /></template>
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
      </DataTable>
    </section>

    <section v-if="lastResult" class="panel">
      <div class="panel-head">
        <h2>催缴结果</h2>
        <button @click="lastResult = null">关闭</button>
      </div>
      <div class="result-summary">
        <span>本次生成 <strong>{{ lastResult.created_count }}</strong> 条催缴</span>
        <span v-if="lastResult.skipped_count > 0">，跳过当日已催缴 <strong>{{ lastResult.skipped_count }}</strong> 条</span>
      </div>
      <DataTable :columns="resultColumns" :rows="lastResult.created">
        <template #cell-status="{ row }"><StatusBadge :status="row.status" /></template>
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
      </DataTable>
    </section>

    <section class="panel">
      <div class="panel-head">
        <h2>催缴记录</h2>
        <button @click="load">刷新</button>
      </div>
      <DataTable :columns="reminderColumns" :rows="reminders">
        <template #cell-status="{ row }"><StatusBadge :status="row.status" /></template>
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
      </DataTable>
    </section>

    <div v-if="showPreview" class="modal-overlay" @click.self="showPreview = false">
      <div class="modal">
        <h3>确认批量催缴</h3>
        <div class="preview-info">
          <p>催缴渠道：<strong>{{ channelLabels[channel] }}</strong></p>
          <p>逾期账单总数：<strong>{{ preview.total }}</strong></p>
          <p>当日已催缴跳过：<strong>{{ preview.skip_count }}</strong></p>
          <p>预计发送通知数：<strong class="highlight">{{ preview.actual_count }}</strong></p>
        </div>
        <div class="modal-actions">
          <button class="btn-secondary" @click="showPreview = false">取消</button>
          <button :disabled="generating" @click="confirmCreate">{{ generating ? '生成中...' : '确认生成' }}</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, ref } from "vue";
import { propertyApi } from "../api/property";
import DataTable from "../components/DataTable.vue";
import StatusBadge from "../components/StatusBadge.vue";

const bills = ref([]);
const reminders = ref([]);
const channel = ref("sms");
const generating = ref(false);
const showPreview = ref(false);
const preview = ref({ total: 0, skip_count: 0, actual_count: 0 });
const lastResult = ref(null);

const channelLabels = { sms: "短信", phone: "电话", wechat: "微信", notice: "纸质通知" };

const debtBills = computed(() => bills.value.filter((bill) => ["unpaid", "overdue"].includes(bill.status)));
const billColumns = [
  { key: "room_label", label: "房屋" },
  { key: "owner_name", label: "业主" },
  { key: "phone", label: "电话" },
  { key: "fee_name", label: "费用" },
  { key: "amount", label: "欠费" },
  { key: "due_date", label: "截止日期" },
  { key: "status", label: "状态" }
];
const resultColumns = [
  { key: "reminder_no", label: "催缴编号" },
  { key: "room_label", label: "房屋" },
  { key: "owner_name", label: "业主" },
  { key: "channel", label: "渠道" },
  { key: "amount", label: "欠费" },
  { key: "result", label: "催缴结果" },
  { key: "status", label: "状态" }
];
const reminderColumns = [
  { key: "reminder_no", label: "催缴编号" },
  { key: "room_label", label: "房屋" },
  { key: "owner_name", label: "业主" },
  { key: "channel", label: "渠道" },
  { key: "amount", label: "欠费" },
  { key: "result", label: "催缴结果" },
  { key: "message", label: "内容" },
  { key: "status", label: "状态" }
];

async function load() {
  [bills.value, reminders.value] = await Promise.all([propertyApi.listBills(), propertyApi.listReminders()]);
}

async function previewReminders() {
  generating.value = true;
  try {
    preview.value = await propertyApi.previewOverdueReminders({ channel: channel.value });
    if (preview.value.actual_count === 0) {
      alert("当前没有需要催缴的账单（所有逾期账单今日已催缴或无逾期账单）。");
      return;
    }
    showPreview.value = true;
  } finally {
    generating.value = false;
  }
}

async function confirmCreate() {
  generating.value = true;
  try {
    const result = await propertyApi.createOverdueReminders({ channel: channel.value });
    lastResult.value = result;
    showPreview.value = false;
    await load();
  } finally {
    generating.value = false;
  }
}

onMounted(load);
</script>
