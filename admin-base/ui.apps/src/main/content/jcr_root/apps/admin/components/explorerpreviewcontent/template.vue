<template>
  <div :class="['explorer-preview-content', `preview-${nodeType}`]">

    <template v-if="currentObject">
      <div class="explorer-preview-nav">
        <ul class="nav-left" v-if="hasMultipleTabs">
          <explorer-preview-nav-item
              v-if="!!($slots.default)"
              icon="view_list"
              title="component explorer"
              :class="{'active': isTab(Tab.COMPONENTS)}"
              @click="setActiveTab(Tab.COMPONENTS)"/>

          <explorer-preview-nav-item
              icon="settings"
              :title="`${nodeType}-info`"
              :class="{'active': isTab(Tab.INFO)}"
              @click="setActiveTab(Tab.INFO)"/>

          <explorer-preview-nav-item
              v-if="hasOgTags"
              icon="label"
              :title="'og-tags'"
              :class="{'active': isTab(Tab.OG_TAGS)}"
              @click="setActiveTab(Tab.OG_TAGS)"/>

          <explorer-preview-nav-item
              v-if="hasReferences"
              icon="list"
              :title="'references'"
              :class="{'active': isTab(Tab.REFERENCES)}"
              @click="setActiveTab(Tab.REFERENCES)"/>

          <explorer-preview-nav-item
              icon="restore_page"
              :title="`${nodeType}-versions`"
              :class="{'active': isTab(Tab.VERSIONS)}"
              @click="setActiveTab(Tab.VERSIONS)"/>

          <explorer-preview-nav-item
              icon="more_vert"
              :title="'actions'"
              :class="{'active': isTab(Tab.ACTIONS)}"
              @click="setActiveTab(Tab.ACTIONS)"/>
        </ul>

        <ul class="nav-right"></ul>
      </div>

      <template v-if="isTab([Tab.COMPONENTS])">
        <slot></slot>
      </template>

      <template v-else-if="isTab([Tab.INFO, Tab.OG_TAGS])">
        <div v-if="hasInfoView && !edit"
             :class="`${nodeType}-info-view`">
          <img v-if="isImage"
               :src="currentObject"
               class="info-view-image"/>
          <iframe
              v-else
              :src="currentObject"
              class="info-view-iframe">
          </iframe>
        </div>
        <vue-form-generator
            v-if="node && getSchemaByActiveTab()"
            :class="{'vfg-preview': !edit}"
            :schema="getSchemaByActiveTab()"
            :model="node"
            :options="options"
            @validated="onValidated()"
            @model-updated="onModelUpdate">
        </vue-form-generator>
        <div class="explorer-confirm-dialog">
          <template v-if="edit">
            <button
                class="btn btn-raised waves-effect waves-light right"
                type="button"
                :title="`cancel editing ${nodeType} properties`"
                @click.stop.prevent="onCancel()">
              <icon icon="close"/>
            </button>
            <button
                class="btn btn-raised waves-effect waves-light right"
                type="button"
                :title="`save ${nodeType} properties`"
                :disabled="!valid"
                @click.stop.prevent="save()">
              <icon icon="check_box"/>
            </button>
          </template>
          <template v-else>
            <span></span>
            <button
                class="btn btn-raised waves-effect waves-light right"
                type="button"
                :title="`edit ${nodeType} properties`"
                @click.stop.prevent="onEdit()">
              <icon icon="edit"/>
            </button>
          </template>
        </div>
      </template>

      <template v-else-if="isTab(Tab.REFERENCES)">
        <ul :class="['collection', 'with-header', `explorer-${nodeType}-referenced-by`]">
          <li class="collection-header">
            referenced in {{ referencedBy.length }} location<span v-if="referencedBy.length !== 1 ">s</span>
          </li>
          <li v-for="item in referencedBy" :key="item.path" class="collection-item">
            <span>
              <span v-if="item.count" class="count">{{
                  item.count > 99 ? '99+' : item.count
                }}</span>
              <span class="right">
                <action
                    v-bind:model="{
                      target: itemToTarget(item.path),
                      command: 'editReference',
                      tooltipTitle: `edit '${item.name}'`
                    }">
                    <bdo>{{ item.path }}</bdo>
                </action>
              </span>
              <span class="edit-icon">
                <action
                    v-bind:model="{
                      target: itemToTarget(item.path),
                      command: 'editReference',
                      tooltipTitle: `edit '${item.name}'`
                    }">
                    <icon-edit-page/>
                </action>
              </span>
            </span>
          </li>
        </ul>
      </template>

      <template v-else-if="isTab(Tab.VERSIONS)">
        <div v-if="allowOperations" class="action-list">
          <div class="action"
               v-on:click.stop.prevent="createVersion"
               v-bind:title="`create new ${nodeType} version`">
            <icon icon="create"/>
            Create new {{ nodeType }} version
          </div>

          <p v-if="!hasVersions"
             v-bind:title="`no versions created yet`">
            This {{ nodeType }} has no versions
          </p>
          <template v-else>
            <div v-for="version in versions"
                 class="action"
                 v-bind:key="version.name"
                 v-on:click="checkoutVersion(version)"
                 v-bind:title="`Version ${version.name}`">
              <icon v-if="version.base" icon="check_box"/>
              <icon v-else-if="!version.base" icon="check_box_outline_blank"/>
              {{ version.name }} -{{ version.created }} {{ version.base ? '(current)' : '' }}
              <span v-if="!version.base" class="deleteVersionWrapper">
                          <action
                              v-bind:model="{
                                command: 'deleteVersion',
                                target: {version: version, path: currentObject},
                                tooltipTitle: 'delete version'}">
                            <icon icon="delete"/>
                          </action>
                      </span>
            </div>
          </template>

        </div>
      </template>

      <template v-else-if="isTab(Tab.ACTIONS)">
        <div v-if="allowOperations" class="action-list">
          <div v-if="nodeType === NodeType.PAGE"
               class="action"
               title="open live version"
               @click="openLiveVersion">
            <icon icon="external-link" :lib="IconLib.FONT_AWESOME"/>
            Open live version
          </div>
          <div class="action" :title="`rename ${nodeType}`" @click="$refs.renameModal.open()">
            <icon :lib="IconLib.MATERIAL_ICONS" icon="text_format"/>
            Rename {{ nodeType }}
          </div>
          <div class="action" :title="`move ${nodeType}`" @click="moveNode()">
            <icon icon="compare_arrows"/>
            Move {{ nodeType }}
          </div>
          <div class="action" :title="`copy ${nodeType}`" @click="copyNode()">
            <icon icon="content_copy"/>
            Copy {{ nodeType }}
          </div>
          <div class="action" :title="`delete ${nodeType}`" @click="deleteNode()">
            <icon icon="delete"/>
            Delete {{ nodeType }}
          </div>
        </div>
      </template>

    </template>

    <template v-else>
      <div v-if="!currentObject" class="explorer-preview-empty">
        <span>{{ $i18n(`no${uNodeType}Selected`) }}</span>
        <i class="material-icons">info</i>
      </div>
    </template>

    <materialize-modal
        ref="renameModal"
        v-bind:modalTitle="modalTitle"
        v-on:ready="onReady">
      <vue-form-generator
          :model="formmodel"
          :schema="nameSchema"
          :options="formOptions"
          ref="renameForm">
      </vue-form-generator>
      <template slot="footer">
        <confirm-dialog
            submitText="submit"
            v-on:confirm-dialog="onConfirmDialog"/>
      </template>
    </materialize-modal>

    <path-browser
        v-if="isOpen"
        :isOpen="isOpen"
        :header="`Move ${nodeName}`"
        :browserRoot="browserRoot"
        :browserType="nodeType"
        :currentPath="path.current"
        :selectedPath="path.selected"
        :setCurrentPath="setCurrentPath"
        :setSelectedPath="setSelectedPath"
        :onCancel="onMoveCancel"
        :onSelect="onMoveSelect">
    </path-browser>

    <path-browser
        v-if="isCopyOpen"
        :isOpen="isCopyOpen"
        :header="`Copy ${nodeName}`"
        :browserRoot="browserRoot"
        :browserType="nodeType"
        :currentPath="path.current"
        :selectedPath="path.selected"
        :setCurrentPath="setCurrentPath"
        :setSelectedPath="setSelectedPath"
        :onCancel="onCopyCancel"
        :onSelect="onCopySelect">
    </path-browser>

  </div>
</template>

<script>
import {IconLib, MimeType, NodeType, SUFFIX_PARAM_SEPARATOR} from '../../../../../../js/constants'
import {deepClone, get} from '../../../../../../js/utils'
import NodeNameValidation from '../../../../../../js/mixins/NodeNameValidation'
import Icon from '../icon/template.vue'
import PathBrowser from '../pathbrowser/template.vue'
import MaterializeModal from '../materializemodal/template.vue'
import ConfirmDialog from '../confirmdialog/template.vue'
import Action from '../action/template.vue'
import ExplorerPreviewNavItem from '../explorerpreviewnavitem/template.vue'
import IconEditPage from '../iconeditpage/template.vue'

const Tab = {
  INFO: 'info',
  OG_TAGS: 'og-tags',
  REFERENCES: 'references',
  VERSIONS: 'versions',
  COMPONENTS: 'components',
  ACTIONS: 'actions'
};

const SchemaKey = {
  MODEL: 'model',
  OG_TAGS: 'ogTags'
};

export default {
  name: 'ExplorerPreviewContent',
  components: {
    Icon,
    PathBrowser,
    MaterializeModal,
    ConfirmDialog,
    Action,
    ExplorerPreviewNavItem,
    IconEditPage
  },
  props: {
    model: {
      type: Object,
      required: true
    },
    nodeType: {
      type: String,
      required: true
    },
    browserRoot: {
      type: String,
      required: true
    },
    currentPath: {
      type: String,
      required: true
    },
    tab: {
      type: String,
      default: Tab.INFO
    },
    isEdit: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      Tab,
      SchemaKey,
      NodeType,
      IconLib,
      activeTab: null,
      edit: false,
      valid: {
        state: true,
        errors: null
      },
      isOpen: false,
      isCopyOpen: false,
      selectedPath: null,
      options: {
        validateAfterLoad: true,
        validateAfterChanged: true,
        focusFirstField: true
      },
      nodeTypeGroups: {
        ogTags: [NodeType.PAGE, NodeType.TEMPLATE],
        references: [NodeType.ASSET, NodeType.PAGE, NodeType.TEMPLATE, NodeType.OBJECT],
        selectStateAction: [NodeType.ASSET, NodeType.OBJECT],
        showProp: [NodeType.ASSET, NodeType.OBJECT],
        allowMove: [NodeType.PAGE, NodeType.TEMPLATE, NodeType.ASSET]
      },
      path: {
        current: null,
        selected: null
      },
      formGenerator: {
        changes: []
      }
    }
  },
  mixins: [NodeNameValidation],
  computed: {
    uNodeType() {
      return this.capFirstLetter(this.nodeType);
    },
    modalTitle() {
      return `Rename ${this.uNodeType}`
    },
    rawCurrentObject() {
      return $perAdminApp.getNodeFromViewOrNull(`/state/tools/${this.nodeType}`);
    },
    currentObject() {
      const obj = this.rawCurrentObject;
      if (this.nodeTypeGroups.showProp.indexOf(this.nodeType) > -1) {
        if (obj && obj.hasOwnProperty('show')) {
          return obj.show;
        } else {
          return null;
        }
      }
      return obj;
    },
    node() {
      if (this.nodeType === NodeType.OBJECT) {
        return this.rawCurrentObject.data
      }
      return $perAdminApp.findNodeFromPath(this.$root.$data.admin.nodes, this.currentObject);
    },
    allowOperations() {
      return this.currentObject.split('/').length > 4;
    },
    allowMove() {
      return this.nodeTypeGroups.allowMove.indexOf(this.nodeType) > -1;
    },
    hasOgTags() {
      return this.nodeTypeGroups.ogTags.indexOf(this.nodeType) > -1;
    },
    hasReferences() {
      return this.nodeTypeGroups.references.indexOf(this.nodeType) > -1;
    },
    hasMultipleTabs() {
      return this.hasOgTags || this.hasReferences;
    },
    referencedBy() {
      return this.trimReferences($perAdminApp.getView().state.referencedBy.referencedBy);
    },
    versions() {
      return this.hasVersions ? $perAdminApp.getView().state.versions.versions : []
    },
    isImage() {
      const node = $perAdminApp.findNodeFromPath(
          $perAdminApp.getView().admin.nodes, this.currentObject);
      if (!node) {
        return false;
      }
      const mime = node.mimeType;
      return Object.values(MimeType.Image).indexOf(mime) >= 0
    },
    hasInfoView() {
      return [NodeType.ASSET].indexOf(this.nodeType) > -1;
    },
    hasVersions() {
      return $perAdminApp.getView().state.versions
          ? $perAdminApp.getView().state.versions.has_versions : false
    },
    nodeName() {
      let nodeName = this.node.name;
      if (this.nodeType === NodeType.OBJECT) {
        nodeName = this.node.path.split('/').slice(-1).pop()
      }
      return nodeName
    }
  },
  watch: {
    edit(newVal) {
      $perAdminApp.getNodeFromView('/state/tools').edit = newVal;
    },
    activeTab: function (tab) {
      if (tab === 'versions') {
        this.showVersions()
      }
    },
    currentObject: function (path) {
      if (this.activeTab === 'versions') {
        this.showVersions()
      }
    }
  },
  created() {
    this.activeTab = this.tab
  },
  mounted() {
    this.path.selected = this.selectedPath
    this.path.current = this.currentPath
  },
  methods: {
    itemToTarget(path) {
      const ret = {path, target: path}
      const tenant = $perAdminApp.getNodeFromViewOrNull('/state/tenant')
      if (path.startsWith(`/content/${tenant.name}/pages`)) {
        ret.path = `/content/admin/pages/pages/edit.html/path:${path}`
      } else if (path.startsWith(`/content/${tenant.name}/templates`)) {
        ret.path = `/content/admin/pages/templates/edit.html/path:${path}`
      } else {
        const segments = path.split('/')
        if (segments.length > 0) {
          segments.pop()
        }
        path = segments.join('/')
        ret.target = path
        if (path.startsWith(`/content/${tenant.name}/assets`)) {
          ret.load = ret.path = `/content/admin/pages/assets.html/path:${path}`
          ret.type = 'ASSET'
        } else if (path.startsWith(`/content/${tenant.name}/objects`)) {
          ret.load = ret.path = `/content/admin/pages/objects.html/path:${path}`
          ret.type = 'OBJECT'
        }
      }
      return ret
    },
    getSchema(schemaKey) {
      if (!this.node) {
        return null;
      }
      const view = $perAdminApp.getView();
      let component = this.node.component;
      if (this.nodeType === NodeType.ASSET) {
        component = 'admin-components-assetview';
      }
      if (this.nodeType === NodeType.OBJECT) {
        component = this.getObjectComponent();
      }
      const componentDefinitions = view.admin.componentDefinitions
      if (!componentDefinitions) {
        return {}
      }
      const cmpDefinition = view.admin.componentDefinitions[component]
      if (!cmpDefinition) {
        return {}
      }
      let schema = cmpDefinition[schemaKey];
      if (this.edit) {
        return schema;
      }
      if (!schema) {
        return {};
      }
      schema = deepClone(schema);
      schema.fields.forEach((field) => {
        field.preview = true;
        field.readonly = true;
        if (field.fields) {
          field.fields.forEach((field) => {
            field.readonly = true;
          });
        }
      });
      return schema;
    },
    getSchemaByActiveTab() {
      if (this.activeTab === Tab.INFO) {
        return this.getSchema(SchemaKey.MODEL);
      } else if (this.activeTab === Tab.OG_TAGS) {
        return this.getSchema(SchemaKey.OG_TAGS);
      } else {
        return {};
      }
    },
    trimReferences(referenceList) {
      return referenceList.reduce(
          (map => (r, a) => (!map.has(a.path) && map.set(a.path,
              r[r.push({
                name: a.name,
                path: a.path,
                propertyName: a.propertyName,
                propertyPath: a.propertyPath,
                count: 0
              }) - 1]),
              map.get(a.path).count++, r))(new Map),
          []
      );
    },
    getObjectComponent() {
      let resourceType = this.rawCurrentObject.data['component'];
      if (!resourceType) {
        resourceType = this.rawCurrentObject.data['sling:resourceType'];
        if (!resourceType) {
          resourceType = this.rawCurrentObject.data['objectPath'];
        }
      }
      return resourceType.split('/').join('-');
    },
    capFirstLetter(string) {
      return string.charAt(0).toUpperCase() + string.slice(1);
    },
    onEdit() {
      this.edit = true
      this.formGenerator.original = deepClone(this.node)
    },
    onCancel() {
      const payload = {selected: this.currentObject}
      this.edit = false
      let node = this.node
      this.formGenerator.changes.forEach((ch) => {
        node[ch.key] = ch.oldVal
      })
      this.formGenerator.changes = []
    },
    onModelUpdate(newVal, schemaKey) {
      if (this.edit) {
        this.formGenerator.changes.push({
          key: schemaKey,
          oldVal: this.formGenerator.original[schemaKey],
          newVal: newVal
        })
      }
    },
    onValidated(isValid, errors) {
      if (this.edit) {
        return;
      }
      this.valid.state = isValid;
      this.valid.errors = errors;
    },
    onConfirmDialog(event) {
      if (event === 'confirm') {
        const isValid = this.$refs.renameForm.validate()
        if (isValid) {
          this.renameNode(this.formmodel.name, this.formmodel.title)
        } else {
          return
        }
      }
      this.nameChanged = false
      this.formmodel.name = ''
      this.formmodel.title = ''
      this.$refs.renameForm.clearValidationErrors()
      this.$refs.renameModal.close()
    },
    onReady(event) {
      this.formmodel.name = this.node.name
      this.formmodel.title = this.node.title
    },
    renameNode(newName, newTitle) {
      const that = this;
      $perAdminApp.stateAction(`rename${this.uNodeType}`, {
        path: this.currentObject,
        name: newName,
        title: newTitle,
        edit: this.isEdit
      }).then(() => {
        if (that.nodeType === 'asset' || that.nodeType === 'object') {
          const currNode = $perAdminApp.getNodeFromView(`/state/tools/${that.nodeType}/show`)
          const currNodeArr = currNode.split('/');
          currNodeArr[currNodeArr.length - 1] = newName
          $perAdminApp.getNodeFromView(`/state/tools/${that.nodeType}`).show = currNodeArr.join(
              '/')
        } else { // page and template handling
          const currNode = $perAdminApp.getNodeFromView('/state/tools')[that.nodeType]
          const currNodeArr = currNode.split('/');
          currNodeArr[currNodeArr.length - 1] = newName
          $perAdminApp.getNodeFromView('/state/tools')[that.nodeType] = currNodeArr.join('/')
        }
        this.setActiveTab(Tab.INFO)
      })
    },
    moveNode() {
      $perAdminApp.getApi().populateNodesForBrowser(this.path.current, 'pathBrowser')
          .then(() => {
            this.isOpen = true;
          }).catch(() => {
        $perAdminApp.getApi().populateNodesForBrowser(`/content/${site.tenant}`, 'pathBrowser');
      });
    },
    copyNode() {
      $perAdminApp.getApi().populateNodesForBrowser(this.path.current, 'pathBrowser')
          .then(() => {
            this.isCopyOpen = true;
          }).catch(() => {
        $perAdminApp.getApi().populateNodesForBrowser(`/content/${site.tenant}`, 'pathBrowser');
      });

    },
    deleteNode() {
      const really = confirm(`Are you sure you want to delete this ${this.nodeType}?`);
      if (really) {
        $perAdminApp.stateAction(`delete${this.uNodeType}`, this.node.path).then(() => {
          $perAdminApp.stateAction(`unselect${this.uNodeType}`, {})
        }).then(() => {
          const path = $perAdminApp.getNodeFromView('/state/tools/pages')
          $perAdminApp.loadContent(
              '/content/admin/pages/pages.html/path' + SUFFIX_PARAM_SEPARATOR + path)
        })
        this.isOpen = false;
      }
    },
    setCurrentPath(path) {
      this.path.current = path;
    },
    setSelectedPath(path) {
      this.path.selected = path;
    },
    showVersions() {
      $perAdminApp.getApi().populateVersions(this.currentObject);
    },
    deleteVersion(me, target) {
      $perAdminApp.stateAction('deleteVersion', {path: target.path, version: target.version.name});
    },
    createVersion() {
      $perAdminApp.stateAction('createVersion', this.currentObject);
    },
    checkoutVersion(version) {
      if (version.base === true) {
        $perAdminApp.notifyUser('Info', 'You cannot checkout the current version')
        return
      }
      let self = this;
      $perAdminApp.askUser('Restore Version',
          `Would you like to restore ${version.name}? You may lose work unless you create a new version saving the current state.`,
          {
            yesText: 'Yes',
            noText: 'No',
            yes() {
              $perAdminApp.stateAction('restoreVersion',
                  {path: self.currentObject, versionName: version.name});
            },
            no() {
              console.log('no')
            }
          })
    },
    onCopySelect() {
      $perAdminApp.stateAction('copyPage', {
        srcPath: this.currentObject,
        targetPath: this.path.selected
      });
      this.isCopyOpen = false;
    },
    onCopyCancel() {
      this.isCopyOpen = false;
    },
    onMoveCancel() {
      this.isOpen = false;
    },
    onMoveSelect() {
      $perAdminApp.stateAction(`move${this.uNodeType}`, {
        path: this.node.path,
        to: this.path.selected,
        type: 'child'
      });
      $perAdminApp.stateAction(`unselect${this.uNodeType}`, {});
      this.isOpen = false;
    },
    save() {
      if (this.nodeType === NodeType.OBJECT) {
        this.saveObject();
      } else {
        $perAdminApp.stateAction(`save${this.uNodeType}Properties`, this.node);
        this.edit = false;
      }
    },
    saveObject() {
      let data = this.node;
      let {show} = this.rawCurrentObject;
      let _deleted = $perAdminApp.getNodeFromViewWithDefault('/state/tools/_deleted', {});

      //Find child nodes with subchildren for our edited object
      for (const key in data) {
        if (!data.hasOwnProperty(key)) {
          continue;
        }
        //If node (or deleted node) is an array of objects then we have a child node
        if ((Array.isArray(data[key]) && data[key].length && typeof data[key][0] === 'object') ||
            (Array.isArray(_deleted[key]) && _deleted[key].length && typeof _deleted[key][0]
                === 'object')) {

          let node = data[key];

          //loop through children
          let targetNode = {};
          //Insert deleted children
          for (const j in _deleted[key]) {
            if (!_deleted[key].hasOwnProperty(j)) {
              continue;
            }
            const deleted = _deleted[key][j];
            targetNode[deleted.name] = deleted;
          }
          //Insert children
          for (const i in node) {
            if (!node.hasOwnProperty(i)) {
              continue;
            }
            const child = node[i];
            targetNode[child.name] = child;
          }
          data[key] = targetNode;
        }
      }
      $perAdminApp.stateAction('saveObjectEdit', {data: data, path: show}).then(() => {
        $perAdminApp.getNodeFromView('/state/tools')._deleted = {}
      });
      $perAdminApp.stateAction('selectObject', {selected: show})
      this.edit = false;
    },
    setActiveTab(clickedTab) {
      this.activeTab = clickedTab;
    },
    isTab(arg) {
      if (Array.isArray(arg)) {
        return arg.indexOf(this.activeTab) > -1;
      }
      return this.activeTab === arg;
    },
    openLiveVersion() {
      const view = $perAdminApp.getView()
      const page = get(view, '/pageView/page', null)

      if (!page) return;

      const {
        primaryDomain,
        pagePath
      } = page

      if (primaryDomain && pagePath) {
        const tenant = pagePath.split('/')[1]

        window.open(`${primaryDomain}${pagePath}.html`, `${tenant}-live-version`)
      }
    }
  }
}
</script>

<style>
.deleteVersionWrapper {
  margin-left: auto;
}
</style>
