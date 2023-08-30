<template>
  <div>
    <input type="file" @change="onUpload" />
    <span>MD5: {{ md5Value }}</span>
    <span>%: {{ uploadProgress }}</span>
    {{ test }}
  </div>
</template>

<script setup lang="ts">
import SparkMD5 from 'spark-md5'

const md5Value = ref<string>()
const test = {
  a: reactive({ b: 1 }),
  c: 2
}
setTimeout(() => (test.a.b = 3), 1000)
const uploadProgress = ref<number>()
const onUpload = async ({ target }: Event) => {
  // const spark = new SparkMD5.ArrayBuffer()
  const file = (target as HTMLInputElement).files![0]
  // const fr = new FileReader()
  // fr.readAsArrayBuffer(file)
  // fr.onload = ({ target }: ProgressEvent<FileReader>) => {
  //     spark.append((target?.result as ArrayBuffer))
  //     md5Value.value = spark.end();
  // }
  const chunks = await createFileChunks(file)
  const concurrentChunks = 3
  const promiseList = []

  while (chunks.length) {
    const request = chunks.splice(0, concurrentChunks).map(
      (t) =>
        new Promise((r) => {
          setTimeout(() => {
            console.log(t.currentNum)
            r(t.currentNum)
          }, 1000)
        })
    )
    await Promise.all(request)
    promiseList.push(Promise.all(request))
  }
  await Promise.all(promiseList)
  console.log(await Promise.all(promiseList))
}

interface FileChunk {
  currentNum: number
  fileName: string
  totalNum: number
  file: Blob
  key: string
}
const getFileHash = async ({ chunks }: { chunks: FileChunk[] }) =>
  new Promise<string>((resolve) => {
    const spark = new SparkMD5.ArrayBuffer()
    const appendToTaskQueue = ({ file }: { file: Blob }) =>
      new Promise((r) => {
        const fr = new FileReader()
        fr.readAsArrayBuffer(file)
        fr.onload = ({ target }: ProgressEvent<FileReader>) => {
          spark.append(target?.result as ArrayBuffer)
          r(null)
        }
      })
    let count = 0
    const workLoop = async (deadline: IdleDeadline) => {
      while (count < chunks.length && deadline.timeRemaining() > 1) {
        await appendToTaskQueue({ file: chunks[count].file })
        count++
        uploadProgress.value = Number(((100 * count) / chunks.length).toFixed(2))
        if (count >= chunks.length) resolve(spark.end())
      }
      window.requestIdleCallback(workLoop)
    }
    window.requestIdleCallback(workLoop)
  })

const createFileChunks = async (file: File, maxChunkSize = 1) => {
  maxChunkSize = maxChunkSize * 1024 * 1024
  const totalSize = file.size
  const fileType = file.type
  const fileName = file.name
  const fileChunks: FileChunk[] = []
  const chunkNum = Math.ceil(totalSize / maxChunkSize)
  const chunkSize = Math.ceil(totalSize / chunkNum)

  let start = 0
  for (let i = 0; i < chunkNum; i++) {
    const end = start + chunkSize
    fileChunks.push({
      currentNum: i + 1,
      fileName,
      totalNum: chunkNum,
      file: new Blob([file.slice(start, end)], { type: fileType }),
      key: ''
    })
    start = end
  }

  // 获取文件hash
  const fileKey = await getFileHash({ chunks: fileChunks })
  fileChunks.forEach((slice) => (slice.key = fileKey))
  return fileChunks
}

// const uploadChunks = () => {

// }
</script>

<style scoped></style>
