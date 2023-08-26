<template>
  <el-tree @click.right.prevent="handleNotifyMenu" :data="data" node-key="id" :props="defaultProps"
    @node-click="handleNodeClick" @current-change="handleCurrentChange" />
  <span v-html="selectedData.map(it => it.id).join('<br>')"></span>
  <ul class="list" ref="listRef" @click="handleClickListItem">
    <li class="list-item">操作1</li>
    <li class="list-item">操作2</li>
    <li class="list-item">操作3</li>
    <li class="list-item">操作4</li>
  </ul>
</template>

<script lang="ts" setup>
import { ElMessage } from 'element-plus';
import type Node from 'element-plus/es/components/tree/src/model/node'

type Tree = {
  label: string
  checked?: boolean,
  id: string,
  children?: Tree[]
} | {
  [key: string]: any
}
const listRef = ref<HTMLElement | null>(null)
const handleCurrentChange = (node: Tree) => {
}
const handleNotifyMenu = ({ clientX, clientY }: PointerEvent) => {
  listRef.value!.style.cssText += `visibility: visible; left: ${clientX}px; top: ${clientY}px`
}
const handleClickListItem = ({ target }: MouseEvent) => {
  listRef.value!.style.setProperty('visibility', 'hidden')
  ElMessage.success((target as HTMLElement).closest('.list-item')?.textContent!)
}
const recordStack: (Node | null)[] = new Array(2)
const selectedData = ref<Tree[]>([])
const handleNodeClick = (_data: Tree, node: Node) => {
  const siblings = node.parent.childNodes
  checkDir(siblings)

  if (shiftDown) {
    if (recordStack.includes(node)) return
    recordStack.splice(1, 1, node)
    const [firstEle, secondEle] = recordStack
    const firstEleIdx = siblings.findIndex(ele => ele.id === firstEle?.id),
      secondEleIdx = siblings.findIndex(ele => ele.id === secondEle?.id)
    const startIndex = Math.min(firstEleIdx, secondEleIdx),
      endIndex = Math.max(firstEleIdx, secondEleIdx)
    const temp = []
    for (let i = 0; i < siblings.length; i++) {
      const { data } = siblings[i];
      const flag = i >= startIndex && i <= endIndex
      if (flag) {
        data.checked = true
        temp.push(data)
      } else {
        data.checked = false
      }
    }
    selectedData.value = temp
  } else {
    recordStack.splice(0, 2, node, null)
    if (ctrlDown) {
      _data.checked = !_data.checked;
      const flag = selectedData.value.findIndex(t => t.id === _data.id)
      if (!~flag) {
        selectedData.value.push(_data)
        console.log(selectedData.value);
      } else {
        selectedData.value.splice(flag, 1)
      }
    } else {
      for (let i = 0; i < siblings.length; i++) {
        const { data } = siblings[i];
        if (data.id === _data.id) {
          data.checked = true
          selectedData.value = Array.of(data)
        } else {
          data.checked = false
        }
      }
    }
  }
}

const checkDir = (siblings: Node[]) => {
  const [fe, se] = recordStack
  const currentDirIdList = siblings.map(t => t.data.id)
  if (Array.of(fe?.data.id, se?.data.id).filter(Boolean).some(t => !currentDirIdList.includes(t))) {
    selectedData.value.forEach(item => void (item.checked = false))
    selectedData.value.length = 0
    recordStack.splice(0, 2, null)
  }
}

const data: Tree[] = reactive([
  {
    label: 'Level one 1',
    id: crypto.randomUUID(),
    children: [
      {
        label: 'Level two 1-1',
        id: crypto.randomUUID(),
        children: [
          {
            label: 'Level three 1-1-1',
          },
        ],
      },
    ],
  },
  {
    label: 'Level one 2',
    id: crypto.randomUUID(),
    children: [
      {
        label: 'Level two 2-1',
        id: crypto.randomUUID(),
        children: [
          {
            label: 'Level three 2-1-1',
            id: crypto.randomUUID(),
          },
          {
            label: 'Level three 2-1-2',
            id: crypto.randomUUID(),
          },
          {
            label: 'Level three 2-1-3',
            id: crypto.randomUUID(),
          },
          {
            label: 'Level three 2-1-4',
            id: crypto.randomUUID(),
          },
        ],
      },
      {
        label: 'Level two 2-2',
        id: crypto.randomUUID(),
        children: [
          {
            label: 'Level three 2-2-1',
            id: crypto.randomUUID(),
          },
        ],
      },
    ],
  },
  {
    label: 'Level one 3',
    id: crypto.randomUUID(),
    children: [
      {
        label: 'Level two 3-1',
        id: crypto.randomUUID(),
        children: [
          {
            label: 'Level three 3-1-1',
            id: crypto.randomUUID(),
          },
        ],
      },
      {
        label: 'Level two 3-2',
        id: crypto.randomUUID(),
        children: [
          {
            label: 'Level three 3-2-1',
            id: crypto.randomUUID(),
          },
        ],
      },
    ],
  },
])

const customNodeClass = (data: Tree) => {
  if (data.checked) {
    return 'is-selected'
  }
  return ''
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
onMounted(() => {
  document.addEventListener('keydown', keydownListener)
  document.addEventListener('keyup', keyupListener)
})
onUnmounted(() => {
  document.removeEventListener('keydown', keydownListener)
  document.removeEventListener('keyup', keyupListener)
})

const defaultProps = {
  children: 'children',
  label: 'label',
  class: customNodeClass
}
</script>

<style lang="scss" scoped>
:deep(.el-tree-node) {
  user-select: none;

  &.is-selected {
    background-color: var(--el-color-primary-light-9);

    .el-tree-node__content:hover {
      background-color: var(--el-color-primary-light-9);
    }

    &:focus {
      .el-tree-node__content {
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
  box-shadow: 0 0 6px 0 rgba(0, 0, 0, .2);
  visibility: hidden;

  &-item {
    width: 100px;
    padding: 5px 10px;
    text-align: center;
    transition: background-color .25s ease-in;

    &:hover {
      background-color: var(--el-color-primary-light-9);
      cursor: pointer;
    }
  }
}
</style>