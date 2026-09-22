<script setup lang="ts">
import { computed, nextTick, ref, watch } from 'vue'
import {
  BookOpen,
  Check,
  ChevronRight,
  CirclePlus,
  Download,
  GripVertical,
  Menu,
  MoreHorizontal,
  Pencil,
  Plus,
  Search,
  Tags,
  Trash2,
  Upload,
  X,
} from '@lucide/vue'

type Question = {
  id: string
  title: string
  answer: string
  familiar: boolean
  updatedAt: number
}

type QuestionGroup = {
  id: string
  name: string
  questions: Question[]
}

type Term = {
  id: string
  name: string
  definition: string
  updatedAt: number
}

type View = 'questions' | 'terms'

type BackupData = {
  version: 1
  exportedAt: string
  groups: QuestionGroup[]
  terms: Term[]
}

const STORAGE_KEY = 'interview-notebook-groups'
const TERMS_STORAGE_KEY = 'interview-notebook-terms'
const makeId = () => crypto.randomUUID()

const sampleGroups: QuestionGroup[] = [
  {
    id: makeId(),
    name: 'JavaScript 基础',
    questions: [
      {
        id: makeId(),
        title: '闭包是什么？有哪些常见使用场景？',
        answer: '闭包是函数与其词法环境的组合。常用于封装私有变量、函数柯里化、回调和缓存等场景。',
        familiar: true,
        updatedAt: Date.now(),
      },
      {
        id: makeId(),
        title: '事件循环的执行顺序是什么？',
        answer: '同步任务先进入调用栈执行，随后清空微任务队列，再执行一个宏任务，循环往复。',
        familiar: false,
        updatedAt: Date.now() - 1000,
      },
    ],
  },
  {
    id: makeId(),
    name: 'Vue 3',
    questions: [
      {
        id: makeId(),
        title: 'ref 和 reactive 有什么区别？',
        answer: 'ref 可以持有任意类型，通过 .value 访问；reactive 用于对象并返回深层响应式代理。',
        familiar: false,
        updatedAt: Date.now() - 2000,
      },
    ],
  },
]

function loadGroups(): QuestionGroup[] {
  try {
    const saved = localStorage.getItem(STORAGE_KEY)
    return saved ? JSON.parse(saved) : sampleGroups
  } catch {
    return sampleGroups
  }
}

function loadTerms(): Term[] {
  try {
    const saved = localStorage.getItem(TERMS_STORAGE_KEY)
    return saved ? JSON.parse(saved) : []
  } catch {
    return []
  }
}

const groups = ref<QuestionGroup[]>(loadGroups())
const terms = ref<Term[]>(loadTerms())
const activeView = ref<View>('questions')
const selectedGroupId = ref(groups.value[0]?.id ?? '')
const query = ref('')
const showGroupForm = ref(false)
const newGroupName = ref('')
const editingGroupId = ref<string | null>(null)
const editingGroupName = ref('')
const groupInput = ref<HTMLInputElement | null>(null)
const sidebarOpen = ref(false)
const editorOpen = ref(false)
const editingQuestionId = ref<string | null>(null)
const editingQuestionGroupId = ref<string | null>(null)
const draftQuestionGroupId = ref('')
const draftTitle = ref('')
const draftAnswer = ref('')
const termEditorOpen = ref(false)
const editingTermId = ref<string | null>(null)
const draftTermName = ref('')
const draftTermDefinition = ref('')
const dragState = ref<{ kind: 'groups' | 'questions' | 'terms'; id: string } | null>(null)
const dragOverId = ref<string | null>(null)
const importInput = ref<HTMLInputElement | null>(null)

watch(
  groups,
  (value) => localStorage.setItem(STORAGE_KEY, JSON.stringify(value)),
  { deep: true },
)

watch(
  terms,
  (value) => localStorage.setItem(TERMS_STORAGE_KEY, JSON.stringify(value)),
  { deep: true },
)

const selectedGroup = computed(() =>
  groups.value.find((group) => group.id === selectedGroupId.value),
)

const visibleQuestions = computed(() => {
  const questions = selectedGroup.value?.questions ?? []
  const keyword = query.value.trim().toLowerCase()
  if (!keyword) return questions
  return questions.filter(
    (question) =>
      question.title.toLowerCase().includes(keyword) ||
      question.answer.toLowerCase().includes(keyword),
  )
})

const visibleTerms = computed(() => {
  const keyword = query.value.trim().toLowerCase()
  if (!keyword) return terms.value
  return terms.value.filter(
    (term) =>
      term.name.toLowerCase().includes(keyword) ||
      term.definition.toLowerCase().includes(keyword),
  )
})

const totalQuestions = computed(() =>
  groups.value.reduce((total, group) => total + group.questions.length, 0),
)

const familiarQuestions = computed(() =>
  groups.value.reduce(
    (total, group) => total + group.questions.filter((question) => question.familiar).length,
    0,
  ),
)

const progress = computed(() =>
  totalQuestions.value ? Math.round((familiarQuestions.value / totalQuestions.value) * 100) : 0,
)

async function startGroupForm() {
  showGroupForm.value = true
  await nextTick()
  groupInput.value?.focus()
}

function createGroup() {
  const name = newGroupName.value.trim()
  if (!name) return
  const group: QuestionGroup = { id: makeId(), name, questions: [] }
  groups.value.push(group)
  selectedGroupId.value = group.id
  newGroupName.value = ''
  showGroupForm.value = false
  sidebarOpen.value = false
}

function startRenameGroup(group: QuestionGroup) {
  editingGroupId.value = group.id
  editingGroupName.value = group.name
}

function saveGroupName(group: QuestionGroup) {
  const name = editingGroupName.value.trim()
  if (name) group.name = name
  editingGroupId.value = null
  editingGroupName.value = ''
}

function selectGroup(id: string) {
  activeView.value = 'questions'
  selectedGroupId.value = id
  query.value = ''
  sidebarOpen.value = false
}

function showTerms() {
  activeView.value = 'terms'
  query.value = ''
  sidebarOpen.value = false
}

function removeGroup(group: QuestionGroup) {
  if (!confirm(`确定删除“${group.name}”及其中的所有问题吗？`)) return
  const index = groups.value.findIndex((item) => item.id === group.id)
  groups.value.splice(index, 1)
  if (selectedGroupId.value === group.id) {
    selectedGroupId.value = groups.value[Math.max(0, index - 1)]?.id ?? ''
  }
}

function openCreateQuestion() {
  editingQuestionId.value = null
  editingQuestionGroupId.value = null
  draftQuestionGroupId.value = selectedGroupId.value
  draftTitle.value = ''
  draftAnswer.value = ''
  editorOpen.value = true
}

function openEditQuestion(question: Question) {
  editingQuestionId.value = question.id
  editingQuestionGroupId.value = selectedGroupId.value
  draftQuestionGroupId.value = selectedGroupId.value
  draftTitle.value = question.title
  draftAnswer.value = question.answer
  editorOpen.value = true
}

function closeEditorFromBackdrop() {
  if (draftTitle.value.trim() || draftAnswer.value.trim()) return
  editorOpen.value = false
}

function saveQuestion() {
  const title = draftTitle.value.trim()
  const targetGroup = groups.value.find((group) => group.id === draftQuestionGroupId.value)
  if (!title || !targetGroup) return

  if (editingQuestionId.value && editingQuestionGroupId.value) {
    const sourceGroup = groups.value.find((group) => group.id === editingQuestionGroupId.value)
    const questionIndex = sourceGroup?.questions.findIndex(
      (item) => item.id === editingQuestionId.value,
    ) ?? -1
    if (sourceGroup && questionIndex >= 0) {
      const question = sourceGroup.questions[questionIndex]
      question.title = title
      question.answer = draftAnswer.value.trim()
      question.updatedAt = Date.now()
      if (sourceGroup.id !== targetGroup.id) {
        sourceGroup.questions.splice(questionIndex, 1)
        targetGroup.questions.unshift(question)
      }
    }
  } else {
    targetGroup.questions.unshift({
      id: makeId(),
      title,
      answer: draftAnswer.value.trim(),
      familiar: false,
      updatedAt: Date.now(),
    })
  }
  editorOpen.value = false
}

function toggleFamiliar(question: Question) {
  question.familiar = !question.familiar
  question.updatedAt = Date.now()
}

function removeQuestion(question: Question) {
  if (!selectedGroup.value || !confirm('确定删除这个问题吗？')) return
  selectedGroup.value.questions = selectedGroup.value.questions.filter(
    (item) => item.id !== question.id,
  )
}

function openCreateTerm() {
  editingTermId.value = null
  draftTermName.value = ''
  draftTermDefinition.value = ''
  termEditorOpen.value = true
}

function openEditTerm(term: Term) {
  editingTermId.value = term.id
  draftTermName.value = term.name
  draftTermDefinition.value = term.definition
  termEditorOpen.value = true
}

function closeTermEditorFromBackdrop() {
  if (draftTermName.value.trim() || draftTermDefinition.value.trim()) return
  termEditorOpen.value = false
}

function saveTerm() {
  const name = draftTermName.value.trim()
  if (!name) return

  if (editingTermId.value) {
    const term = terms.value.find((item) => item.id === editingTermId.value)
    if (term) {
      term.name = name
      term.definition = draftTermDefinition.value.trim()
      term.updatedAt = Date.now()
    }
  } else {
    terms.value.unshift({
      id: makeId(),
      name,
      definition: draftTermDefinition.value.trim(),
      updatedAt: Date.now(),
    })
  }
  termEditorOpen.value = false
}

function removeTerm(term: Term) {
  if (!confirm(`确定删除“${term.name}”吗？`)) return
  terms.value = terms.value.filter((item) => item.id !== term.id)
}

function startDrag(kind: 'groups' | 'questions' | 'terms', id: string, event: DragEvent) {
  if (query.value.trim()) return
  dragState.value = { kind, id }
  event.dataTransfer?.setData('text/plain', id)
  if (event.dataTransfer) event.dataTransfer.effectAllowed = 'move'
}

function dragOver(kind: 'groups' | 'questions' | 'terms', id: string) {
  if (dragState.value?.kind === kind && dragState.value.id !== id) dragOverId.value = id
}

function reorderById<T extends { id: string }>(items: T[], sourceId: string, targetId: string) {
  const sourceIndex = items.findIndex((item) => item.id === sourceId)
  const targetIndex = items.findIndex((item) => item.id === targetId)
  if (sourceIndex < 0 || targetIndex < 0 || sourceIndex === targetIndex) return
  const [moved] = items.splice(sourceIndex, 1)
  items.splice(targetIndex, 0, moved)
}

function dropItem(kind: 'groups' | 'questions' | 'terms', targetId: string) {
  const source = dragState.value
  if (!source || source.kind !== kind) return endDrag()
  if (kind === 'groups') reorderById(groups.value, source.id, targetId)
  if (kind === 'questions' && selectedGroup.value) {
    reorderById(selectedGroup.value.questions, source.id, targetId)
  }
  if (kind === 'terms') reorderById(terms.value, source.id, targetId)
  endDrag()
}

function endDrag() {
  dragState.value = null
  dragOverId.value = null
}

function isQuestion(value: unknown): value is Question {
  if (!value || typeof value !== 'object') return false
  const question = value as Record<string, unknown>
  return (
    typeof question.id === 'string' &&
    typeof question.title === 'string' &&
    typeof question.answer === 'string' &&
    typeof question.familiar === 'boolean' &&
    typeof question.updatedAt === 'number'
  )
}

function isGroup(value: unknown): value is QuestionGroup {
  if (!value || typeof value !== 'object') return false
  const group = value as Record<string, unknown>
  return (
    typeof group.id === 'string' &&
    typeof group.name === 'string' &&
    Array.isArray(group.questions) &&
    group.questions.every(isQuestion)
  )
}

function isTerm(value: unknown): value is Term {
  if (!value || typeof value !== 'object') return false
  const term = value as Record<string, unknown>
  return (
    typeof term.id === 'string' &&
    typeof term.name === 'string' &&
    typeof term.definition === 'string' &&
    typeof term.updatedAt === 'number'
  )
}

function isBackupData(value: unknown): value is BackupData {
  if (!value || typeof value !== 'object') return false
  const backup = value as Record<string, unknown>
  return (
    backup.version === 1 &&
    typeof backup.exportedAt === 'string' &&
    Array.isArray(backup.groups) &&
    backup.groups.every(isGroup) &&
    Array.isArray(backup.terms) &&
    backup.terms.every(isTerm)
  )
}

function exportData() {
  const backup: BackupData = {
    version: 1,
    exportedAt: new Date().toISOString(),
    groups: groups.value,
    terms: terms.value,
  }
  const blob = new Blob([JSON.stringify(backup, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  const date = new Date().toISOString().slice(0, 10)
  link.href = url
  link.download = `问迹备份-${date}.json`
  link.click()
  URL.revokeObjectURL(url)
}

async function importData(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  input.value = ''
  if (!file) return

  try {
    const parsed: unknown = JSON.parse(await file.text())
    if (!isBackupData(parsed)) throw new Error('invalid backup')
    if (!confirm('导入将覆盖当前的所有问题和专有名词，确定继续吗？')) return

    groups.value = parsed.groups
    terms.value = parsed.terms
    selectedGroupId.value = parsed.groups[0]?.id ?? ''
    activeView.value = parsed.groups.length ? 'questions' : 'terms'
    query.value = ''
  } catch {
    alert('导入失败：请选择由问迹导出的有效 JSON 备份文件。')
  }
}
</script>

<template>
  <div class="app-shell">
    <aside class="sidebar" :class="{ open: sidebarOpen }">
      <div class="brand">
        <div class="brand-mark"><BookOpen :size="19" stroke-width="2.4" /></div>
        <div>
          <strong>问迹</strong>
          <span>INTERVIEW NOTES</span>
        </div>
        <button class="icon-button sidebar-close" title="关闭菜单" @click="sidebarOpen = false">
          <X :size="19" />
        </button>
      </div>

      <button
        class="terms-nav-button"
        :class="{ active: activeView === 'terms' }"
        @click="showTerms"
      >
        <Tags :size="17" />
        <span>专有名词表</span>
        <span class="group-count">{{ terms.length }}</span>
        <ChevronRight :size="15" />
      </button>

      <div class="sidebar-section-header">
        <span>问题组</span>
        <button class="icon-button" title="创建问题组" @click="startGroupForm">
          <Plus :size="18" />
        </button>
      </div>

      <form v-if="showGroupForm" class="group-form" @submit.prevent="createGroup">
        <input
          ref="groupInput"
          v-model="newGroupName"
          placeholder="问题组名称"
          maxlength="30"
          @keydown.esc="showGroupForm = false"
        />
        <button type="submit" title="保存问题组"><Check :size="17" /></button>
      </form>

      <nav class="group-list" aria-label="问题组列表">
        <div
          v-for="group in groups"
          :key="group.id"
          class="group-item"
          :class="{
            active: activeView === 'questions' && group.id === selectedGroupId,
            'drag-over': dragOverId === group.id,
            dragging: dragState?.id === group.id,
          }"
          :draggable="!query && editingGroupId !== group.id"
          @dragstart="startDrag('groups', group.id, $event)"
          @dragover.prevent="dragOver('groups', group.id)"
          @drop.prevent="dropItem('groups', group.id)"
          @dragend="endDrag"
        >
          <GripVertical class="drag-handle" :size="15" aria-hidden="true" />
          <form v-if="editingGroupId === group.id" class="group-rename" @submit.prevent="saveGroupName(group)">
            <input v-model="editingGroupName" autofocus maxlength="30" @keydown.esc="editingGroupId = null" />
            <button type="submit" title="保存名称"><Check :size="15" /></button>
          </form>
          <template v-else>
            <button class="group-select" @click="selectGroup(group.id)">
              <span class="group-dot"></span>
              <span class="group-name">{{ group.name }}</span>
              <span class="group-count">{{ group.questions.length }}</span>
              <ChevronRight :size="15" class="chevron" />
            </button>
            <button class="group-edit" title="修改问题组名称" @click="startRenameGroup(group)">
              <Pencil :size="14" />
            </button>
            <button class="group-delete" title="删除问题组" @click="removeGroup(group)">
              <Trash2 :size="15" />
            </button>
          </template>
        </div>
      </nav>

      <button v-if="!groups.length" class="empty-group-button" @click="startGroupForm">
        <CirclePlus :size="18" /> 创建第一个问题组
      </button>

      <div class="progress-card">
        <div class="progress-copy">
          <span>总体熟悉度</span>
          <strong>{{ progress }}%</strong>
        </div>
        <div class="progress-track"><span :style="{ width: `${progress}%` }"></span></div>
        <p>{{ familiarQuestions }} / {{ totalQuestions }} 个问题已熟悉</p>
      </div>
    </aside>

    <div v-if="sidebarOpen" class="sidebar-backdrop" @click="sidebarOpen = false"></div>

    <main class="main-content">
      <header class="topbar">
        <button class="icon-button menu-button" title="打开菜单" @click="sidebarOpen = true">
          <Menu :size="21" />
        </button>
        <div class="search-box">
          <Search :size="18" />
          <input
            v-model="query"
            :placeholder="activeView === 'terms' ? '搜索专有名词或解释' : '搜索当前问题或答案'"
          />
          <kbd>⌘ K</kbd>
        </div>
        <div class="topbar-actions">
          <input
            ref="importInput"
            class="file-input"
            type="file"
            accept="application/json,.json"
            @change="importData"
          />
          <button class="icon-button" title="导入数据" aria-label="导入数据" @click="importInput?.click()">
            <Upload :size="17" />
          </button>
          <button class="icon-button" title="导出数据" aria-label="导出数据" @click="exportData">
            <Download :size="17" />
          </button>
          <div class="topbar-note">数据已保存在本机</div>
        </div>
      </header>

      <section v-if="activeView === 'terms'" class="workspace">
        <div class="workspace-header">
          <div>
            <p class="eyebrow">术语库</p>
            <h1>专有名词表</h1>
            <p class="group-summary">共 {{ terms.length }} 个名词，集中记录缩写、概念与技术术语</p>
          </div>
          <button class="primary-button" @click="openCreateTerm">
            <Plus :size="18" stroke-width="2.5" /> 新增名词
          </button>
        </div>

        <div v-if="visibleTerms.length" class="term-grid">
          <article v-for="term in visibleTerms" :key="term.id" class="term-card">
            <div class="term-heading">
              <div class="term-mark">{{ term.name.slice(0, 1).toUpperCase() }}</div>
              <h2>{{ term.name }}</h2>
              <div class="term-actions">
                <button class="icon-button" title="编辑名词" @click="openEditTerm(term)">
                  <Pencil :size="17" />
                </button>
                <button class="icon-button danger" title="删除名词" @click="removeTerm(term)">
                  <Trash2 :size="17" />
                </button>
              </div>
            </div>
            <p v-if="term.definition" class="term-definition">{{ term.definition }}</p>
            <button v-else class="add-answer term-add-definition" @click="openEditTerm(term)">
              <Plus :size="15" /> 添加解释
            </button>
          </article>
        </div>

        <div v-else class="empty-state">
          <div class="empty-icon"><Tags :size="24" /></div>
          <h2>{{ query ? '没有找到匹配的名词' : '专有名词表还是空的' }}</h2>
          <p>{{ query ? '换一个关键词试试。' : '记录缩写和技术概念，查阅时更方便。' }}</p>
          <button v-if="!query" class="secondary-button" @click="openCreateTerm">
            <Plus :size="17" /> 新增名词
          </button>
        </div>
      </section>

      <section v-else-if="selectedGroup" class="workspace">
        <div class="workspace-header">
          <div>
            <p class="eyebrow">问题组</p>
            <h1>{{ selectedGroup.name }}</h1>
            <p class="group-summary">
              共 {{ selectedGroup.questions.length }} 个问题 ·
              {{ selectedGroup.questions.filter((item) => item.familiar).length }} 个已熟悉
            </p>
          </div>
          <button class="primary-button" @click="openCreateQuestion">
            <Plus :size="18" stroke-width="2.5" /> 新建问题
          </button>
        </div>

        <div v-if="visibleQuestions.length" class="question-list">
          <article
            v-for="(question, index) in visibleQuestions"
            :key="question.id"
            class="question-card"
            :class="{
              familiar: question.familiar,
              'drag-over': dragOverId === question.id,
              dragging: dragState?.kind === 'questions' && dragState.id === question.id,
            }"
            @dragover.prevent="dragOver('questions', question.id)"
            @drop.prevent="dropItem('questions', question.id)"
          >
            <div
              class="drag-handle question-drag-handle"
              :draggable="!query"
              :title="query ? '清空搜索后可排序' : '拖拽调整顺序'"
              role="button"
              aria-label="拖拽调整顺序"
              @dragstart="startDrag('questions', question.id, $event)"
              @dragend="endDrag"
            >
              <GripVertical :size="17" aria-hidden="true" />
            </div>
            <button
              class="check-button"
              :class="{ checked: question.familiar }"
              :title="question.familiar ? '取消熟悉' : '标记为熟悉'"
              :aria-label="question.familiar ? '取消熟悉' : '标记为熟悉'"
              @click="toggleFamiliar(question)"
            >
              <Check :size="17" stroke-width="3" />
            </button>

            <div class="question-body">
              <div class="question-title-row">
                <span class="question-number">{{ String(index + 1).padStart(2, '0') }}</span>
                <h2>{{ question.title }}</h2>
              </div>
              <p v-if="question.answer" class="answer">{{ question.answer }}</p>
              <button v-else class="add-answer" @click="openEditQuestion(question)">
                <Plus :size="15" /> 添加答案
              </button>
            </div>

            <div class="question-actions">
              <span v-if="question.familiar" class="status-label">已熟悉</span>
              <button class="icon-button" title="编辑问题" @click="openEditQuestion(question)">
                <Pencil :size="17" />
              </button>
              <button class="icon-button danger" title="删除问题" @click="removeQuestion(question)">
                <Trash2 :size="17" />
              </button>
            </div>
          </article>
        </div>

        <div v-else class="empty-state">
          <div class="empty-icon"><MoreHorizontal :size="25" /></div>
          <h2>{{ query ? '没有找到匹配的问题' : '这个问题组还是空的' }}</h2>
          <p>{{ query ? '换一个关键词试试。' : '记录第一个面试问题，开始构建你的知识库。' }}</p>
          <button v-if="!query" class="secondary-button" @click="openCreateQuestion">
            <Plus :size="17" /> 新建问题
          </button>
        </div>
      </section>

      <section v-else class="welcome-state">
        <div class="welcome-mark"><BookOpen :size="28" /></div>
        <h1>创建一个问题组</h1>
        <p>按技术方向或岗位整理问题，复习会更清晰。</p>
        <button class="primary-button" @click="startGroupForm(); sidebarOpen = true">
          <Plus :size="18" /> 创建问题组
        </button>
      </section>
    </main>

    <Teleport to="body">
      <div v-if="termEditorOpen" class="modal-backdrop" @click.self="closeTermEditorFromBackdrop">
        <form class="editor-modal" @submit.prevent="saveTerm">
          <div class="modal-header">
            <div>
              <p class="eyebrow">{{ editingTermId ? '编辑术语' : '新增术语' }}</p>
              <h2>{{ editingTermId ? '编辑专有名词' : '记录专有名词' }}</h2>
            </div>
            <button type="button" class="icon-button" title="关闭" @click="termEditorOpen = false">
              <X :size="20" />
            </button>
          </div>

          <label class="field">
            <span>名词</span>
            <input
              v-model="draftTermName"
              autofocus
              required
              maxlength="100"
              placeholder="例如：CSR、事件委托、幂等性"
            />
          </label>

          <label class="field">
            <span>解释</span>
            <textarea
              v-model="draftTermDefinition"
              rows="7"
              placeholder="记录它的完整含义、作用或使用场景……"
            ></textarea>
          </label>

          <div class="modal-footer">
            <button type="button" class="text-button" @click="termEditorOpen = false">取消</button>
            <button type="submit" class="primary-button" :disabled="!draftTermName.trim()">
              保存名词
            </button>
          </div>
        </form>
      </div>

      <div v-if="editorOpen" class="modal-backdrop" @click.self="closeEditorFromBackdrop">
        <form class="editor-modal" @submit.prevent="saveQuestion">
          <div class="modal-header">
            <div>
              <p class="eyebrow">{{ editingQuestionId ? '编辑记录' : '新建记录' }}</p>
              <h2>{{ editingQuestionId ? '编辑问题' : '记录一个新问题' }}</h2>
            </div>
            <button type="button" class="icon-button" title="关闭" @click="editorOpen = false">
              <X :size="20" />
            </button>
          </div>

          <label class="field">
            <span>问题</span>
            <input
              v-model="draftTitle"
              autofocus
              required
              maxlength="200"
              placeholder="例如：浏览器的事件循环是如何工作的？"
            />
          </label>

          <label class="field">
            <span>所属问题组</span>
            <select v-model="draftQuestionGroupId" required>
              <option v-for="group in groups" :key="group.id" :value="group.id">
                {{ group.name }}
              </option>
            </select>
          </label>

          <label class="field">
            <span>答案</span>
            <textarea
              v-model="draftAnswer"
              rows="9"
              placeholder="写下你的理解、关键点或示例……"
            ></textarea>
          </label>

          <div class="modal-footer">
            <button type="button" class="text-button" @click="editorOpen = false">取消</button>
            <button type="submit" class="primary-button" :disabled="!draftTitle.trim()">
              保存问题
            </button>
          </div>
        </form>
      </div>
    </Teleport>
  </div>
</template>
