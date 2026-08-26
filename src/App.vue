<script setup lang="ts">
import { computed, nextTick, ref, watch } from 'vue'
import {
  BookOpen,
  Check,
  ChevronRight,
  CirclePlus,
  Menu,
  MoreHorizontal,
  Pencil,
  Plus,
  Search,
  Trash2,
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

const STORAGE_KEY = 'interview-notebook-groups'
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

const groups = ref<QuestionGroup[]>(loadGroups())
const selectedGroupId = ref(groups.value[0]?.id ?? '')
const query = ref('')
const showGroupForm = ref(false)
const newGroupName = ref('')
const groupInput = ref<HTMLInputElement | null>(null)
const sidebarOpen = ref(false)
const editorOpen = ref(false)
const editingQuestionId = ref<string | null>(null)
const draftTitle = ref('')
const draftAnswer = ref('')

watch(
  groups,
  (value) => localStorage.setItem(STORAGE_KEY, JSON.stringify(value)),
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

function selectGroup(id: string) {
  selectedGroupId.value = id
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
  draftTitle.value = ''
  draftAnswer.value = ''
  editorOpen.value = true
}

function openEditQuestion(question: Question) {
  editingQuestionId.value = question.id
  draftTitle.value = question.title
  draftAnswer.value = question.answer
  editorOpen.value = true
}

function saveQuestion() {
  const title = draftTitle.value.trim()
  if (!title || !selectedGroup.value) return

  if (editingQuestionId.value) {
    const question = selectedGroup.value.questions.find(
      (item) => item.id === editingQuestionId.value,
    )
    if (question) {
      question.title = title
      question.answer = draftAnswer.value.trim()
      question.updatedAt = Date.now()
    }
  } else {
    selectedGroup.value.questions.unshift({
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
          :class="{ active: group.id === selectedGroupId }"
        >
          <button class="group-select" @click="selectGroup(group.id)">
            <span class="group-dot"></span>
            <span class="group-name">{{ group.name }}</span>
            <span class="group-count">{{ group.questions.length }}</span>
            <ChevronRight :size="15" class="chevron" />
          </button>
          <button class="group-delete" title="删除问题组" @click="removeGroup(group)">
            <Trash2 :size="15" />
          </button>
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
          <input v-model="query" placeholder="搜索当前问题或答案" />
          <kbd>⌘ K</kbd>
        </div>
        <div class="topbar-note">数据已保存在本机</div>
      </header>

      <section v-if="selectedGroup" class="workspace">
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
            :class="{ familiar: question.familiar }"
          >
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
      <div v-if="editorOpen" class="modal-backdrop" @click.self="editorOpen = false">
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
