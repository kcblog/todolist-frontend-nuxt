<template>
  <div>
    <div class="flex justify-center shadow">
      <nuxt-link :class="[status === 'active' ? 'pill-active' : 'pill-inactive']" class="pill-default lg:flex-grow-0 no-underline" :to="{name: 'app-status', params: { status: 'active' }}" exact>
        Active ({{ activeTasks.length }})
      </nuxt-link>

      <nuxt-link :class="[status === 'all' ? 'pill-active' : 'pill-inactive']" class="pill-default lg:flex-grow-0 no-underline" :to="{name: 'app-status', params: { status: 'all' }}" exact>
        All ({{ allTasks.length }})
      </nuxt-link>

      <nuxt-link :class="[status === 'completed' ? 'pill-active' : 'pill-inactive']" class="pill-default lg:flex-grow-0 no-underline" :to="{name: 'app-status', params: { status: 'completed' }}" exact>
        Completed ({{ completedTasks.length }})
      </nuxt-link>
    </div>

    <div class="container mx-auto mt-3 px-4">
      <div v-if="false && isLoading" class="text-xl text-center my-6 text-gray-800">
        <fa icon="spinner" class="mr-1" spin />
        Loading
      </div>

      <transition-group class="relative" name="fade-out-left" tag="ul">
        <task v-for="task in tasks"
              :key="task.id"
              :task="task"
              class="animated task-item"
        />
      </transition-group>

      <new-task v-if="status != 'completed'" />

      <div v-else class="flex justify-end my-4">
        <loading-button
          v-if="completedTasks.length"
          :is-loading="isRemoveLoading"
          :class="[isRemoveLoading ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer hover:underline hover:text-red-500']"
          type="button"
          icon="trash"
          class="text-gray-800 text-sm"
          @click.native="deleteTasks"
        >
          Delete completed tasks
        </loading-button>
      </div>

      <div v-if="timeToChill" class="text-center mb-6">
        <p class="text-5xl">
          🍻
        </p>
        Time to chill ! You have no tasks
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import Task from '@/components/Tasks/Task'
import NewTask from '@/components/Tasks/NewTask'
import LoadingButton from '@/components/LoadingButton'
import TaskModel from '@/models/Task'

export default {
  components: {
    Task,
    NewTask,
    LoadingButton
  },

  head () {
    return {
      title: `${this.activeTasks.length} tasks`
    }
  },

  data () {
    return {
      isLoading: false,
      isRemoveLoading: false,
      isAnimated: true,
      initialTasks: []
    }
  },

  computed: {
    ...mapGetters({
      allTasks: 'entities/tasks/allTasks',
      activeTasks: 'entities/tasks/activeTasks',
      completedTasks: 'entities/tasks/completedTasks'
    }),

    tasks () {
      if (this.isAnimated) {
        return this.initialTasks
      }

      return this.filteredTasks
    },

    filteredTasks () {
      return TaskModel.getters('filteredTasks')(this.status)
    },

    status () {
      return this.$route.params.status
    },

    timeToChill () {
      if (this.isLoading) {
        return false
      }

      return TaskModel.getters('timeToChill')(this.status)
    }
  },

  mounted () {
    this.getTasks()

    if (window.Echo) {
      window.Echo.private(`App.User.${this.$auth.user.data.id}`)
        .listen('TaskCreated', e => TaskModel.insert({ data: e.task }))
        .listen('TaskUpdated', e => TaskModel.update({ where: e.task.id, data: e.task }))
        .listen('TaskDeleted', e => TaskModel.delete(e.task.id))
        .listen('TasksDeleted', e => TaskModel.delete(task => task.is_completed))
    }
  },

  methods: {
    getTasks () {
      this.isLoading = true

      TaskModel.$fetch()
        .then(() => {
          this.isLoading = false
          this.setInitialTasks([...this.filteredTasks])
        })
    },

    setInitialTasks (tasks) {
      if (!tasks.length) {
        this.isAnimated = false

        return false
      }

      let interval = null

      this.initialTasks.push(tasks.shift())

      interval = setInterval(() => {
        if (tasks.length > 0) {
          this.initialTasks.push(tasks.shift())
        } else {
          clearInterval(interval)
          this.isAnimated = false
        }
      }, 25)
    },

    deleteTasks () {
      if (this.isRemoveLoading || !window.confirm('Are you sure ? All your completed tasks will be deleted forever.')) {
        return false
      }

      this.isRemoveLoading = true

      TaskModel.dispatch('deleteTasks')
        .then(() => {
          this.isRemoveLoading = false
        })
    },

    tasksDeleted () {
      this.completedTasks.forEach(task => this.removeTask(task))
    }
  }

}
</script>

<style scoped>
.fade-out-left-leave-active.task-item {
  position: absolute;
  left: 0;
  right: 0;
  margin: 0;
}
</style>
