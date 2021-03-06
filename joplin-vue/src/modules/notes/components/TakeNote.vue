<template>
  <form method="post" class="form-horizontal" @submit.prevent="doNote">
    <b-alert v-if="updated == 1" v-model="updated" variant="success" show>Update done</b-alert>
    <b-alert v-if="updated == 0" v-model="updated" variant="danger">Update failed</b-alert>
    <p v-if="errors.length">
      <b>Please correct the following error(s):</b>
      <b-alert variant="danger" v-for="error in errors" :key="error" show>{{ error }}</b-alert>
    </p>
    <div class="input-group input-group-sm mb-3">
      <b-form-input v-model="title"
                    type="text"
                    name="title"
                    id="title"
                    placeholder="Enter your title">
      </b-form-input>
      <div class="input-group-append">
        <b-button v-if="id" size="sm" variant="danger" @click="removeNote(id)"><i class="fas fa-trash"></i> Delete this note ?
        </b-button>
      </div>
    </div>
    <div class="form-group row">
      <div class="col-sm-2">
      <label v-if="is_todo==1" class="btn btn-secondary btn-sm active"><i class="fas fa-tasks"></i> Tasks ?
        <input name="is_todo" id="is_todo" v-model="is_todo" type="checkbox" checked autocomplete="off"/>
      </label>
      <label v-if="is_todo==0" class="btn btn-secondary btn-sm"><i class="fas fa-tasks"></i> Tasks ?
        <input name="is_todo" id="is_todo" v-model="is_todo" type="checkbox" autocomplete="on"/>
      </label>
      </div>
      <div class="col-sm-6">
        <b-form-input v-model="tag"
                      type="text"
                      name="tag"
                      id="tag"
                      placeholder="tags, tags">
        </b-form-input>
      </div>
      <div class="col-sm-4">
        <div class="input-group-prepend">
          <span class="text-muted">Created: {{ moment(created_time).format('lll') }}</span>&nbsp;
          <b-btn id="note-info" size="sm" variant="secondary"><i class="fas fa-info-circle"></i></b-btn>
          <b-popover target="note-info" triggers="hover focus">
            <template slot="title">Note details</template>
            <ul>
              <li v-if="author !== ''">Author: {{ author }}</li>
              <li v-else>Author n/a</li>
              <li v-if="source_url !== ''">URL: {{ source_url }}</li>
              <li v-else>URL n/a</li>
              <li>Date
                <ul>
                  <li v-if="created_time > 0">Created: {{ moment(created_time).format('lll') }}</li>
                  <li v-if="updated_time > 0">Updated: {{ moment(updated_time).format('lll') }}</li>
                </ul>
              </li>
              <li>Geo location
                <ul>
                <li>Latitude {{ latitude }}</li>
                <li>Longitude {{ longitude }}</li>
                <li>Altitude {{ altitude }}</li>
                </ul>
              </li>
              <li>Tasks
                <ul>
                  <li v-if="todo_completed > 0">Todo Completed: {{ moment(todo_completed).format('lll') }}</li>
                  <li v-else>Todo Competed: n/a</li>
                  <li v-if="todo_due > 0">Todo Due: {{ moment(todo_due).format('lll') }}</li>
                  <li v-else>Todo Due: n/a</li>
                </ul>
              </li>
              <li>Source
                <ul>
                  <li>Source: {{ source }}</li>
                  <li>Source Application: {{ source_application }}</li>
                  <li>ID: {{ id }}</li>
                </ul>
              </li>
            </ul>
          </b-popover>
        </div>
      </div>
    </div>
    <div class="form-group">
      <div class="input-group input-group-sm mb-3">
        <div class="input-group-prepend">
          <label class="input-group-text" for="inputGroupSelect01"><i class="fas fa-folder-open"></i> Folder</label>
          <treeselect v-model="parent_id"
                :multiple="false"
                :disable-branch-nodes="false"
                :options="this.getTreeFolders"
                ref="treeselect"/>
        </div>
      </div>
    </div>
    <div class="form-group row">
      <div class="col-sm-6">
        <b-form-textarea id="body"
                         v-model="body"
                         placeholder="Enter something"
                         :rows="25"
                         :max-rows="40"
                         @input="updateBody">
        </b-form-textarea>
        <b-alert v-if="updated == 1" v-model="updated" variant="success" show>Update done</b-alert>
        <b-alert v-if="updated == 0" v-model="updated" variant="danger">Update failed</b-alert>
        <button class="btn btn-primary">
          <span><i class="fas fa-save"></i> Save</span>
        </button>
      </div>
      <div class="col-sm-6">
        <div v-html="compiledMarkdown"></div>
      </div>
    </div>
  </form>
</template>

<script>
import { createNamespacedHelpers } from 'vuex'

import getters from '../getters'
import actions from '../actions'
import types from '../types'

import { createHelpers } from 'vuex-map-fields'

// import the component
import Treeselect from '@riophae/vue-treeselect'
// import the styles
import '@riophae/vue-treeselect/dist/vue-treeselect.css'

import _ from 'lodash'
// required because marked does not deal with sanitze at all
const createDOMPurify = require('dompurify')
const { JSDOM } = require('jsdom')

const window = (new JSDOM('')).window
const DOMPurify = createDOMPurify(window)
// required because marked does not deal with sanitze at all
let marked = require('marked')

const namespace = 'notes'
const { mapGetters, mapActions } = createNamespacedHelpers(namespace)

const { mapFields } = createHelpers({
  getterType: 'getNoteField',
  mutationType: 'updateNoteField'
})

export default {

  data () {
    return {
      urlResources: '/files',
      updated: -1,
      folders: [],
      errors: []
    }
  },
  components: { Treeselect },
  methods: {
    doNote (e) {
      this.errors = []

      if (this.title === '') {
        this.errors.push('title required.')
      }
      if (this.parent_id === undefined) {
        this.errors.push('folder required.')
      }
      if (this.body === '') {
        this.errors.push('body required.')
      }

      if (!this.errors.length) {
        if (this.id === undefined || this.id === 0) {
          this.addNote()
        } else {
          this.updateNote()
        }
        return true
      }

      e.preventDefault()
    },
    /* create a note */
    addNote () {
      let payload = {
        'title': this.title,
        'body': this.body,
        'parent_id': this.parent_id,
        'is_todo': (this.is_todo ? 1 : 0),
        'tag': this.tag
      }
      this.$store.dispatch('notes/' + types.NOTE_CREATE, payload)
    },
    /* update the note */
    updateNote () {
      // payload
      let payload = {
        'id': this.id,
        'title': this.title,
        'body': this.body,
        'parent_id': this.parent_id,
        'tag': this.tag,
        'is_todo': (this.is_todo ? 1 : 0)
      }
      // push the data
      this.$store.dispatch('notes/' + types.NOTE_CHANGE, payload)
        .then((res) => {
          this.updated = 1
        })
        // eslint-disable-next-line
        .catch((error) => {
          this.updated = 0
        })
    },
    /* delete action pressed */
    removeNote (id) {
      this.$store.dispatch('notes/' + types.NOTE_REMOVE, id)
    },
    // translate markdown to html
    updateBody: _.debounce(function (e) {
      // spot image markdown
      // eslint-disable-next-line
      // let re = /\!\[(.*)\.(\w+)\]\(:\/(.*)\)/g
      // eslint-disable-next-line
      let re = /\!\[(.*)\]\(:\/(.*)\)/g
      // image markdown : ![image name.extension](:/resource_id)
      // becomes
      // image markdown : ![image name.extension](http://127.0.0.1:8001/static/resource_id.extension)
      // add the URL to access to the image from the back http service
      // this.body = e.replace(re, '![$1.$2](' + this.urlResources + '/$3.$2)')

      this.body = e.replace(re, '![$1](' + this.urlResources + '/$2)')
    }, 300),
    ...mapActions(Object.keys(actions))
  },
  // load the data of the store into the form for each input
  computed: {
    compiledMarkdown: function () {
      let body = ''
      if (this.$store.state.notes.note.body !== undefined) {
        body = this.$store.state.notes.note.body
      }
      let re = /!\[(.*)\.(\w+)\]\(:\/(.*)\)/g
      body = body.replace(re, '![$1.$2](' + this.urlResources + '/$3.$2)')
      let cleanBody = DOMPurify.sanitize(body)
      return marked(cleanBody)
    },
    ...mapGetters(Object.keys(getters)),
    ...mapFields('notes', {
      id: 'note.id',
      title: 'note.title',
      body: 'note.body',
      is_todo: 'note.is_todo',
      parent_id: 'note.parent_id',
      created_time: 'note.user_created_time',
      updated_time: 'note.updated_time',
      todo_completed: 'note.todo_completed',
      todo_due: 'note.todo_due',
      source_url: 'note.source_url',
      author: 'note.author',
      latitude: 'note.latitude',
      longitude: 'note.longitude',
      altitude: 'note.altitude',
      source: 'note.source',
      source_application: 'note.source_application',
      tag: 'tag'
    })
  },
  mounted () {
    // trick : put the default value of the selected folder id by using $ref
    // defined as property in the form :P
    if (this.parent_id !== undefined || this.parent_id !== 0) {
      this.$refs.treeselect.$emit('select', this.parent_id)
    }
  }
}
</script>
