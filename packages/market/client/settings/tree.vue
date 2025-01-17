<template>
  <el-scrollbar class="plugin-tree" ref="root">
    <div class="search">
      <el-input v-model="keyword" #suffix>
        <k-icon name="search"></k-icon>
      </el-input>
    </div>
    <k-tab-item class="k-tab-group-title" label="@global" v-model="model">全局设置</k-tab-item>
    <el-tree
      ref="tree"
      node-key="id"
      @contextmenu.prevent="openClickMenu"
      :data="plugins.data"
      :draggable="true"
      :auto-expand-parent="false"
      :default-expanded-keys="plugins.expanded"
      :expand-on-click-node="false"
      :filter-node-method="filterNode"
      :props="{ class: getClass }"
      :allow-drag="allowDrag"
      :allow-drop="allowDrop"
      @node-click="handleClick"
      @node-drop="handleDrop"
      @node-expand="handleExpand"
      @node-collapse="handleCollapse"
      #="{ node }">
      <div class="item">
        <div class="label">{{ node.label === 'group' ? '分组：' + node.data.alias : node.label || '待添加' }}</div>
        <div class="right"></div>
      </div>
    </el-tree>
  </el-scrollbar>
  <teleport to='body'>
    <transition name="el-zoom-in-top">
      <clickMenu 
        :close="()=>{clickMenuSettings.isshow=false}"
        :open="updateClickMenuHeight"
        :node="clickMenuSettings.node"
        :style="clickMenuSettings.style"
        v-model="clickMenuHeight"
        v-if="clickMenuSettings.isshow">
      </clickMenu>
    </transition>
  </teleport>
</template>

<script lang="ts" setup>

import { ref, computed, reactive, onActivated, nextTick, watch } from 'vue'
import { send } from '@koishijs/client'
import { Tree, plugins, setPath, splitPath } from './utils'

import clickMenu from './clickMenu.vue';

const props = defineProps<{
  modelValue: string
}>()


const emits = defineEmits(['update:modelValue'])

const model = computed({
  get: () => props.modelValue,
  set: val => emits('update:modelValue', val),
})

function filterNode(value: string, data: Tree) {
  return data.label.includes(keyword.value)
}

function allowDrag(node: Node) {
  return node.data.path !== ''
}

interface Node {
  data: Tree
  parent: Node
  expanded: boolean
  isLeaf: boolean
  childNodes: Node[]
}

function allowDrop(source: Node, target: Node, type: 'inner' | 'prev' | 'next') {
  if (type !== 'inner') return target.data.path !== ''
  const segments = splitPath(target.data.path)
  if(!segments[segments.length - 1].startsWith) {
    return false;
  }
  return segments[segments.length - 1].startsWith('group:')
}

function handleClick(tree: Tree) {
  model.value = tree.path
}

function handleExpand(data: Tree, target: Node, instance) {
  send('manager/meta', data.path, { $collapsed: null })
}

function handleCollapse(data: Tree, target: Node, instance) {
  send('manager/meta', data.path, { $collapsed: true })
}

function handleDrop(source: Node, target: Node, position: 'before' | 'after' | 'inner', event: DragEvent) {
  const parent = position === 'inner' ? target : target.parent
  const oldPath = source.data.path
  const ctxPath = parent.data.path
  console.log("old",oldPath,'ctx',ctxPath);
  const index = parent.childNodes.findIndex(node => node.data.path === oldPath)
  console.log(index);
  
  send('manager/teleport', oldPath, ctxPath, index)
  const segments1 = splitPath(oldPath)
  const segments2 = ctxPath ? splitPath(ctxPath) : []
  segments2.push(segments1.pop())
  setPath(oldPath, segments2.join('/'))
}

function getClass(tree: Tree) {
  const words: string[] = []
  if (tree.children) words.push('is-group')
  if (tree.disabled) words.push('is-disabled')
  if (tree.path === model.value) words.push('is-active')
  return words.join(' ')
}

const root = ref<{ $el: HTMLElement }>(null)
const tree = ref(null)
const keyword = ref('')

watch(keyword, (val) => {
  tree.value.filter(val)
})

onActivated(async () => {
  const container = root.value.$el
  await nextTick()
  const element = container.querySelector('.k-tab-item.active') as HTMLElement
  if (!element) return
  root.value['setScrollTop'](element.offsetTop - (container.offsetHeight - element.offsetHeight) / 2)
})

var clickMenuSettings = reactive({
  isshow: false,
  hasDisabledSetting: false,
  style: {
    left: "0px",
    top: "0px",
    position: "absolute",
  },
  node: {
    short: '',
    long: ''
  }
})

var clickMenuHeight = ref(0);
const updateClickMenuHeight = ()=>{
  if(Number(clickMenuSettings.style.top.replace("px","")) + clickMenuHeight.value >= document.documentElement.clientHeight) clickMenuSettings.style.top = document.documentElement.clientHeight - clickMenuHeight.value - 30 + "px";
}

const openClickMenu = (event) => {  
  if (event.pageX + 400 > document.documentElement.clientWidth) {
    clickMenuSettings.style.left = (event.pageX - 200) + "px"
  } else {
    clickMenuSettings.style.left = event.pageX + "px"
  }
  console.log(event.path);
  
  let groupName = event.path[0].innerHTML.split("：")[1]
  groupName = groupName == undefined?event.path[0].innerHTML:groupName
  for (const item in plugins.value.paths) {
    if(item.endsWith(`${groupName}`)) {
      clickMenuSettings.node.short = groupName;
      clickMenuSettings.node.long = item;
    } else {
      // console.log(groupName,item);
    }
  }
  clickMenuSettings.isshow = true
  clickMenuSettings.style.top = event.pageY + "px"
    
}

</script>

<style lang="scss">

.plugin-tree {
  width: 100%;
  height: 100%;
  border-right: 1px solid var(--border);
  overflow: auto;

  .el-scrollbar__view {
    padding: 1rem 0;
    line-height: 2.25rem;
  }

  .search {
    padding: 0 1.5rem;
  }

  .k-icon-filter {
    height: 15px;
  }

  .readonly {
    color: var(--fg3t);
  }

  .el-tree {
    margin-top: 0.5rem;
    user-select: none;
  }

  .el-tree-node__expand-icon {
    margin-left: 8px;
  }

  .el-tree-node {
    &.is-group > .el-tree-node__content {
      font-weight: bold;
    }

    &:focus:not(:hover) > .el-tree-node__content {
      background-color: unset;
    }

    &.is-disabled > .el-tree-node__content .label {
      color: var(--fg3t);
    }

    &.is-active > .el-tree-node__content {
      background-color: var(--hover-bg);
      color: var(--active);
    }
  }

  .el-tree-node__content {
    line-height: 2.25rem;
    height: 2.25rem;
    transition: var(--color-transition);

    .item {
      flex: 1;
      height: 100%;
      display: flex;
      align-items: center;
      justify-content: space-between;
      overflow: hidden;
    }

    .label {
      overflow: hidden;
      text-overflow: ellipsis;
      transition: var(--color-transition);
    }

    .right {
      height: 100%;
      margin: 0 0.75rem 0 0.5rem;

      > span.button {
        display: inline-flex;
        height: 100%;
        width: 1.5rem;
        justify-content: center;
        align-items: center;
        opacity: 0.75;
        color: var(--fg3);
        transition: var(--color-transition);

        &:hover {
          color: var(--fg2);
          opacity: 1 !important;
        }
      }
    }

    &:hover {
      background-color: var(--hover-bg);
      .right > .button {
        opacity: 0.75;
      }
    }
  }

  .el-tree-node__label {
    font-size: 16px;
  }
}

</style>
