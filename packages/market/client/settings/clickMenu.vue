<script lang="ts" setup>
import { defineProps, computed } from 'vue';
import { Tree, plugins, setPath, splitPath } from './utils'
import { send } from '@koishijs/client'

const props = defineProps<{
  close: Function,
  node: { short: string; long: string; }
}>()

const onClickItems = (oldPath: string,ctxPath: string) => {
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

interface groupList {
    name: string,
    path: string
}

const getGroupList = (startGroup = plugins.value.data[0].children,parentPath='') => {
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
        // console.log('clid',startGroup[item].alias,startGroup[item].children);
        if(startGroup[item].children != undefined && startGroup[item].children.length != 0) {
            // console.log("递归");
            list.push(...getGroupList(startGroup[item].children,`${parentPath}${parentPath+startGroup[item].alias}/`))
        }
    }
    return list
}

const groupList = computed(getGroupList)

</script>
<template>
    <div class="clickMenu-container">
        <el-card class="box-card">
            <div 
                v-for="item of groupList" 
                :key="item.name"
                @click="onClickItems(props.node.long,item.path)"
                class="item">
                {{'移动到 ' + item.name }}
            </div>
        </el-card>
        <teleport to='body'>
            <div class="shadow" @click="props.close()"></div>
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
    margin-top: 4px;
    margin-right: 2px;
    margin-bottom: 4px;
    margin-left: 2px;
    cursor: pointer;
}
</style>