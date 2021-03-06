<template>
  <div id="edit-weakaura">
    <div class="flex-container">
      <div class="flex-col flex-left">
        <md-input-container>
          <label for="customfn" v-html="$t('Edit [-file-]', {file: editorFile})"></label>
          <md-select name="customfn" id="customfn" v-model="editorSelected" md-menu-class="customFn">
            <md-option value="tabledata" >{{ $t("Table data") }}</md-option>
            <template v-for="fn in customFn(tableData)">
              <md-subheader v-if="typeof fn === 'string'">{{ fn }}</md-subheader>
              <md-option v-else :value="fn.id.replace(/,/g, '')+','+fn.path">{{ fn.name }}</md-option>
            </template>
            <md-subheader v-if="customFn(tableData).length === 0">{{ $t("No custom functions found") }}</md-subheader>
          </md-select>
        </md-input-container>
      </div>
      <div class="flex-col flex-right">
        <md-menu v-if="groupedWA" md-size="6" md-align-trigger>
          <md-button md-menu-trigger id="extractFromGroupButton"><md-icon>call_missed_outgoing</md-icon> {{ $t("Extract from group") }}</md-button>
          <md-menu-content>
            <md-menu-item v-for="(wa, key) in groupedWA" @click="extractWA(wa)" :key="key">{{ wa }}</md-menu-item>
          </md-menu-content>
        </md-menu>        
        <md-button @click="exportChanges"><md-icon>open_in_new</md-icon> {{ $t("Export/Fork changes") }}</md-button>
        <md-button v-if="canEdit" @click="saveChanges"><md-icon>save</md-icon> {{ $t("Save changes") }}</md-button>
      </div>
    </div>

    <editor v-model="editorContent" @init="editorInit" :lang="aceLanguage" :theme="editorTheme" width="100%" height="500"></editor>

    <md-dialog md-open-from="#extractFromGroupButton" md-close-to="#extractFromGroupButton" ref="extractFromGroupDialog">
      <md-dialog-title>{{ $t("Extract from group") }}</md-dialog-title>

      <md-dialog-content>
        <p v-if="extractData">{{ $t("Copy the WeakAura import string for \"[-groupedWA-]\"", {groupedWA: extractData.name}) }}</p>
        <textarea v-if="extractData" class="wago-importstring" spellcheck="false" id="wago-extractString">{{ extractData.encoded }}</textarea>
        <p v-else>{{ $t("Loading extraction string") }}</p>
      </md-dialog-content>

      <md-dialog-actions>
        <md-button v-if="extractData" class="md-primary" @click="extractCopy()">{{ $t("Copy string") }}</md-button>
        <md-button class="md-primary" @click="this.$refs.extractFromGroupDialog.close()">{{ $t("Close") }}</md-button>
      </md-dialog-actions>
    </md-dialog>
    <export-modal :json="tableString" :type="'WeakAura'" :showExport="showExport" :wagoID="wago._id" @hideExport="hideExport"></export-modal>
  </div>
</template>

<script>
export default {
  name: 'edit-weakaura',
  data: function () {
    return {
      editorSelected: 'tabledata',
      editorPrevious: 'tabledata',
      editorPreviousObj: {},
      tableData: JSON.parse(this.$store.state.wago.code.json),
      tableString: this.$store.state.wago.code.json,
      editorFile: '',
      aceLanguage: 'json',
      aceEditor: null,
      showExport: false,
      extractData: false
    }
  },
  watch: {
    editorSelected: function (fn) {
      console.log(fn)
      if (fn !== 'tabledata') {
        fn = fn.split(',')
        var custFns = this.customFn(this.tableData)
        for (var i = 0; i < custFns.length; i++) {
          if (typeof custFns[i] === 'object' && fn[0] === custFns[i].id.replace(/,/g, '') && fn[1] === custFns[i].path) {
            fn = custFns[i]
            break
          }
        }
      }
      console.log(fn)
      // save current data to json object
      try {
        /* eslint-disable no-unused-vars */
        /* eslint-disable no-eval */
        var root
        console.log('set editor')

        // if switching FROM table data TO a custom Fn
        if (this.editorPrevious === 'tabledata') {
          this.$store.commit('setWagoJSON', this.aceEditor.getValue())
          this.tableData = JSON.parse(this.aceEditor.getValue())

          // switch to lua
          this.aceEditor.getSession().setMode('ace/mode/lua')

          if (fn.ix.table === 'd') {
            root = JSON.parse(this.editorContent).d
            this.aceEditor.setValue(eval('root.' + fn.path), -1)
            console.log('set1')
          }
          else if (fn.ix.table === 'c') {
            root = JSON.parse(this.editorContent).c[fn.ix.index]
            this.aceEditor.setValue(eval('root.' + fn.path), -1)
            console.log('set2')
          }

          this.editorFile = root.id
        }
        // if switching FROM a custom function
        else {
          var updated = this.aceEditor.getValue().replace(/\\/g, '\\\\').replace(/\r\n|\n|\r/g, '\\n').replace(/"/g, '\\"')

          // update table data
          if (this.editorPreviousObj.ix.table === 'd') {
            eval('this.tableData.d.' + this.editorPreviousObj.path + ' = "' + updated + '"')
          }
          else if (this.editorPreviousObj.ix.table === 'c') {
            eval('this.tableData.c[' + this.editorPreviousObj.ix.index + '].' + this.editorPreviousObj.path + ' = "' + updated + '"')
          }

          var json = JSON.stringify(this.tableData, null, 2)
          this.$store.commit('setWagoJSON', json)
          var vue = this

          setTimeout(function () {
            // if switching TO table data
            if (fn === 'tabledata') {
              vue.aceEditor.setValue(json, -1)
              console.log('set3')
              vue.aceEditor.getSession().setMode('ace/mode/json')
              vue.editorFile = vue.tableData.d.id
            }
            // if we are switching TO a custom function
            else if (typeof fn === 'object') {
              vue.aceEditor.getSession().setMode('ace/mode/lua')
              if (fn.ix.table === 'd') {
                root = vue.tableData.d
                vue.aceEditor.setValue(eval('root.' + fn.path), -1)
                console.log('set4', eval('root.' + fn.path))
              }
              else if (fn.ix.table === 'c') {
                root = vue.tableData.c[fn.ix.index]
                vue.aceEditor.setValue(eval('root.' + fn.path), -1)
                console.log('set5')
              }
              vue.editorFile = root.id
            }
          }, 100)
        }
      }
      catch (e) {
        window.eventHub.$emit('showSnackBar', this.$t('error:An error occurred reading the table data'))
      }

      // set previous to what is set NOW, for next time
      if (fn === 'tabledata') {
        this.editorPrevious = fn
      }
      else {
        this.editorPrevious = 'fn'
        this.editorPreviousObj = fn
      }
    }
  },
  components: {
    editor: require('vue2-ace-editor'),
    'export-modal': require('./ExportJSON.vue')
  },
  methods: {
    editorInit: function (editor) {
      this.editorFile = this.tableData.d.id
      this.aceEditor = editor
      window.braceRequires()
      editor.setOptions({
        autoScrollEditorIntoView: true,
        scrollPastEnd: true,
        printMargin: false,
        minLines: 80,
        maxLines: 1000
      })
    },
    customFn: function (json) {
      // if single aura then place in array
      if (!this.tableData.d) {
        return []
      }
      var auras = this.tableData.c || [this.tableData.d]
      if (typeof auras === 'object') {
        auras = Object.values(auras)
      }
      var table = this.tableData.c && 'c' || 'd'
      var func = []
      // loop through each aura in array and look for pre-defined custom functions
      auras.forEach((item, key) => {
        var ix = {index: key, table: table}

        // actions functions
        if (item.actions) {
          // onInit
          if (item.actions.init && item.actions.init.do_custom) {
            func.push(item.id)
            func.push({id: item.id, name: this.$t('onInit'), ix: ix, path: 'actions.init.custom'})
          }

          // onShow
          if (item.actions.start && item.actions.start.do_custom) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }
            func.push({id: item.id, name: this.$t('onShow'), ix: ix, path: 'actions.start.custom'})
          }

          // onHide
          if (item.actions.finish && item.actions.finish.do_custom) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }
            func.push({id: item.id, name: this.$t('onHide'), ix: ix, path: 'actions.finish.custom'})
          }
        }

        // display text
        if (((typeof item.displayText === 'string' && item.displayText.indexOf('%c') > -1) ||
        (typeof item.text1 === 'string' && item.text1.indexOf('%c') > -1) ||
        (typeof item.text2 === 'string' && item.text2.indexOf('%c') > -1) ||
        (typeof item.displayTextLeft === 'string' && item.displayTextLeft.indexOf('%c') > -1) ||
        (typeof item.displayTextRight === 'string' && item.displayTextRight.indexOf('%c') > -1)) &&
        item.customText) {
          if (func.indexOf(item.id) < 0) {
            func.push(item.id)
          }
          func.push({ id: item.id, name: this.$t('DisplayText'), ix: ix, path: 'customText' })
        }

        // display stacks
        else if (typeof item.displayStacks === 'string' && item.displayStacks.indexOf('%c') > -1) {
          if (func.indexOf(item.id) < 0) {
            func.push(item.id)
          }
          func.push({ id: item.id, name: this.$t('DisplayStacks'), ix: ix, path: 'customText' })
        }

        // primary trigger
        if (item.trigger && item.trigger.type === 'custom' && item.trigger.custom) {
          if (func.indexOf(item.id) < 0) {
            func.push(item.id)
          }

          func.push({ id: item.id, name: this.$t('Trigger ([-count-])', {count: 1}), ix: ix, path: 'trigger.custom' })

          // main untrigger
          if (item.untrigger && item.untrigger.custom && item.untrigger.custom.length > 0) {
            func.push({ id: item.id, name: this.$t('Untrigger ([-count-])', {count: 1}), ix: ix, path: 'untrigger.custom' })
          }

          // duration
          if (item.trigger && item.trigger.customDuration && item.trigger.customDuration.length > 0) {
            func.push({ id: item.id, name: this.$t('Duration Info ([-count-])', {count: 1}), ix: ix, path: 'trigger.customDuration' })
          }

          // overlay
          if (item.trigger && item.trigger.customOverlay1 && item.trigger.customOverlay1.length > 0) {
            var overlayCount = 1
            while (item.trigger['customOverlay' + overlayCount] && item.trigger['customOverlay' + overlayCount].length > 0) {
              func.push({ id: item.id, name: this.$t('Overlay [-num-] ([-count-])', {num: overlayCount, count: 1}), ix: ix, path: 'trigger.customOverlay' + overlayCount })
              overlayCount++
            }
          }

          // name
          if (item.trigger && item.trigger.customName && item.trigger.customName.length > 0) {
            func.push({ id: item.id, name: this.$t('Name Info ([-count-])', {count: 1}), ix: ix, path: 'trigger.customName' })
          }

          // icon
          if (item.trigger && item.trigger.customIcon && item.trigger.customIcon.length > 0) {
            func.push({ id: item.id, name: this.$t('Icon Info ([-count-])', {count: 1}), ix: ix, path: 'trigger.customIcon' })
          }

          // texture
          if (item.trigger && item.trigger.customTexture && item.trigger.customTexture.length > 0) {
            func.push({ id: item.id, name: this.$t('Texture Info ([-count-])', {count: 1}), ix: ix, path: 'trigger.customTexture' })
          }

          // stacks
          if (item.trigger && item.trigger.customStacks && item.trigger.customStacks.length > 0) {
            func.push({ name: this.$t('Stack Info ([-count-])', {count: 1}), ix: ix, path: 'trigger.customStacks' })
          }
        }

        // secondary triggers
        if (item.additional_triggers && item.additional_triggers.length > 0) {
          for (var k = 0; k < item.additional_triggers.length; k++) {
            if (item.additional_triggers[k].trigger && item.additional_triggers[k].trigger.type === 'custom' && item.additional_triggers[k].trigger.custom) {
              if (func.indexOf(item.id) < 0) {
                func.push(item.id)
              }

              func.push({ id: item.id, name: this.$t('Trigger ([-count-])', {count: k + 2}), ix: ix, path: 'additional_triggers[' + k + '].trigger.custom' })

              // untrigger
              if (item.additional_triggers[k].untrigger && item.additional_triggers[k].untrigger.custom && item.additional_triggers[k].untrigger.custom.length > 0) {
                func.push({ id: item.id, name: this.$t('Untrigger ([-count-])', {count: k + 2}), ix: ix, path: 'additional_triggers[' + k + '].untrigger.custom' })
              }

              // duration
              if (item.trigger && item.trigger.customDuration && item.trigger.customDuration.length > 0) {
                func.push({ id: item.id, name: this.$t('Duration Info ([-count-])', {count: k + 2}), ix: ix, path: 'additional_triggers[' + k + '].trigger.customDuration' })
              }

              // name
              if (item.trigger && item.trigger.customName && item.trigger.customName.length > 0) {
                func.push({ id: item.id, name: this.$t('Name Info ([-count-])', {count: k + 2}), ix: ix, path: 'additional_triggers[' + k + '].trigger.customName' })
              }

              // icon
              if (item.trigger && item.trigger.customIcon && item.trigger.customIcon.length > 0) {
                func.push({ id: item.id, name: this.$t('Icon Info ([-count-])', {count: k + 2}), ix: ix, path: 'additional_triggers[' + k + '].trigger.customIcon' })
              }

              // texture
              if (item.trigger && item.trigger.customTexture && item.trigger.customTexture.length > 0) {
                func.push({ id: item.id, name: this.$t('Texture Info ([-count-])', {count: k + 2}), ix: ix, path: 'additional_triggers[' + k + '].trigger.customTexture' })
              }

              // stacks
              if (item.trigger && item.trigger.customStacks && item.trigger.customStacks.length > 0) {
                func.push({ id: item.id, name: this.$t('Stack Info ([-count-])', {count: k + 2}), ix: ix, path: 'additional_triggers[' + k + '].trigger.customStacks' })
              }
            }
          }

          if (func.indexOf(item.id) < 0) {
            func.push(item.id)
          }

          // trigger logic (must have at least 2 triggers)
          if (item.disjunctive === 'custom' && item.customTriggerLogic && item.customTriggerLogic.length > 0) {
            func.push({ id: item.id, name: this.$t('Trigger Logic'), ix: ix, path: 'customTriggerLogic' })
          }
        }

        // animation onStart functions
        if (item.animation && item.animation.start) {
          // animate alpha
          if (item.animation.start.use_alpha && item.animation.start.alphaType === 'custom' && item.animation.start.alphaFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('onStart animate alpha'), ix: ix, path: 'animation.start.alphaFunc' })
          }

          // animate color
          if (item.animation.start.use_color && item.animation.start.colorType === 'custom' && item.animation.start.colorFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('onStart animate color'), ix: ix, path: 'animation.start.colorFunc' })
          }

          // animate rotation
          if (item.animation.start.use_rotate && item.animation.start.rotateType === 'custom' && item.animation.start.rotateFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('onStart animate rotation'), ix: ix, path: 'animation.start.rotateFunc' })
          }

          // animate scale
          if (item.animation.start.use_scale && item.animation.start.scaleType === 'custom' && item.animation.start.scaleFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('onStart animate scale'), ix: ix, path: 'animation.start.scaleFunc' })
          }

          // animate translation
          if (item.animation.start.use_translate && item.animation.start.translateType === 'custom' && item.animation.start.translateFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('onStart animate translation'), ix: ix, path: 'animation.start.translateFunc' })
          }
        }

        // animation main/ongoing functions
        if (item.animation && item.animation.main) {
          // animate alpha
          if (item.animation.main.use_alpha && item.animation.main.alphaType === 'custom' && item.animation.main.alphaFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Main animate alpha'), ix: ix, path: 'animation.main.alphaFunc' })
          }

          // animate color
          if (item.animation.main.use_color && item.animation.main.colorType === 'custom' && item.animation.main.colorFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Main animate color'), ix: ix, path: 'animation.main.colorFunc' })
          }

          // animate rotation
          if (item.animation.main.use_rotate && item.animation.main.rotateType === 'custom' && item.animation.main.rotateFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Main animate rotation'), ix: ix, path: 'animation.main.rotateFunc' })
          }

          // animate scale
          if (item.animation.main.use_scale && item.animation.main.scaleType === 'custom' && item.animation.main.scaleFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Main animate scale'), ix: ix, path: 'animation.main.scaleFunc' })
          }

          // animate translation
          if (item.animation.main.use_translate && item.animation.main.translateType === 'custom' && item.animation.main.translateFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Main animate translation'), ix: ix, path: 'animation.main.translateFunc' })
          }
        }

        // animation finish functions
        if (item.animation && item.animation.finish) {
          // animate alpha
          if (item.animation.finish.use_alpha && item.animation.finish.alphaType === 'custom' && item.animation.finish.alphaFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Finish animate alpha'), ix: ix, path: 'animation.finish.alphaFunc' })
          }

          // animate color
          if (item.animation.finish.use_color && item.animation.finish.colorType === 'custom' && item.animation.finish.colorFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Finish animate color'), ix: ix, path: 'animation.finish.colorFunc' })
          }

          // animate rotation
          if (item.animation.finish.use_rotate && item.animation.finish.rotateType === 'custom' && item.animation.finish.rotateFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Finish animate rotation'), ix: ix, path: 'animation.finish.rotateFunc' })
          }

          // animate scale
          if (item.animation.finish.use_scale && item.animation.finish.scaleType === 'custom' && item.animation.finish.scaleFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Finish animate scale'), ix: ix, path: 'animation.finish.scaleFunc' })
          }

          // animate translation
          if (item.animation.finish.use_translate && item.animation.finish.translateType === 'custom' && item.animation.finish.translateFunc) {
            if (func.indexOf(item.id) < 0) {
              func.push(item.id)
            }

            func.push({ id: item.id, name: this.$t('Finish animate translation'), ix: ix, path: 'animation.finish.translateFunc' })
          }
        }
      })

      return func
    },

    saveChanges: function () {
      if (this.editorSelected === 'tabledata') {
        var post = {}
        post.wagoID = this.wago._id
        post.type = this.wago.type
        post.json = this.aceEditor.getValue()
        var vue = this
        this.http.post('/import/json/save', post).then((res) => {
          if (res.success) {
            window.eventHub.$emit('showSnackBar', this.$t('Wago saved successfully'))
            vue.$router.push('/' + vue.wago.slug)
          }
          else if (res && res.error) {
            window.eventHub.$emit('showSnackBar', res.error)
          }
          else {
            window.eventHub.$emit('showSnackBar', this.$t('Unknown error could not save'))
          }
        })
      }
      else {
        this.editorSelected = 'tabledata'
        var t = this
        setTimeout(function () {
          t.saveChanges()
        }, 50)
      }
    },
    extractWA: function (wa) {
      this.extractData = false
      this.$refs.extractFromGroupDialog.open()
      if (!this.tableData.d || !this.tableData.c) {
        return
      }
      // copy table
      var table = JSON.parse(JSON.stringify(this.tableData))
      delete table.c
      delete table.d

      // loop through grouped auras and find the one we selected
      for (var i = 0; i < this.tableData.c.length; i++) {
        if (this.tableData.c[i].id === wa) {
          table.d = this.tableData.c[i]
          var vue = this
          this.http.post('/import/json/extract', {wagoID: this.wago._id, type: 'WEAKAURA', json: JSON.stringify(table)}).then((res) => {
            if (res.success) {
              vue.extractData = {}
              vue.extractData.encoded = res.extracted
              vue.extractData.name = wa
            }
          })
          break
        }
      }
    },
    extractCopy: function () {
      try {
        document.getElementById('wago-extractString').select()
        var copied = document.execCommand('copy')
        if (copied) {
          window.eventHub.$emit('showSnackBar', this.$t('Import string copied'))
        }
        else {
          window.eventHub.$emit('showSnackBar', this.$t('Import string failed to copy please upgrade to a modern browser'))
        }
        return copied
      }
      catch (e) {
        console.log(e)
        window.eventHub.$emit('showSnackBar', this.$t('Import string failed to copy please upgrade to a modern browser'))
      }
    },
    exportChanges: function () {
      if (this.editorSelected === 'tabledata') {
        this.tableString = this.aceEditor.getValue()
        this.showExport = true
      }
      else {
        this.editorSelected = 'tabledata'
        var t = this
        setTimeout(function () {
          t.exportChanges()
        }, 50)
      }
    },
    hideExport: function () {
      this.showExport = false
    }
  },
  computed: {
    editorContent: {
      get: function () {
        return this.$store.state.wago.code.json
      },

      set: function () {

      }
    },
    groupedWA: function () {
      if (!this.tableData.c) {
        return false
      }

      var auras = []

      // loop through each aura in array
      this.tableData.c.forEach((item, key) => {
        if (item && item.id) {
          auras.push(item.id)
        }
      })
      return auras
    },
    wago: function () {
      return this.$store.state.wago
    },
    canEdit: function () {
      var user = this.$store.state.user
      var wago = this.$store.state.wago

      if (user && user.UID && wago.UID && user.UID === wago.UID) {
        return true
      }
      return false
    },
    editorTheme: function () {
      if (!this.$store.state.user || !this.$store.state.user.config || !this.$store.state.user.config.editor) {
        return 'tomorrow'
      }
      else {
        return this.$store.state.user.config.editor || 'tomorrow'
      }
    }
  }
}
</script>

<style>
#edit-weakaura .md-select-content { max-height: calc(70vh); margin-bottom: 32px }
#edit-weakaura .md-select { width: auto }
#edit-weakaura .md-input-container { margin-bottom: 10px}
#edit-weakaura .md-input-container:after { content: none }
#edit-weakaura .flex-container { display: flex; flex-direction: row; flex-wrap: nowrap; align-items: flex-end}
#edit-weakaura .flex-left { order: 0; flex: 0 1 auto}
#edit-weakaura .flex-left .md-input-container label { white-space: nowrap}
#edit-weakaura .flex-right { order: 1; flex: 1 1 auto; text-align: right}
#edit-weakaura .ace_editor { box-shadow: 0 1px 5px rgba(0, 0, 0, 0.2), 0 2px 2px rgba(0, 0, 0, 0.14), 0 3px 1px -2px rgba(0, 0, 0, 0.12); }
.customFn .md-subheader { color: #c0272e }
</style>
