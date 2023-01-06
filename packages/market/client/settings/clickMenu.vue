<script lang="ts" setup>
import { defineProps, computed } from 'vue';
import { Tree, plugins, setPath, splitPath,removeItem } from './utils'
import { send } from '@koishijs/client'

const props = defineProps<{
  close: Function,
  open: Function,
  node: { short: string; long: string; }
}>()

const emits = defineEmits(['update:modelValue'])


interface groupList {
    name: string,
    path: string
}

interface pluginList {
    name: string,
    path: string
}

const getGroupList = (startGroup = plugins.value.data[0].children,parentPath = '') => {
    let list: Array<groupList> = [];
        for (const item in startGroup) {

        // 不是分组的 删掉删掉！
        if(startGroup[item].label != "group") continue;
        
        if(startGroup[item].alias != props.node.short) {
            // 管他，塞进去就完了
            list.push({
                name:parentPath+startGroup[item].alias,
                path:startGroup[item].path
            });
        } else {
            continue;
        }

        // 没孩子的爬 孩子0个的爬
        if(startGroup[item].children != undefined && startGroup[item].children.length != 0) {
            list.push(...getGroupList(startGroup[item].children,`${parentPath}${parentPath+startGroup[item].alias}/`))
        }
    }
    emits('update:modelValue', (list.length + 3) * 45 + 24)
    props.open()
    return list
}

const getPluginList = (startGroup: Array<Tree>) => {
    let list: Array<pluginList> = [];
    for (const item in startGroup) {
        // 是插件的压进去！
        if(startGroup[item].label != "group") {
            list.push({
                name:startGroup[item].label,
                path:startGroup[item].path
            });
        };
        
        // 有孩子的搜索出来
        if(startGroup[item].children != undefined && startGroup[item].children.length != 0) {
            list.push(...getGroupList(startGroup[item].children))
        }
    }
    return list
}

const groupList = computed(getGroupList)

const moveItem = (oldPath: string,ctxPath: string) => {
    console.log('op',oldPath,'ctx',ctxPath);
    const index = plugins.value.paths[ctxPath].children.length;
    console.log(index);
    send('manager/teleport', oldPath, ctxPath, index)
    const segments1 = splitPath(oldPath)
    const segments2 = ctxPath ? splitPath(ctxPath) : []
    segments2.push(segments1.pop())
    setPath(oldPath, segments2.join('/'))
    props.close()
}

const unGroup = (path: string) => {
    const list: Array<pluginList> = getPluginList(plugins.value.paths[path].children);
    // console.log(splitPath(path),path);
    const pathArray = splitPath(path);
    for (const item of list) {
        moveItem(item.path,pathArray.length == 1?"":pathArray[pathArray.length-2])
    }
    removeItem(path);
    return 6;
}

const isGroup = computed(()=>{
    console.log(props.node.long);
    
    return plugins.value.paths[props.node.long].label == "group";
})
</script>
<template>
    <div class="clickMenu-container">
        <el-card
            class="box-card"
            body-style="padding: 12px 10px">
            <div 
                class="ungroup item"
                v-if="isGroup"
                @click="unGroup(props.node.long)">
                解散分组
            </div>
            <div 
                class="disable-plugin item"
                v-if="isGroup"
                @click="undefined">
                禁用分组下所有插件
            </div>
            <div             
                class="enable-plugin item"
                v-if="isGroup"
                @click="undefined">
                启用分组下所有插件
            </div>
            <div 
                v-for="item of groupList" 
                :key="item.name"
                @click="moveItem(props.node.long,item.path)"
                class="item">
                {{'移动到 ' + item.name }}
            </div>
        </el-card>
        <teleport to='body'>
            <div class="shadow" @click="props.close()" @contextmenu.prevent="props.close()"></div>
        </teleport>
    </div>
</template>
<style scoped>
.clickMenu-container {
    z-index: 999999999;
}
.box-card {
    background-color: white;
}
html.dark .box-card {
    background-color: black;
}
.shadow {
    height: 100vh;
    width: 100vw;
    opacity: 0;
    background-color: black;
    z-index: 999999998;
    position: absolute;
}
.item {
    transition: 0.3s;
    border-radius: 5px;
    padding-top: 12px;
    padding-right: 8px;
    padding-bottom: 12px;
    padding-left: 8px;
    cursor: pointer;
    color: black;
}
.item:hover {
    background-color: rgba(0, 0, 0, 0.057);
    color: rgb(64, 64, 64);
}
.ungroup {
    color: #F56C6C;
}
.ungroup:hover {
    color: #f97f7f;
}
.disable-plugin {
    color: #F56C6C;
}
.disable-plugin:hover {
    color: #f97f7f;
}
.enable-plugin {
    color:#67C23A;
}
.enable-plugin:hover {
    color:#7cc658;
}
.a-line {

}
</style>