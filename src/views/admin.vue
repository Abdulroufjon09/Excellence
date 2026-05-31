<script setup>
import { ref, onMounted, computed } from "vue";
import { useRouter } from "vue-router";

const router = useRouter();
const API = "https://davomat-djang1-4.onrender.com/api";

const user = JSON.parse(localStorage.getItem("user") || "null");
if (!user) router.push("/login");
if (user && !user.is_admin) router.push("/");

function logout() {
  localStorage.removeItem("user");
  router.push("/login");
}

// ─── STATE ───────────────────────────────────────
const teachers = ref([]);
const students = ref([]);
const payments = ref([]);

const loadingStudents = ref(false);
const loadingPayments = ref(false);
const stageLoading = ref(null);

const showForm = ref(false);
const showReassign = ref(false);
const showStudents = ref(false);
const showTeacherPenalties = ref(false);

const newTeacher = ref({ name: "", phone: "" });
const fromTeacher = ref(null);
const toTeacher = ref(null);
const selectedTeacherId = ref(null);

const STAGES = [1, 2, 3, 4, 5, 6, 7];

// ─── TEACHER JA'ZOLARI ───────────────────────────
const teacherPenalties = ref([]);
const loadingTeacherPenalties = ref(false);
const showTeacherPenaltyForm = ref(false);
const penaltyTeacherId = ref(null);
const newTeacherPenalty = ref({ reason: "other", description: "" });

// ─── STUDENT JA'ZOLARI ───────────────────────────
const studentPenaltiesMap = ref({});
const loadingStudentPenalties = ref({});
const openStudentPenaltyId = ref(null);
const newStudentPenalty = ref({ reason: "other", description: "" });
const showStudentPenaltyForm = ref(false);

const PENALTY_REASONS = [
  { value: "late",     label: "Kech kelish" },
  { value: "absent",   label: "Darsga kelmadi" },
  { value: "behavior", label: "Xulq-atvor" },
  { value: "other",    label: "Boshqa" },
];

// ─── COMPUTED ────────────────────────────────────
const selectedTeacherName = computed(() =>
  teachers.value.find(t => t.id === selectedTeacherId.value)?.name || ""
);
const penaltyTeacherName = computed(() =>
  teachers.value.find(t => t.id === penaltyTeacherId.value)?.name || ""
);
const teacherPenaltyCount = computed(() => teacherPenalties.value.length);

// ─── FETCH ───────────────────────────────────────
onMounted(fetchTeachers);

async function fetchTeachers() {
  try {
    const res = await fetch(`${API}/teachers/`);
    teachers.value = await res.json();
  } catch (e) {
    console.error(e);
  }
}

async function fetchStudents(teacherId) {
  loadingStudents.value = true;
  try {
    const res = await fetch(`${API}/students/?teacher_id=${teacherId}`);
    students.value = await res.json();
  } catch (e) {
    alert("Studentlarni yuklashda xatolik");
  } finally {
    loadingStudents.value = false;
  }
}

async function fetchPayments(teacherId) {
  loadingPayments.value = true;
  try {
    const res = await fetch(`${API}/payments/?teacher_id=${teacherId}`);
    payments.value = await res.json();
  } catch (e) {
    console.error(e);
  } finally {
    loadingPayments.value = false;
  }
}

async function fetchTeacherPenalties(teacherId) {
  loadingTeacherPenalties.value = true;
  try {
    const res = await fetch(`${API}/penalties/teacher/${teacherId}/`);
    teacherPenalties.value = await res.json();
  } catch (e) {
    console.error(e);
  } finally {
    loadingTeacherPenalties.value = false;
  }
}

async function fetchStudentPenalties(studentId) {
  loadingStudentPenalties.value = { ...loadingStudentPenalties.value, [studentId]: true };
  try {
    const res = await fetch(`${API}/penalties/student/${studentId}/`);
    const data = await res.json();
    studentPenaltiesMap.value = { ...studentPenaltiesMap.value, [studentId]: data };
  } catch (e) {
    console.error(e);
  } finally {
    loadingStudentPenalties.value = { ...loadingStudentPenalties.value, [studentId]: false };
  }
}

// ─── TEACHER ACTIONS ─────────────────────────────
async function createTeacher() {
  if (!newTeacher.value.name.trim()) return alert("Ism kiriting");
  try {
    const res = await fetch(`${API}/teachers/create/`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(newTeacher.value),
    });
    if (!res.ok) throw new Error();
    newTeacher.value = { name: "", phone: "" };
    showForm.value = false;
    await fetchTeachers();
  } catch {
    alert("Xatolik yuz berdi");
  }
}

async function deleteTeacher(id) {
  if (!confirm("O'chirishni tasdiqlaysizmi?")) return;
  try {
    const res = await fetch(`${API}/teachers/${id}/delete/`, { method: "DELETE" });
    if (!res.ok) {
      const data = await res.json().catch(() => ({}));
      return alert(data.error || "O'chirishda xatolik");
    }
    if (selectedTeacherId.value === id) {
      selectedTeacherId.value = null;
      showStudents.value = false;
      students.value = [];
    }
    if (penaltyTeacherId.value === id) {
      penaltyTeacherId.value = null;
      showTeacherPenalties.value = false;
    }
    await fetchTeachers();
  } catch {
    alert("Server bilan aloqa yo'q");
  }
}

async function reassignStudents() {
  if (!fromTeacher.value || !toTeacher.value) return alert("Teacher tanlang");
  if (fromTeacher.value === toTeacher.value) return alert("Bir xil teacher tanlandi");
  try {
    const res = await fetch(`${API}/teachers/reassign/`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ from_teacher_id: fromTeacher.value, to_teacher_id: toTeacher.value }),
    });
    const data = await res.json();
    if (!res.ok) return alert(data.error || "Xatolik");
    showReassign.value = false;
    fromTeacher.value = null;
    toTeacher.value = null;
    if (selectedTeacherId.value === fromTeacher.value) await fetchStudents(selectedTeacherId.value);
  } catch {
    alert("Server bilan aloqa yo'q");
  }
}

// ─── SELECT ──────────────────────────────────────
async function selectTeacher(teacherId) {
  if (showStudents.value && selectedTeacherId.value === teacherId) {
    showStudents.value = false;
    selectedTeacherId.value = null;
    return;
  }
  selectedTeacherId.value = teacherId;
  showStudents.value = true;
  showTeacherPenalties.value = false;
  studentPenaltiesMap.value = {};
  openStudentPenaltyId.value = null;
  await Promise.all([fetchStudents(teacherId), fetchPayments(teacherId)]);
}

async function selectTeacherPenalties(teacherId) {
  if (showTeacherPenalties.value && penaltyTeacherId.value === teacherId) {
    showTeacherPenalties.value = false;
    penaltyTeacherId.value = null;
    return;
  }
  penaltyTeacherId.value = teacherId;
  showTeacherPenalties.value = true;
  showStudents.value = false;
  showTeacherPenaltyForm.value = false;
  await fetchTeacherPenalties(teacherId);
}

// ─── STAGE & SCHEDULE ────────────────────────────
async function updateStage(student, stage) {
  if (stageLoading.value === student.id) return;
  stageLoading.value = student.id;
  try {
    const res = await fetch(`${API}/students/update/${student.id}/`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ stage }),
    });
    const result = await res.json();
    if (!res.ok) return alert(result.error || "Xatolik");
    if (stage === 5 && result.teacher_id !== selectedTeacherId.value) {
      students.value = students.value.filter(s => s.id !== student.id);
    } else {
      const s = students.value.find(s => s.id === student.id);
      if (s) { s.stage = result.stage; s.teacher_id = result.teacher_id; }
    }
  } catch {
    alert("Server bilan aloqa yo'q");
  } finally {
    stageLoading.value = null;
  }
}

async function updateSchedule(student, schedule) {
  await fetch(`${API}/students/update/${student.id}/`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ schedule }),
  });
  student.schedule = schedule;
}

// ─── TEACHER PENALTIES ───────────────────────────
async function createTeacherPenalty() {
  if (!penaltyTeacherId.value) return;
  try {
    const res = await fetch(`${API}/penalties/create/`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ teacher_id: penaltyTeacherId.value, ...newTeacherPenalty.value }),
    });
    if (!res.ok) throw new Error();
    newTeacherPenalty.value = { reason: "other", description: "" };
    showTeacherPenaltyForm.value = false;
    await fetchTeacherPenalties(penaltyTeacherId.value);
  } catch {
    alert("Xatolik yuz berdi");
  }
}

async function deleteTeacherPenalty(id) {
  if (!confirm("O'chirishni tasdiqlaysizmi?")) return;
  await fetch(`${API}/penalties/${id}/delete/`, { method: "DELETE" });
  teacherPenalties.value = teacherPenalties.value.filter(p => p.id !== id);
}

// ─── STUDENT PENALTIES ───────────────────────────
async function toggleStudentPenalties(studentId) {
  if (openStudentPenaltyId.value === studentId) {
    openStudentPenaltyId.value = null;
    showStudentPenaltyForm.value = false;
    return;
  }
  openStudentPenaltyId.value = studentId;
  showStudentPenaltyForm.value = false;
  if (!studentPenaltiesMap.value[studentId]) {
    await fetchStudentPenalties(studentId);
  }
}

async function createStudentPenalty(studentId) {
  try {
    const res = await fetch(`${API}/penalties/create/`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ student_id: studentId, ...newStudentPenalty.value }),
    });
    if (!res.ok) throw new Error();
    newStudentPenalty.value = { reason: "other", description: "" };
    showStudentPenaltyForm.value = false;
    await fetchStudentPenalties(studentId);
  } catch {
    alert("Xatolik yuz berdi");
  }
}

async function deleteStudentPenalty(penaltyId, studentId) {
  if (!confirm("O'chirishni tasdiqlaysizmi?")) return;
  await fetch(`${API}/penalties/${penaltyId}/delete/`, { method: "DELETE" });
  studentPenaltiesMap.value[studentId] = studentPenaltiesMap.value[studentId].filter(p => p.id !== penaltyId);
}

// ─── HELPERS ─────────────────────────────────────
function getStudentPayment(studentId) {
  return payments.value.find(p => p.student_id === studentId);
}

function formatMoney(value) {
  return Number(value || 0).toLocaleString("uz-UZ") + " so'm";
}

function reasonLabel(reason) {
  return PENALTY_REASONS.find(r => r.value === reason)?.label || reason;
}
</script>

<template>
  <div class="admin-wrap">
    <!-- HEADER -->
    <header class="admin-header">
      <div class="header-left">
        <div class="logo-dot"></div>
        <div>
          <h1 class="header-title">Admin panel</h1>
          <p class="header-sub">Xush kelibsiz, {{ user?.name }}</p>
        </div>
      </div>
      <button @click="logout" class="btn-ghost">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"/><polyline points="16 17 21 12 16 7"/><line x1="21" y1="12" x2="9" y2="12"/></svg>
        Chiqish
      </button>
    </header>

    <!-- QUICK NAV -->
    <div class="quick-nav">
      <div class="nav-card" @click="$router.push('/students')">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M23 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/></svg>
        <span>O'quvchilar</span>
      </div>
      <div class="nav-card" @click="$router.push('/attendance')">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/><polyline points="10 9 9 9 8 9"/></svg>
        <span>Davomat</span>
      </div>
    </div>

    <!-- TEACHERS SECTION -->
    <section class="section">
      <div class="section-header">
        <h2 class="section-title">O'qituvchilar</h2>
        <div class="header-actions">
          <button @click="showReassign = !showReassign" class="btn-outline">
            O'tkazish
          </button>
          <button @click="showForm = !showForm" class="btn-primary">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>
            Qo'shish
          </button>
        </div>
      </div>

      <!-- ADD FORM -->
      <transition name="slide">
        <div v-if="showForm" class="form-box">
          <div class="form-grid">
            <input v-model="newTeacher.name" placeholder="Ism familya" class="input" />
            <input v-model="newTeacher.phone" placeholder="Telefon" class="input" />
          </div>
          <button @click="createTeacher" class="btn-primary w-full">Saqlash</button>
        </div>
      </transition>

      <!-- REASSIGN -->
      <transition name="slide">
        <div v-if="showReassign" class="form-box form-box--warn">
          <p class="form-label">O'quvchilarni o'tkazish</p>
          <div class="form-grid">
            <select v-model.number="fromTeacher" class="input">
              <option :value="null">Kimdan</option>
              <option v-for="t in teachers" :key="t.id" :value="t.id">{{ t.name }}</option>
            </select>
            <select v-model.number="toTeacher" class="input">
              <option :value="null">Kimga</option>
              <option v-for="t in teachers" :key="t.id" :value="t.id">{{ t.name }}</option>
            </select>
          </div>
          <button @click="reassignStudents" class="btn-warn w-full">O'tkazish</button>
        </div>
      </transition>

      <!-- TEACHER LIST -->
      <div class="teacher-list">
        <div
          v-for="t in teachers"
          :key="t.id"
          class="teacher-row"
          :class="{ 'teacher-row--active': selectedTeacherId === t.id && showStudents }"
        >
          <div class="teacher-info">
            <div class="teacher-avatar">{{ t.name[0] }}</div>
            <div>
              <p class="teacher-name">{{ t.name }}</p>
              <p class="teacher-phone">{{ t.phone || "—" }}</p>
            </div>
          </div>
          <div class="teacher-actions">
            <button
              @click="selectTeacher(t.id)"
              class="action-btn action-btn--blue"
              :class="{ active: selectedTeacherId === t.id && showStudents }"
            >
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/></svg>
              O'quvchilar
            </button>

            <button @click="deleteTeacher(t.id)" class="action-btn action-btn--red">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"/><path d="M19 6l-1 14a2 2 0 0 1-2 2H8a2 2 0 0 1-2-2L5 6"/><path d="M10 11v6"/><path d="M14 11v6"/></svg>
              O'chirish
            </button>
          </div>
        </div>
      </div>
    </section>

    <!-- STUDENTS PANEL -->
    <transition name="panel">
      <section v-if="showStudents" class="section section--blue">
        <div class="section-header">
          <div>
            <h2 class="section-title">{{ selectedTeacherName }}</h2>
            <p class="section-sub">{{ students.length }} ta o'quvchi</p>
          </div>
        </div>

        <div v-if="loadingStudents" class="loading-state">
          <div class="spinner"></div>
          <span>Yuklanmoqda...</span>
        </div>

        <div v-else-if="students.length === 0" class="empty-state">
          O'quvchilar yo'q
        </div>

        <div v-else class="student-list">
          <div v-for="s in students" :key="s.id" class="student-card">
            <!-- Student main info -->
            <div class="student-main">
              <div class="student-avatar">
                {{ (s.name[0] || "") + (s.surname?.[0] || "") }}
              </div>
              <div class="student-info">
                <p class="student-name">{{ s.name }} {{ s.surname }}</p>
                <p class="student-phone">{{ s.phone }}</p>
                <div v-if="getStudentPayment(s.id)" class="payment-badge-row">
                  <span class="payment-badge" :class="getStudentPayment(s.id).is_paid ? 'paid' : 'unpaid'">
                    {{ getStudentPayment(s.id).is_paid ? "To'langan ✓" : "To'lanmagan" }}
                  </span>
                  <span class="payment-amount">{{ formatMoney(getStudentPayment(s.id).amount_due) }}</span>
                </div>
              </div>
              <button
                @click="toggleStudentPenalties(s.id)"
                class="penalty-toggle-btn"
                :class="{ active: openStudentPenaltyId === s.id }"
              >
                <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"/><line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/></svg>
                <span>Ja'zo</span>
                <span v-if="studentPenaltiesMap[s.id]?.length" class="penalty-count">
                  {{ studentPenaltiesMap[s.id].length }}
                </span>
              </button>
            </div>

            <!-- Stage buttons -->
            <div class="stage-row">
              <button
                v-for="st in STAGES"
                :key="st"
                @click="updateStage(s, st)"
                class="stage-btn"
                :class="{ 'stage-btn--active': s.stage === st, 'stage-btn--loading': stageLoading === s.id }"
              >
                {{ st }}
              </button>
            </div>

            <!-- Schedule buttons -->
            <div class="schedule-row">
              <button
                @click="updateSchedule(s, 'odd')"
                class="schedule-btn"
                :class="{ 'schedule-btn--active': s.schedule === 'odd' }"
              >
                Du / Chor / Juma
              </button>
              <button
                @click="updateSchedule(s, 'even')"
                class="schedule-btn"
                :class="{ 'schedule-btn--active': s.schedule === 'even' }"
              >
                Se / Pay / Shan
              </button>
            </div>

            <!-- Student penalties panel -->
            <transition name="slide">
              <div v-if="openStudentPenaltyId === s.id" class="student-penalty-panel">
                <div class="sp-header">
                  <span class="sp-title">Ja'zolar</span>
                  <button
                    @click="showStudentPenaltyForm = !showStudentPenaltyForm"
                    class="sp-add-btn"
                  >
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>
                    Qo'shish
                  </button>
                </div>

                <!-- penalty form -->
                <transition name="slide">
                  <div v-if="showStudentPenaltyForm" class="sp-form">
                    <select v-model="newStudentPenalty.reason" class="input input--sm">
                      <option v-for="r in PENALTY_REASONS" :key="r.value" :value="r.value">
                        {{ r.label }}
                      </option>
                    </select>
                    <input
                      v-model="newStudentPenalty.description"
                      placeholder="Izoh (ixtiyoriy)"
                      class="input input--sm"
                    />
                    <button @click="createStudentPenalty(s.id)" class="btn-primary btn--sm w-full">
                      Saqlash
                    </button>
                  </div>
                </transition>

                <!-- loading -->
                <div v-if="loadingStudentPenalties[s.id]" class="sp-loading">
                  <div class="spinner spinner--sm"></div>
                </div>

                <!-- list -->
                <div v-else-if="studentPenaltiesMap[s.id]?.length > 0">
                  <div
                    v-for="p in studentPenaltiesMap[s.id]"
                    :key="p.id"
                    class="sp-item"
                  >
                    <div class="sp-item-left">
                      <span class="sp-reason-badge">{{ p.reason_display }}</span>
                      <p v-if="p.description" class="sp-desc">{{ p.description }}</p>
                      <p class="sp-date">{{ p.date }}</p>
                    </div>
                    <button @click="deleteStudentPenalty(p.id, s.id)" class="sp-delete">
                      <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>
                    </button>
                  </div>
                </div>

                <div v-else class="sp-empty">Ja'zolar yo'q</div>
              </div>
            </transition>
          </div>
        </div>
      </section>
    </transition>

    <!-- TEACHER PENALTIES PANEL -->
    <transition name="panel">
      <section v-if="showTeacherPenalties" class="section section--orange">
        <div class="section-header">
          <div>
            <h2 class="section-title">{{ penaltyTeacherName }}</h2>
            <p class="section-sub">{{ teacherPenaltyCount }} ta ja'zo</p>
          </div>
          <button
            @click="showTeacherPenaltyForm = !showTeacherPenaltyForm"
            class="btn-orange"
          >
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>
            Qo'shish
          </button>
        </div>

        <transition name="slide">
          <div v-if="showTeacherPenaltyForm" class="form-box form-box--orange">
            <select v-model="newTeacherPenalty.reason" class="input">
              <option v-for="r in PENALTY_REASONS" :key="r.value" :value="r.value">
                {{ r.label }}
              </option>
            </select>
            <input
              v-model="newTeacherPenalty.description"
              placeholder="Izoh (ixtiyoriy)"
              class="input"
            />
            <button @click="createTeacherPenalty" class="btn-orange w-full">Saqlash</button>
          </div>
        </transition>

        <div v-if="loadingTeacherPenalties" class="loading-state">
          <div class="spinner"></div>
          <span>Yuklanmoqda...</span>
        </div>
        <div v-else-if="teacherPenalties.length === 0" class="empty-state">
          Ja'zolar yo'q
        </div>
        <div v-else class="penalty-list">
          <div v-for="p in teacherPenalties" :key="p.id" class="penalty-item">
            <div class="penalty-item-left">
              <span class="penalty-reason-badge">{{ p.reason_display }}</span>
              <p v-if="p.description" class="penalty-desc">{{ p.description }}</p>
              <p class="penalty-date">{{ p.date }}</p>
            </div>
            <button @click="deleteTeacherPenalty(p.id)" class="penalty-delete">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>
            </button>
          </div>
        </div>
      </section>
    </transition>
  </div>
</template>

<style scoped>
/* ── BASE ── */
.admin-wrap {
  max-width: 860px;
  margin: 0 auto;
  padding: 0 16px 80px;
  font-family: 'Inter', -apple-system, sans-serif;
}

/* ── HEADER ── */
.admin-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px 0 20px;
  border-bottom: 1px solid #f0f0f0;
  margin-bottom: 24px;
}
.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}
.logo-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: #111;
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

/* ── QUICK NAV ── */
.quick-nav {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-bottom: 24px;
}
.nav-card {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 16px 20px;
  border: 1px solid #ebebeb;
  border-radius: 14px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  color: #333;
  transition: all 0.15s;
}
.nav-card:hover {
  background: #f8f8f8;
  border-color: #ddd;
}
.nav-card svg {
  color: #888;
}

/* ── SECTION ── */
.section {
  background: #fff;
  border: 1px solid #ebebeb;
  border-radius: 16px;
  padding: 20px;
  margin-bottom: 16px;
}
.section--blue {
  border-color: #dbeafe;
  background: #fafcff;
}
.section--orange {
  border-color: #fed7aa;
  background: #fffaf6;
}
.section-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 16px;
}
.section-title {
  font-size: 15px;
  font-weight: 600;
  color: #111;
  margin: 0;
}
.section-sub {
  font-size: 12px;
  color: #999;
  margin: 3px 0 0;
}
.header-actions {
  display: flex;
  gap: 8px;
}

/* ── BUTTONS ── */
.btn-primary {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: #111;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 8px 14px;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: opacity 0.15s;
}
.btn-primary:hover { opacity: 0.85; }
.btn-primary.w-full { width: 100%; justify-content: center; }

.btn-outline {
  background: transparent;
  color: #555;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 8px 14px;
  font-size: 13px;
  cursor: pointer;
  transition: background 0.15s;
}
.btn-outline:hover { background: #f5f5f5; }

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
.btn-ghost:hover { background: #f5f5f5; }

.btn-warn {
  background: #f59e0b;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 9px 16px;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  width: 100%;
  transition: opacity 0.15s;
}
.btn-warn:hover { opacity: 0.9; }

.btn-orange {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: #ea580c;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 8px 14px;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: opacity 0.15s;
}
.btn-orange:hover { opacity: 0.9; }
.btn-orange.w-full { width: 100%; justify-content: center; }

.btn--sm {
  padding: 7px 12px;
  font-size: 12px;
}

/* ── FORMS ── */
.form-box {
  background: #f9f9f9;
  border: 1px solid #ebebeb;
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 16px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.form-box--warn {
  background: #fffbeb;
  border-color: #fde68a;
}
.form-box--orange {
  background: #fff7ed;
  border-color: #fed7aa;
}
.form-label {
  font-size: 13px;
  font-weight: 500;
  color: #555;
  margin: 0;
}
.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}
.input {
  width: 100%;
  padding: 9px 12px;
  border: 1px solid #e5e5e5;
  border-radius: 8px;
  font-size: 13px;
  color: #111;
  background: #fff;
  outline: none;
  box-sizing: border-box;
  transition: border-color 0.15s;
}
.input:focus { border-color: #aaa; }
.input--sm {
  padding: 7px 10px;
  font-size: 12px;
}

/* ── TEACHER LIST ── */
.teacher-list { display: flex; flex-direction: column; gap: 4px; }
.teacher-row {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 12px;
  border-radius: 10px;
  transition: background 0.1s;
}
.teacher-row:hover { background: #f8f8f8; }
.teacher-row--active { background: #f0f7ff; }

.teacher-info {
  display: flex;
  align-items: center;
  gap: 10px;
}
.teacher-avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: #111;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  font-weight: 600;
  text-transform: uppercase;
  flex-shrink: 0;
}
.teacher-name { font-size: 14px; font-weight: 500; color: #111; margin: 0; }
.teacher-phone { font-size: 12px; color: #999; margin: 2px 0 0; }

.teacher-actions { display: flex; gap: 6px; flex-wrap: wrap; }

.action-btn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 6px 11px;
  border-radius: 7px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  border: 1px solid;
  transition: all 0.15s;
}
.action-btn--blue {
  color: #2563eb;
  border-color: #bfdbfe;
  background: transparent;
}
.action-btn--blue:hover, .action-btn--blue.active { background: #eff6ff; }

.action-btn--orange {
  color: #ea580c;
  border-color: #fed7aa;
  background: transparent;
}
.action-btn--orange:hover, .action-btn--orange.active { background: #fff7ed; }

.action-btn--red {
  color: #dc2626;
  border-color: #fecaca;
  background: transparent;
}
.action-btn--red:hover { background: #fef2f2; }

/* ── STUDENTS ── */
.student-list { display: flex; flex-direction: column; gap: 12px; }

.student-card {
  background: #fff;
  border: 1px solid #e8edf5;
  border-radius: 12px;
  padding: 14px;
}

.student-main {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  margin-bottom: 12px;
}
.student-avatar {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  background: #1d4ed8;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  font-weight: 600;
  flex-shrink: 0;
  text-transform: uppercase;
}
.student-info { flex: 1; min-width: 0; }
.student-name { font-size: 14px; font-weight: 500; color: #111; margin: 0; }
.student-phone { font-size: 12px; color: #999; margin: 2px 0 0; }
.payment-badge-row { display: flex; align-items: center; gap: 8px; margin-top: 6px; }
.payment-badge {
  font-size: 11px;
  padding: 3px 8px;
  border-radius: 20px;
  font-weight: 500;
}
.payment-badge.paid { background: #dcfce7; color: #15803d; }
.payment-badge.unpaid { background: #fee2e2; color: #dc2626; }
.payment-amount { font-size: 12px; color: #666; }

/* penalty toggle button */
.penalty-toggle-btn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 6px 10px;
  border: 1px solid #fed7aa;
  background: transparent;
  color: #ea580c;
  border-radius: 7px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
  flex-shrink: 0;
  margin-left: auto;
}
.penalty-toggle-btn:hover, .penalty-toggle-btn.active {
  background: #fff7ed;
}
.penalty-count {
  background: #ea580c;
  color: #fff;
  font-size: 10px;
  font-weight: 600;
  padding: 1px 5px;
  border-radius: 10px;
}

/* ── STAGE ── */
.stage-row {
  display: flex;
  gap: 5px;
  margin-bottom: 8px;
}
.stage-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  border: 1px solid #e5e5e5;
  background: transparent;
  font-size: 12px;
  font-weight: 500;
  color: #555;
  cursor: pointer;
  transition: all 0.15s;
}
.stage-btn:hover { border-color: #aaa; }
.stage-btn--active { background: #111; color: #fff; border-color: #111; }
.stage-btn--loading { opacity: 0.5; pointer-events: none; }

/* ── SCHEDULE ── */
.schedule-row { display: flex; gap: 6px; }
.schedule-btn {
  flex: 1;
  padding: 7px;
  border: 1px solid #e5e5e5;
  border-radius: 8px;
  font-size: 12px;
  color: #888;
  background: transparent;
  cursor: pointer;
  transition: all 0.15s;
}
.schedule-btn:hover { background: #f5f5f5; color: #000000;}
.schedule-btn--active { background: #000000; color: #fff; border-color: #111; }

/* ── STUDENT PENALTY PANEL ── */
.student-penalty-panel {
  margin-top: 12px;
  border-top: 1px dashed #fed7aa;
  padding-top: 12px;
}
.sp-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}
.sp-title { font-size: 13px; font-weight: 600; color: #ea580c; }
.sp-add-btn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  font-weight: 500;
  color: #ea580c;
  background: #fff7ed;
  border: 1px solid #fed7aa;
  border-radius: 6px;
  padding: 5px 10px;
  cursor: pointer;
  transition: background 0.15s;
}
.sp-add-btn:hover { background: #ffedd5; }

.sp-form {
  background: #fff7ed;
  border: 1px solid #fed7aa;
  border-radius: 10px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 10px;
}

.sp-loading {
  display: flex;
  justify-content: center;
  padding: 16px;
}

.sp-item {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 10px 12px;
  background: #fff7ed;
  border: 1px solid #fed7aa;
  border-radius: 9px;
  margin-bottom: 6px;
}
.sp-item-left { flex: 1; }
.sp-reason-badge {
  display: inline-block;
  font-size: 12px;
  font-weight: 500;
  color: #c2410c;
  background: #ffedd5;
  padding: 3px 8px;
  border-radius: 5px;
  margin-bottom: 4px;
}
.sp-desc { font-size: 12px; color: #666; margin: 4px 0 2px; }
.sp-date { font-size: 11px; color: #aaa; margin: 0; }
.sp-delete {
  background: transparent;
  border: none;
  color: #ccc;
  cursor: pointer;
  padding: 2px;
  line-height: 1;
  transition: color 0.15s;
  flex-shrink: 0;
}
.sp-delete:hover { color: #ef4444; }
.sp-empty { font-size: 13px; color: #bbb; text-align: center; padding: 12px; }

/* ── TEACHER PENALTY ── */
.penalty-list { display: flex; flex-direction: column; gap: 6px; }
.penalty-item {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 12px;
  background: #fff7ed;
  border: 1px solid #fed7aa;
  border-radius: 10px;
}
.penalty-item-left { flex: 1; }
.penalty-reason-badge {
  display: inline-block;
  font-size: 12px;
  font-weight: 500;
  color: #c2410c;
  background: #ffedd5;
  padding: 3px 8px;
  border-radius: 5px;
  margin-bottom: 4px;
}
.penalty-desc { font-size: 13px; color: #666; margin: 4px 0 2px; }
.penalty-date { font-size: 12px; color: #aaa; margin: 0; }
.penalty-delete {
  background: transparent;
  border: none;
  color: #ccc;
  cursor: pointer;
  padding: 2px;
  transition: color 0.15s;
}
.penalty-delete:hover { color: #ef4444; }

/* ── STATES ── */
.loading-state {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 32px;
  color: #aaa;
  font-size: 13px;
}
.empty-state {
  text-align: center;
  padding: 32px;
  color: #bbb;
  font-size: 13px;
}

/* ── SPINNER ── */
.spinner {
  width: 20px;
  height: 20px;
  border: 2px solid #e5e5e5;
  border-top-color: #888;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}
.spinner--sm {
  width: 16px;
  height: 16px;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* ── TRANSITIONS ── */
.slide-enter-active,
.slide-leave-active {
  transition: all 0.2s ease;
  overflow: hidden;
}
.slide-enter-from,
.slide-leave-to {
  opacity: 0;
  max-height: 0;
  padding-top: 0;
  padding-bottom: 0;
}
.slide-enter-to,
.slide-leave-from {
  opacity: 1;
  max-height: 600px;
}

.panel-enter-active,
.panel-leave-active {
  transition: all 0.25s ease;
}
.panel-enter-from,
.panel-leave-to {
  opacity: 0;
  transform: translateY(-8px);
}

/* ── RESPONSIVE ── */
@media (max-width: 600px) {
  .form-grid { grid-template-columns: 1fr; }
  .teacher-row { flex-direction: column; align-items: flex-start; }
  .teacher-actions { width: 100%; }
  .quick-nav { grid-template-columns: 1fr; }
  .stage-row { flex-wrap: wrap; }
}
</style>