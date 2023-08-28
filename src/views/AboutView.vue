<template>
  <div class="box">
    <el-tree
      class="box-left"
      @contextmenu.prevent="handleNotifyMenu"
      :data="categoryData"
      node-key="id"
      :expand-on-click-node="false"
      :props="defaultProps"
      @node-click="handleNodeClick"
    />
    <span
      class="box-right"
      v-html="selectedData.map((it) => `${it.data.name}--${it.data.id}`).join('<br>')"
    ></span>
    <ul class="list" ref="listRef" @click="handleClickListItem">
      <li class="list-item">操作1</li>
      <li class="list-item">操作2</li>
      <li class="list-item">操作3</li>
      <li class="list-item">操作4</li>
    </ul>
  </div>
</template>

<script lang="ts" setup>
import { ElMessage } from 'element-plus'
import axios from 'axios'
import type Node from 'element-plus/es/components/tree/src/model/node'
import type { TreeNodeData } from 'element-plus/es/components/tree/src/tree.type'

type Tree = {
  name: string
  checked?: boolean
  id: number
  path: string
  children: Tree[]
}
const listRef = ref<HTMLElement | null>(null)

const handleNotifyMenu = ({ clientX, clientY }: PointerEvent) => {
  listRef.value!.style.cssText += `visibility: visible; left: ${clientX}px; top: ${clientY}px`
}
const handleClickListItem = ({ target }: MouseEvent) => {
  ElMessage.success((target as HTMLElement).closest('.list-item')?.textContent!)
}
const recordStack: (Node | null)[] = new Array(2)
const selectedData = ref<Node[]>([])
const handleNodeClick = (_data: Tree, node: Node) => {
  const siblings = node.parent.childNodes
  const hasChildNodes = node.childNodes.length > 0
  checkDir(node)
  rightMenuListener()

  if (shiftDown) {
    if (recordStack.includes(node)) return
    recordStack.splice(1, 1, node)
    const [firstEle, secondEle] = recordStack
    const firstEleIdx = siblings.findIndex((ele) => ele.id === firstEle?.id),
      secondEleIdx = siblings.findIndex((ele) => ele.id === secondEle?.id)
    const startIndex = Math.min(firstEleIdx, secondEleIdx),
      endIndex = Math.max(firstEleIdx, secondEleIdx)
    const temp = []
    for (let i = 0; i < siblings.length; i++) {
      const { data, childNodes } = siblings[i]
      const flag = i >= startIndex && i <= endIndex
      if (flag) {
        data.checked = true
        temp.push(siblings[i])
        hasChildNodes && handleChildNodes(childNodes, true, temp)
      } else {
        data.checked = false
        hasChildNodes && handleChildNodes(childNodes, false, temp)
      }
    }
    selectedData.value = temp
  } else {
    recordStack.splice(0, 2, node, null)
    if (ctrlDown) {
      _data.checked = !_data.checked
      const flag = selectedData.value.findIndex((t) => t.data.id === _data.id)
      if (!~flag) {
        selectedData.value.push(node)
        hasChildNodes && handleChildNodes(node.childNodes, true)
      } else {
        selectedData.value.splice(flag, 1)
        hasChildNodes && handleChildNodes(node.childNodes, false)
      }
    } else {
      for (let i = 0; i < siblings.length; i++) {
        const { data, childNodes } = siblings[i]
        if (data.id === _data.id) {
          data.checked = true
          selectedData.value = Array.of(toRaw(siblings[i]))
          hasChildNodes && handleChildNodes(childNodes, true)
        } else {
          data.checked = false
          hasChildNodes && handleChildNodes(childNodes, false)
        }
      }
    }
  }
}

const handleChildNodes = (childNodes: Node[], checked = false, o = selectedData.value) => {
  childNodes.forEach((t) => {
    t.data.checked = checked
    if (t.childNodes.length) {
      handleChildNodes(t.childNodes, checked)
    }
    if (checked) {
      o.push(t)
    } else {
      const idx = o.findIndex((it) => it.data.id === t.data.id)
      if (~idx) {
        o.splice(idx, 1)
      }
    }
  })
}

const checkDir = (currentNode: Node) => {
  const [fe, se] = recordStack
  const currentDirIdList = currentNode.parent.childNodes.map((t) => t.data.id)
  if (
    Array.of(fe?.data.id, se?.data.id)
      .filter(Boolean)
      .some((t) => !currentDirIdList.includes(t)) ||
    (!shiftDown && !ctrlDown)
  ) {
    selectedData.value.forEach((item) => {
      item.data.checked = false
      item.childNodes.length && handleChildNodes(item.childNodes, false)
    })
    selectedData.value.length = 0
    recordStack.splice(0, 2, null)
  }
}

const categoryData: Tree[] = reactive([
  {
    name: '企业文档',
    id: 0,
    path: '',
    checked: false,
    children: []
  }
])

const instance = axios.create({
  baseURL: 'xxx',
  headers: {
    Authorization:
      'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJVc2VySWQiOjE0MjMwNzA3MDkxMDU1MSwiQWNjb3VudCI6ImtmYmlvIiwiTmFtZSI6Iui2hee6p-euoeeQhuWRmCIsIlN1cGVyQWRtaW4iOjEsIlBob25lIjoiMTgwMjAwMzA3MjEiLCJpYXQiOjE2OTMyMDMxMjksIm5iZiI6MTY5MzIwMzEyOSwiZXhwIjoxNjkzMjE1MTI5LCJpc3MiOiJkaWxvbiIsImF1ZCI6ImRpbG9uIn0.ARMu1KKNJ1EsCbrg7fdXEp1Lakv7i3GAxjVUXIhDNRE'
  }
})

interface IResponseData<T = any> {
  code: number
  data: T
  success: boolean
  message: string
  [key: string]: any
}
const fetch = async () =>
  (await instance.get<IResponseData<Tree[]>>('/api/documentFolder/list')).data

fetch().then(({ data, success }) => {
  if (success && data) {
    categoryData[0].children.push(...data)
  }
})

const customNodeClass = (data: TreeNodeData) => {
  if (data.checked) {
    return 'is-selected'
  }
  return ''
}
const defaultProps = {
  children: 'children',
  label: 'name',
  isLeaf: 'isLeaf',
  class: customNodeClass
}

let shiftDown = false
let ctrlDown = false
const keyupListener = ({ code }: KeyboardEvent) => {
  shiftDown = code.includes('Shift') ? false : shiftDown
  ctrlDown = code.includes('Control') ? false : ctrlDown
}
const keydownListener = ({ code }: KeyboardEvent) => {
  shiftDown = code.includes('Shift')
  ctrlDown = code.includes('Control')
}
const rightMenuListener = () => {
  listRef.value?.style.setProperty('visibility', 'hidden')
}
onMounted(() => {
  document.addEventListener('keydown', keydownListener)
  document.addEventListener('keyup', keyupListener)
  document.addEventListener('click', rightMenuListener)
})
onUnmounted(() => {
  document.removeEventListener('keydown', keydownListener)
  document.removeEventListener('keyup', keyupListener)
  document.removeEventListener('click', rightMenuListener)
})
</script>

<style lang="scss" scoped>
.box {
  height: 100vh;
  gap: 20px;
  display: flex;

  &-left {
    min-width: 200px;
    overflow: auto;
    padding: 20px;
    border-right: 1px rgb(224, 218, 218) solid;
  }

  &-right {
    overflow: auto;
    flex: 1;
    display: flex;
    justify-content: center;
    align-items: center;
  }
}

:deep(.el-tree-node) {
  user-select: none;

  &.is-selected {
    > .el-tree-node__content {
      background-color: var(--el-color-primary-light-9);
    }

    > .el-tree-node__content:hover {
      background-color: var(--el-color-primary-light-9);
    }

    &:focus {
      > .el-tree-node__content {
        background-color: var(--el-color-primary-light-9);
      }
    }
  }
}

.list {
  position: fixed;
  left: 110vw;
  list-style: none;
  border-radius: 5px;
  background-color: #fff;
  padding: 0;
  box-shadow: 0 0 3px 0 rgba(0, 0, 0, 0.2);
  visibility: hidden;

  &-item {
    width: 100px;
    padding: 5px 10px;
    text-align: center;
    transition: background-color 0.25s ease-in;

    &:hover {
      background-color: var(--el-color-primary-light-9);
      cursor: pointer;
    }
  }
}
</style>
