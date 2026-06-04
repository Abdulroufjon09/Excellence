<script setup>
import { ref, onMounted, computed } from "vue";
import { useRouter } from "vue-router";

const router = useRouter();
const API = "https://davomat-djang1-4.onrender.com/api";
const user = JSON.parse(localStorage.getItem("user") || "{}");

if (!user?.id) router.push("/login");

// ─── STATE ───────────────────────────────────────
const students = ref([]);
const payments = ref([]);
const myPenalties = ref([]);
const teacherName = ref("");

const loadingStudents = ref(true);
const loadingPayments = ref(true);
const loadingPenalties = ref(true);

const activeTab = ref("students");

const avatarColors = [
  { bg: "#ede9fe", color: "#5b21b6" },
  { bg: "#dcfce7", color: "#15803d" },
  { bg: "#fee2e2", color: "#b91c1c" },
  { bg: "#dbeafe", color: "#1d4ed8" },
  { bg: "#fef3c7", color: "#b45309" },
  { bg: "#fce7f3", color: "#9d174d" },
];

// ─── FETCH ───────────────────────────────────────
async function fetchStudents() {
  loadingStudents.value = true;
  try {
    const res = await fetch(`${API}/students/?teacher=${user.teacher_id}`);
    const data = await res.json();
    students.value = data;
    if (data.length > 0) teacherName.value = data[0].teacher_name;
  } catch (e) {
    console.error(e);
  } finally {
    loadingStudents.value = false;
  }
}

async function fetchPayments() {
  loadingPayments.value = true;
  try {
    const res = await fetch(`${API}/payments/${user.id}/`);
    payments.value = await res.json();
  } catch (e) {
    console.error(e);
  } finally {
    loadingPayments.value = false;
  }
}

async function fetchPenalties() {
  loadingPenalties.value = true;
  try {
    const res = await fetch(`${API}/penalties/student/${user.id}/`);
    myPenalties.value = await res.json();
  } catch (e) {
    console.error(e);
  } finally {
    loadingPenalties.value = false;
  }
}

onMounted(() => {
  Promise.all([fetchStudents(), fetchPayments(), fetchPenalties()]);
});

// ─── HELPERS ─────────────────────────────────────
function logout() {
  localStorage.removeItem("user");
  router.push("/login");
}

function initials(s) {
  return ((s.name?.[0] || "") + (s.surname?.[0] || "")).toUpperCase();
}

function formatMoney(value) {
  return Number(value || 0).toLocaleString("uz-UZ") + " so'm";
}

function formatMonth(month) {
  if (!month) return "";
  const [year, mon] = month.split("-");
  const months = [
    "Yanvar",
    "Fevral",
    "Mart",
    "Aprel",
    "May",
    "Iyun",
    "Iyul",
    "Avgust",
    "Sentabr",
    "Oktabr",
    "Noyabr",
    "Dekabr",
  ];
  return `${months[parseInt(mon) - 1]} ${year}`;
}

function formatDate(date) {
  if (!date) return "";
  const d = new Date(date.replace(" ", "T"));
  if (isNaN(d.getTime())) return date;
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
}

function stageColor(stage) {
  if (stage <= 2) return { bg: "#dcfce7", color: "#15803d" };
  if (stage <= 4) return { bg: "#dbeafe", color: "#1d4ed8" };
  return { bg: "#fef3c7", color: "#b45309" };
}
</script>

<template>
  <div class="cab-wrap">
    <!-- HEADER -->
    <header class="cab-header">
      <div class="header-left">
        <button
          v-if="user.is_admin"
          @click="$router.push('/admin')"
          class="back-btn"
          title="Admin panel"
        >
          <svg
            width="18"
            height="18"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
          >
            <polyline points="15 18 9 12 15 6" />
          </svg>
        </button>
        <div>
          <h1 class="header-title">Kabinet</h1>
          <p class="header-sub">Xush kelibsiz, {{ user.name }}</p>
        </div>
      </div>
      <button @click="logout" class="btn-ghost">
        <svg
          width="15"
          height="15"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4" />
          <polyline points="16 17 21 12 16 7" />
          <line x1="21" y1="12" x2="9" y2="12" />
        </svg>
        Chiqish
      </button>
    </header>

    <!-- PROFILE CARD -->
    <div class="profile-card">
      <div
        class="profile-avatar"
        :style="{
          background: avatarColors[0].bg,
          color: avatarColors[0].color,
        }"
      >
        {{ (user.name?.[0] || "").toUpperCase() }}
      </div>
      <div class="profile-info">
        <div class="profile-name-row">
          <span class="profile-name">{{ user.name }} {{ user.surname }}</span>
          <span v-if="user.is_admin" class="badge badge--admin">
            <svg width="11" height="11" viewBox="0 0 24 24" fill="currentColor">
              <path
                d="m12 17.275-4.15 2.5q-.275.175-.575.15t-.525-.2t-.35-.437t-.05-.588l1.1-4.725L3.775 10.8q-.25-.225-.312-.513t.037-.562t.3-.45t.55-.225l4.85-.425 1.875-4.45q.125-.3.388-.45t.537-.15t.537.15t.388.45l1.875 4.45 4.85.425q.35.05.55.225t.3.45t.038.563t-.313.512l-3.675 3.175 1.1 4.725q.075.325-.05.588t-.35.437t-.525.2t-.575-.15z"
              />
            </svg>
            Admin
          </span>
        </div>
        <span class="num_exam">
          <p class="profile-phone">{{ user.phone }}</p>
          <button class="exam">
            <a href="https://daraja-test.vercel.app/" target="_blank">
              📝 Exam
            </a>
          </button>
        </span>
        <p v-if="!user.is_admin" class="profile-teacher">
          <svg
            width="12"
            height="12"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            style="display: inline; vertical-align: -1px; margin-right: 3px"
          >
            <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2" />
            <circle cx="12" cy="7" r="4" />
          </svg>
          {{ teacherName }}
        </p>
      </div>
    </div>

    <!-- TABS -->
    <div class="tabs">
      <button
        @click="activeTab = 'students'"
        class="tab-btn"
        :class="{ active: activeTab === 'students' }"
      >
        <svg
          width="15"
          height="15"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2" />
          <circle cx="9" cy="7" r="4" />
          <path d="M23 21v-2a4 4 0 0 0-3-3.87" />
          <path d="M16 3.13a4 4 0 0 1 0 7.75" />
        </svg>
        Guruh
      </button>
      <button
        v-if="!user.is_admin"
        @click="activeTab = 'payments'"
        class="tab-btn"
        :class="{ active: activeTab === 'payments' }"
      >
        <svg
          width="15"
          height="15"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <rect x="1" y="4" width="22" height="16" rx="2" ry="2" />
          <line x1="1" y1="10" x2="23" y2="10" />
        </svg>
        To'lovlar
      </button>
      <button
        v-if="!user.is_admin"
        @click="activeTab = 'penalties'"
        class="tab-btn"
        :class="{ active: activeTab === 'penalties' }"
      >
        <svg
          width="15"
          height="15"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path
            d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"
          />
          <line x1="12" y1="9" x2="12" y2="13" />
          <line x1="12" y1="17" x2="12.01" y2="17" />
        </svg>
        Ja'zolar
        <span v-if="myPenalties.length" class="tab-count">{{
          myPenalties.length
        }}</span>
      </button>
    </div>

    <!-- ── STUDENTS TAB ── -->
    <div v-if="activeTab === 'students'">
      <p class="tab-meta">{{ students.length }} ta o'quvchi</p>
      <div v-if="loadingStudents" class="loading-state">
        <div class="spinner"></div>
        Yuklanmoqda...
      </div>
      <div v-else-if="students.length === 0" class="empty-state">
        O'quvchilar yo'q
      </div>
      <div v-else class="student-table-wrap">
        <table class="student-table">
          <thead>
            <tr>
              <th>#</th>
              <th>O'quvchi</th>
              <th v-if="user.is_admin">Telefon</th>
              <th>Etap</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(s, i) in students" :key="s.id">
              <td class="td-num">{{ i + 1 }}</td>
              <td>
                <div class="student-row">
                  <div
                    class="s-avatar"
                    :style="{
                      background: avatarColors[i % avatarColors.length].bg,
                      color: avatarColors[i % avatarColors.length].color,
                    }"
                  >
                    {{ initials(s) }}
                  </div>
                  <span class="s-name">{{ s.name }} {{ s.surname }}</span>
                </div>
              </td>
              <td v-if="user.is_admin" class="td-phone">{{ s.phone }}</td>
              <td>
                <span
                  class="stage-pill"
                  :style="{
                    background: stageColor(s.stage).bg,
                    color: stageColor(s.stage).color,
                  }"
                >
                  {{ s.stage }}-etap
                </span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- ── PAYMENTS TAB ── -->
    <div v-if="activeTab === 'payments'">
      <div v-if="loadingPayments" class="loading-state">
        <div class="spinner"></div>
        Yuklanmoqda...
      </div>
      <div v-else-if="payments.length === 0" class="empty-state">
        Hozircha to'lovlar mavjud emas
      </div>
      <div v-else class="payments-list">
        <div v-for="p in payments" :key="p.id" class="payment-card">
          <div class="payment-top">
            <div>
              <p class="payment-month">{{ formatMonth(p.month) }}</p>
              <p class="payment-stage">{{ p.stage }}-etap</p>
            </div>
            <div class="payment-status-col">
              <span
                class="payment-status"
                :class="p.is_paid ? 'paid' : 'unpaid'"
              >
                {{ p.is_paid ? "To'langan ✓" : "To'lanmagan" }}
              </span>
              <p v-if="p.paid_at" class="payment-date">
                {{ formatDate(p.paid_at) }}
              </p>
            </div>
          </div>
          <div class="payment-divider"></div>
          <div class="payment-bottom">
            <p class="payment-label">To'lov summasi</p>
            <p class="payment-amount">{{ formatMoney(p.amount_due) }}</p>
          </div>
        </div>
      </div>
    </div>

    <!-- ── PENALTIES TAB ── -->
    <div v-if="activeTab === 'penalties'">
      <div v-if="loadingPenalties" class="loading-state">
        <div class="spinner"></div>
        Yuklanmoqda...
      </div>
      <div v-else-if="myPenalties.length === 0" class="empty-state">
        Ja'zolar yo'q
      </div>
      <div v-else>
        <div class="penalty-summary">
          <svg
            width="16"
            height="16"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"
            />
            <line x1="12" y1="9" x2="12" y2="13" />
            <line x1="12" y1="17" x2="12.01" y2="17" />
          </svg>
          <span
            >Jami <strong>{{ myPenalties.length }}</strong> ta
            ogohlantirish</span
          >
        </div>
        <div class="penalties-list">
          <div v-for="p in myPenalties" :key="p.id" class="my-penalty-card">
            <div class="my-penalty-left">
              <span class="my-penalty-reason">{{ p.reason_display }}</span>
              <p v-if="p.description" class="my-penalty-desc">
                {{ p.description }}
              </p>
            </div>
            <p class="my-penalty-date">{{ p.date }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* ── BASE ── */
.cab-wrap {
  max-width: 720px;
  margin: 0 auto;
  padding: 0 16px 80px;
  font-family:
    "Inter",
    -apple-system,
    sans-serif;
}

/* ── HEADER ── */
.cab-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px 0 20px;
  border-bottom: 1px solid #f0f0f0;
  margin-bottom: 20px;
}
.header-left {
  display: flex;
  align-items: center;
  gap: 10px;
}
.header-title {
  font-size: 20px;
  font-weight: 600;
  color: #111;
  margin: 0;
  letter-spacing: -0.3px;
}
.header-sub {
  font-size: 13px;
  color: #999;
  margin: 2px 0 0;
}
.back-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 34px;
  height: 34px;
  border-radius: 8px;
  border: 1px solid #e5e5e5;
  background: transparent;
  color: #555;
  cursor: pointer;
  transition: background 0.15s;
  flex-shrink: 0;
}
.back-btn:hover {
  background: #f5f5f5;
}

.btn-ghost {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: transparent;
  color: #666;
  border: 1px solid #e5e5e5;
  border-radius: 8px;
  padding: 8px 14px;
  font-size: 13px;
  cursor: pointer;
  transition: background 0.15s;
}
.btn-ghost:hover {
  background: #f5f5f5;
}

/* ── PROFILE ── */
.profile-card {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 18px 20px;
  border: 1px solid #ebebeb;
  border-radius: 16px;
  margin-bottom: 20px;
  background: #fff;
}
.profile-avatar {
  width: 52px;
  height: 52px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  font-weight: 600;
  flex-shrink: 0;
}
.profile-info {
  flex: 1;
  min-width: 0;
}
.profile-name-row {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}
.profile-name {
  font-size: 16px;
  font-weight: 600;
  color: #111;
}
.badge--admin {
  display: inline-flex;
  align-items: center;
  gap: 3px;
  font-size: 11px;
  font-weight: 600;
  color: #b45309;
  background: #fef3c7;
  padding: 3px 8px;
  border-radius: 20px;
}
.profile-phone {
  font-size: 13px;
  color: #888;
  margin: 4px 0 0;
}
.profile-teacher {
  font-size: 12px;
  color: #aaa;
  margin: 3px 0 0;
}

/* ── TABS ── */
.tabs {
  display: flex;
  gap: 6px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}
.tab-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 9px 16px;
  border-radius: 10px;
  border: 1px solid #e5e5e5;
  background: transparent;
  color: #666;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
}
.tab-btn:hover {
  background: #f5f5f5;
}
.tab-btn.active {
  background: #111;
  color: #fff;
  border-color: #111;
}
.tab-count {
  background: #ef4444;
  color: #fff;
  font-size: 10px;
  font-weight: 700;
  padding: 1px 6px;
  border-radius: 10px;
}

/* ── TAB META ── */
.tab-meta {
  font-size: 12px;
  color: #aaa;
  margin: 0 0 12px;
}

/* ── STUDENTS TABLE ── */
.student-table-wrap {
  border: 1px solid #ebebeb;
  border-radius: 14px;
  overflow: hidden;
  overflow-x: auto;
}
.student-table {
  width: 100%;
  border-collapse: collapse;
  min-width: 340px;
}
.student-table thead {
  background: #f9f9f9;
}
.student-table th {
  padding: 11px 16px;
  text-align: left;
  font-size: 12px;
  font-weight: 600;
  color: #888;
  text-transform: uppercase;
  letter-spacing: 0.4px;
  border-bottom: 1px solid #ebebeb;
}
.student-table tr + tr td {
  border-top: 1px solid #f3f3f3;
}
.student-table td {
  padding: 12px 16px;
  font-size: 14px;
}
.student-table tbody tr:hover {
  background: #fafafa;
}
.td-num {
  color: #bbb;
  font-size: 13px;
  width: 40px;
}
.td-phone {
  color: #666;
  font-size: 13px;
}

.student-row {
  display: flex;
  align-items: center;
  gap: 10px;
}
.s-avatar {
  width: 34px;
  height: 34px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 700;
  flex-shrink: 0;
}
.s-name {
  font-size: 14px;
  font-weight: 500;
  color: #111;
}

.stage-pill {
  display: inline-block;
  padding: 4px 10px;
  border-radius: 20px;
  font-size: 12px;
  font-weight: 600;
}

/* ── PAYMENTS ── */
.payments-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.payment-card {
  border: 1px solid #ebebeb;
  border-radius: 14px;
  padding: 18px 20px;
  background: #fff;
}
.payment-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 14px;
}
.payment-month {
  font-size: 17px;
  font-weight: 600;
  color: #111;
  margin: 0;
}
.payment-stage {
  font-size: 12px;
  color: #aaa;
  margin: 4px 0 0;
}
.payment-status-col {
  text-align: right;
}
.payment-status {
  display: inline-block;
  font-size: 12px;
  font-weight: 600;
  padding: 5px 12px;
  border-radius: 20px;
}
.payment-status.paid {
  background: #dcfce7;
  color: #15803d;
}
.payment-status.unpaid {
  background: #fee2e2;
  color: #dc2626;
}
.payment-date {
  font-size: 11px;
  color: #bbb;
  margin: 5px 0 0;
}
.payment-divider {
  height: 1px;
  background: #f0f0f0;
  margin-bottom: 14px;
}
.payment-label {
  font-size: 12px;
  color: #aaa;
  margin: 0 0 4px;
}
.payment-amount {
  font-size: 28px;
  font-weight: 700;
  color: #111;
  margin: 0;
  letter-spacing: -0.5px;
}

/* ── PENALTIES ── */
.penalty-summary {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 16px;
  background: #fef3c7;
  border: 1px solid #fde68a;
  border-radius: 10px;
  margin-bottom: 14px;
  font-size: 13px;
  color: #92400e;
}
.penalty-summary strong {
  color: #78350f;
}

.penalties-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.my-penalty-card {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 14px 16px;
  border: 1px solid #fed7aa;
  border-radius: 12px;
  background: #fff7ed;
}
.my-penalty-left {
  flex: 1;
}
.my-penalty-reason {
  display: inline-block;
  font-size: 13px;
  font-weight: 600;
  color: #c2410c;
  background: #ffedd5;
  padding: 4px 10px;
  border-radius: 6px;
  margin-bottom: 5px;
}
.my-penalty-desc {
  font-size: 13px;
  color: #666;
  margin: 4px 0 0;
}
.my-penalty-date {
  font-size: 12px;
  color: #aaa;
  white-space: nowrap;
  padding-left: 12px;
  flex-shrink: 0;
}

/* ── STATES ── */
.loading-state {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 48px 16px;
  color: #bbb;
  font-size: 13px;
}
.empty-state {
  text-align: center;
  padding: 48px 16px;
  color: #ccc;
  font-size: 14px;
}
.spinner {
  width: 20px;
  height: 20px;
  border: 2px solid #e5e5e5;
  border-top-color: #999;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* ── RESPONSIVE ── */
@media (max-width: 480px) {
  .profile-card {
    padding: 14px;
  }
  .payment-amount {
    font-size: 22px;
  }
  .tabs {
    gap: 5px;
  }
  .tab-btn {
    padding: 8px 12px;
    font-size: 12px;
  }
}
.exam {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 6px;
  background: transparent;
  color: #555;
  cursor: pointer;
  transition: background 0.15s;
}

.num_exam {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
}
</style>
