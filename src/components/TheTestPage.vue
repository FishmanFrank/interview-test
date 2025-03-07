<script setup lang='ts'>
import { useStorage, useVirtualList } from '@vueuse/core'
import studentList from '@/assets/names.json'
import { computed, nextTick, ref, shallowRef } from 'vue'
import { useFuse, type UseFuseOptions } from '@vueuse/integrations/useFuse'

const students = useStorage('student-list', studentList)
const selectedStudentIds = ref<number[]>([])

const nameInputRef = ref<HTMLInputElement>()

const search = shallowRef('')
const options = computed<UseFuseOptions<typeof students['value'][number]>>(() => ({
  fuseOptions: {
    keys: ['name'],
    isCaseSensitive: false,
    threshold: undefined,
  },
  matchAllWhenSearchEmpty: true,
}))

const { results: filteredStudents } = useFuse(search, students, options)

const { list, containerProps, wrapperProps } = useVirtualList(
  filteredStudents.value,
  {
    itemHeight: 22,
  },
)

const totalStudents = computed(() => students.value.length)
const totalGraduates = computed(() => students.value.filter(student => student.graduated_at !== null).length)
const totalFees = computed(() => students.value.reduce((acc, curr) => acc += curr.fee, 0))

const formOpen = shallowRef(false)
const newStudent = ref<typeof students['value'][number]>({
  name: '',
  id: -1,
  graduated_at: null,
  fee: 0,
})

function formatFee(fee: number) {
  return `$${fee.toFixed(2)}`
}

function formatGraduation(graduationDate: string | null) {
  if (!graduationDate)
    return '-'
  return new Date(graduationDate).toISOString().slice(0, 10)
}

function markSelectedAsGraduated() {
  const currentStudentsList = students.value
  selectedStudentIds.value.forEach(selectedId => {
    const foundStudent = currentStudentsList.find(student => student.id === selectedId)
    if (foundStudent!.graduated_at)
      return
    foundStudent!.graduated_at = new Date().toISOString()
  })
}

function showAddStudentForm() {
  newStudent.value.id = Math.max(...students.value.map(s => s.id)) + 1
  formOpen.value = true

  nextTick(() => {
    nameInputRef.value?.focus()
  })
}

function addNewStudent() {
  students.value.push(newStudent.value)
  newStudent.value = {
    name: '',
    id: -1,
    graduated_at: null,
    fee: 0,
  }
  formOpen.value = false
}

function deleteStudent(id: number) {
  const studentIndexToDelete = students.value.findIndex(student => student.id === id)
  students.value.splice(studentIndexToDelete, 1)
}

function selectAll() {
  selectedStudentIds.value = students.value.map(student => student.id)
}
function deselectAll() {
  selectedStudentIds.value = []
}
</script>

<template>
  <div class="space-y-4">
    <h1>Students List</h1>
    <p>Manage and edit students status. For more information about student status, please email
      <a class="link" href="mailto:example@email.com">example@email.com</a>
      or call
      <a class="link" href="tel:+60111111111">+60111111111</a>
    </p>

    <div class="grid grid-cols-3 gap-4 px-4">
      <div class="stat-item">
        <div>Total Students</div>
        <div>{{ totalStudents }}</div>
      </div>
      <div class="stat-item">
        <div>Total Graduates</div>
        <div>{{ totalGraduates }}</div>
      </div>
      <div class="stat-item">
        <div>Total Fees</div>
        <div>{{ formatFee(totalFees) }}</div>
      </div>
    </div>

    <!-- Search/Actions -->
    <div class="flex items-center justify-between px-4 gap-2">
      <input v-model="search" />

      <div class="space-x-2">
        <!-- Add student form -->
        <div class="relative inline-block">
          <button type="button" @click="showAddStudentForm">Add Student</button>
          <form v-if="formOpen" class="absolute right-0 transition origin-top-right w-56 bg-indigo-200 p-4" :class="{
            'scale-0': !formOpen,
          }" @submit.prevent="addNewStudent">
            <div>ID:</div>
            <input disabled v-model="newStudent.id" />
            <div>Name:</div>
            <input ref="nameInputRef" v-model="newStudent.name" />
            <div>Fees:</div>
            <input v-model.number="newStudent.fee" />

            <div class="flex gap-2">
              <button type="button" @click="formOpen = false">Cancel</button>
              <button type="submit">Add New Student</button>
            </div>
          </form>
        </div>
        <button type="button" @click="markSelectedAsGraduated">Mark As Graduated</button>
      </div>
    </div>

    <!-- Selection -->
    <div class="flex items-center justify-start px-4 gap-2">
      <button type="button" @click="selectAll">Select All</button>
      <button type="button" @click="deselectAll">Deselect All</button>
    </div>

  </div>
  <div v-bind="containerProps" style="height: 300px">
    <div v-bind="wrapperProps">
      <div class="grid student-table grid-cols-6 px-4 gap-4">
        <div class="font-bold">Select</div>
        <div class="font-bold">ID</div>
        <div class="font-bold">Name</div>
        <div class="font-bold">Fee</div>
        <div class="font-bold">Graduated At</div>
        <div class="font-bold">Delete</div>
        <template v-for="item in list" :key="item.index">
          <div>
            <input v-model="selectedStudentIds" type="checkbox" :value="item.data.item.id" />
          </div>
          <div v-text="item.data.item.id" />
          <div v-text="item.data.item.name" />
          <div class="tabular-nums" v-text="formatFee(item.data.item.fee)" />
          <div v-text="formatGraduation(item.data.item.graduated_at)" />
          <button type="button" @click="deleteStudent(item.data.item.id)">Delete</button>
        </template>
      </div>
    </div>
  </div>
</template>

<style scoped>
.student-table>:nth-child(6n+4) {
  text-align: right;
}
</style>
