<script lang="ts" setup>
import { Color } from '@tiptap/extension-color'
import { ListItem, TaskItem, TaskList } from '@tiptap/extension-list'
import { TableKit } from '@tiptap/extension-table'
import TextAlign from '@tiptap/extension-text-align'
import { TextStyleKit } from '@tiptap/extension-text-style'
import StarterKit from '@tiptap/starter-kit'
import { Editor, EditorContent } from '@tiptap/vue-3'
import GlobalDragHandle from 'tiptap-extension-global-drag-handle'
import { computed, onBeforeMount, onMounted, ref } from 'vue'
import { useFileRefs } from '@/hooks/useFileRefs'
import { useFiles } from '@/hooks/useFiles'
import { getFileHash } from '@/utils'
import { getTime } from '@/utils/date'
import { FileUpload } from './extensions/FileUpload/FileUpload'

const props = defineProps<{
  uuid: string
}>()

const emit = defineEmits(['focus', 'blur'])

// 定义编辑器类型
type EditorInstance = Editor | null

const { addFile, getFileByHash, getFileByUrl } = useFiles()
const { addFileRef, getFileRefByHashAndRefid } = useFileRefs()

const editor = ref<EditorInstance>(null)
// const fileInput = ref<HTMLInputElement | null>(null)

function getTitle() {
  // 递归遍历节点提取文本
  function extractTextFromNode(node: any): string {
    if (!node)
      return ''

    // 如果是文本节点，直接返回文本内容
    if (node.type === 'text') {
      return node.text || ''
    }

    // 如果是heading、listItem或paragraph，提取其内部文本并直接返回
    if (['heading', 'listItem', 'paragraph'].includes(node.type)) {
      let text = ''
      if (node.content && Array.isArray(node.content)) {
        node.content.forEach((child: any) => {
          if (child.type === 'text') {
            text += child.text || ''
          }
        })
      }
      return text
    }

    // 其他类型节点，返回节点类型
    return `[${node.type}]`
  }

  const json = editor.value?.getJSON()

  // 获取标题
  let title = ''
  if (json?.content && json.content.length > 0 && json.content[0]) {
    title = extractTextFromNode(json.content[0]).trim()
  }

  // 获取简介
  const smalltext = editor.value?.getText().replace(title, '').replace(/\n+/g, ' ').trim().slice(0, 255)

  return { title, smalltext }
}

onMounted(() => {
  editor.value = new Editor({
    extensions: [
      Color.configure({ types: [TextStyleKit.name, ListItem.name] }),
      TextStyleKit,
      StarterKit,
      TextAlign.configure({
        types: ['heading', 'paragraph'],
      }),
      TaskList,
      TaskItem,
      TableKit,
      FileUpload.configure({
        /**
         * 通过url从indexeddb获取文件并转为url地址
         * 如果没有获取到就直接抛出错误
         * @param url 文件url
         */
        async loadFile(url: string) {
          let fileObj
          // 判断是否为SHA256哈希
          if (/^[a-f0-9]{64}$/i.test(url)) {
            // 如果是SHA256哈希，使用getFileByHash获取文件
            fileObj = await getFileByHash(url)
          }
          else {
            // 否则使用getFileByUrl获取文件
            fileObj = await getFileByUrl(url)
          }

          if (!fileObj) {
            console.warn('文件不存在:', url)
            throw new Error('文件不存在')
          }

          if (fileObj.file) {
            // 如果有文件对象，创建URL
            const fileType = fileObj.file.type || 'unknown'
            const fileUrl = URL.createObjectURL(fileObj.file)

            return {
              url: fileUrl,
              type: fileType,
            }
          }
          else if (fileObj.url) {
            // 如果有URL但没有文件对象，直接返回URL
            return {
              url: fileObj.url,
              type: 'unknown',
            }
          }

          // 如果没有文件也没有URL，抛出错误
          throw new Error('文件格式不支持')
        },
        /**
         * 图片加载完成后的回调
         * @param url 图片url
         * @param width 图片原始宽度
         * @param height 图片原始高度
         */
        onImageLoaded(url: string, width: number, height: number) {
          console.warn('图片加载完成', url, width, height)
          // 这里可以添加图片加载完成后的处理逻辑
          /**
           * 1. 先检查 file 是否存在
           * 2. 如果不存在，请求图片路径，把图片下载下来，然后转为blob再转为File对象
           */
        },
      }),
      GlobalDragHandle.configure({
        dragHandleWidth: 20, // default

        // The scrollTreshold specifies how close the user must drag an element to the edge of the lower/upper screen for automatic
        // scrolling to take place. For example, scrollTreshold = 100 means that scrolling starts automatically when the user drags an
        // element to a position that is max. 99px away from the edge of the screen
        // You can set this to 0 to prevent auto scrolling caused by this extension
        scrollTreshold: 100, // default

        // The css selector to query for the drag handle. (eg: '.custom-handle').
        // If handle element is found, that element will be used as drag handle.
        // If not, a default handle will be created
        dragHandleSelector: '.custom-drag-handle', // default is undefined

        // Tags to be excluded for drag handle
        // If you want to hide the global drag handle for specific HTML tags, you can use this option.
        // For example, setting this option to ['p', 'hr'] will hide the global drag handle for <p> and <hr> tags.
        excludedTags: [], // default

        // Custom nodes to be included for drag handle
        // For example having a custom Alert component. Add data-type="alert" to the node component wrapper.
        // Then add it to this list as ['alert']
        //
        customNodes: [],
      }),
    ],
    content: '',
    onBlur: () => {
      emit('blur')
    },
    onFocus: () => {
      emit('focus')
    },
  })
})

function setContent(content: string): void {
  editor.value?.commands.setContent(content)
}

/**
 * 1. 获取所选择文件，可能为多个
 * 2. 将文件转换为blob地址，并且正确匹配blob的文件类型
 * 3. 将blob地址使用setFileUpload插入到编辑器中
 */
async function insertFile(e: Event) {
  const files = (e.target as HTMLInputElement).files
  if (!files)
    return

  for (const file of Array.from(files)) {
    const hash = await getFileHash(file)
    const existFile = await getFileByHash(hash)
    const existFileRef = await getFileRefByHashAndRefid(hash, props.uuid)
    if (existFile) {
      editor.value!.commands.setFileUpload({ url: existFile?.url || hash })
    }
    else {
      await addFile({ hash, file, id: 0 })
      editor.value!.commands.setFileUpload({ url: hash })
    }
    if (!existFileRef)
      await addFileRef({ hash, refid: props.uuid, lastdotime: getTime(), isdeleted: 0 })
  }
}

function setEditable(editable: boolean) {
  editor.value!.setEditable(editable)
}

function setInputMode(inputMode: 'text' | 'none') {
  editor.value!.setOptions({
    editorProps: {
      attributes: {
        inputmode: inputMode,
      },
    },
  })
}

onBeforeMount(() => {
  editor.value?.destroy()
})

defineExpose({
  getContent: (): string | undefined => editor.value?.getHTML(),
  getTitle,
  setContent,
  setEditable,
  setInputMode,
  editor: computed(() => editor.value),
  insertFile,
})
</script>

<template>
  <div v-if="editor" class="yy-editor">
    <EditorContent :editor="editor as any" />
    <div class="button-group">
      <button
        :disabled="!editor.can().chain().focus().undo().run()"
        @click="editor.chain().focus().undo().run()"
      >
        撤销
      </button>
      <button
        :disabled="!editor.can().chain().focus().redo().run()"
        @click="editor.chain().focus().redo().run()"
      >
        重做
      </button>
      <button
        :disabled="!editor.can().chain().focus().toggleCode().run()"
        :class="{ 'is-active': editor.isActive('code') }"
        @click="editor.chain().focus().toggleCode().run()"
      >
        代码
      </button>
      <button @click="editor.chain().focus().unsetAllMarks().run()">
        清除标记
      </button>
      <button @click="editor.chain().focus().clearNodes().run()">
        清除节点
      </button>
      <button
        :class="{ 'is-active': editor.isActive('codeBlock') }"
        @click="editor.chain().focus().toggleCodeBlock().run()"
      >
        代码块
      </button>
      <button
        :class="{ 'is-active': editor.isActive('blockquote') }"
        @click="editor.chain().focus().toggleBlockquote().run()"
      >
        引用
      </button>
      <button @click="editor.chain().focus().setHorizontalRule().run()">
        水平线
      </button>
      <button @click="editor.chain().focus().setHardBreak().run()">
        硬换行
      </button>
      <button
        :class="{ 'is-active': editor.isActive('textStyle', { color: '#958DF1' }) }"
        @click="editor.chain().focus().setColor('#958DF1').run()"
      >
        紫色
      </button>
    </div>
  </div>
</template>

<style lang="scss">
.yy-editor {
  .button-group {
    button {
      padding: 6px 12px;
      border-radius: 4px;
      background: #161616;
      color: #a1a1a1;
      cursor: pointer;
      margin: 2px;
    }
  }

  .tiptap-image {
    max-width: 100%;
    height: auto;

    &.loading {
      opacity: 0.5;
    }

    &.error {
      border: 1px solid var(--danger);
    }
  }
}
/* Basic editor styles */
.tiptap {
  outline: none;
  padding: 16px;
  padding-bottom: 200px;
  :first-child {
    margin-top: 0;
  }

  /* Heading styles */
  h1,
  h2,
  h3,
  h4,
  h5,
  h6,
  p,
  ul,
  ol {
    margin: 0;
    line-height: 1.6;
    text-wrap: pretty;
  }

  h1 {
    font-size: 1.75rem;
    margin: 0;
  }

  h2 {
    font-size: 1.375rem;
  }

  h3 {
    font-size: 1.0625rem;
    font-weight: 400;
  }

  h4,
  h5,
  h6 {
    font-size: 1rem;
  }

  /* Code and preformatted text styles */
  code {
    background-color: #202329;
    border-radius: 0.4rem;
    padding: 0.25em 0.3em;
  }

  pre {
    background: #161b22;
    border-radius: 0.5rem;
    font-family: 'JetBrainsMono', monospace;
    margin: 8px 0;
    padding: 0.75rem 1rem;
    line-height: 1.45;
    font-size: 14px;
    code {
      background: none;
      padding: 0;
    }
  }

  blockquote {
    border-left: 3px solid var(--gray-3);
    margin: 1.5rem 0;
    padding-left: 1rem;
  }
  // 任务列表
  ul[data-type='taskList'] {
    list-style: none;
    margin-left: 0;
    padding: 0;

    li {
      align-items: center;
      display: flex;
      &[data-checked='true'] {
        label {
          background: url('/public/icons/check-circle-fill.svg') no-repeat center center;
          background-size: 100%;
        }
      }
      &[data-checked='false'] {
        label {
          background: url('/public/icons/circle.svg') no-repeat center center;
          background-size: 100%;
        }
      }

      > label {
        display: inline-flex;
        width: 22px;
        height: 22px;
        margin-right: 0.5rem;
        transform: translateY(-0.1em);
        cursor: pointer;
        //   flex: 0 0 auto;
        //   margin-right: 0.5rem;
        //   user-select: none;
      }

      > div {
        flex: 1 1 auto;
      }
    }

    input[type='checkbox'] {
      opacity: 0;
      pointer-events: none;
    }
  }
  // 列表
  ul,
  ol {
    padding: 0;

    li p {
      margin-top: 0.25em;
      margin-bottom: 0.25em;
    }
  }
  // 有序列表
  ol {
    padding-left: 28px;
    li {
      ol {
        li {
          list-style-type: lower-alpha;
          ol {
            li {
              list-style-type: decimal-leading-zero;
            }
          }
        }
      }
    }
  }
  // 无序列表
  ul {
    padding-left: 28px;
    li {
      &::marker {
        font-size: 20px;
      }
    }
  }
  table {
    border-collapse: collapse;
    border-spacing: 0;
    border: 1px solid var(--c-blue-gray-700);
    width: 100%;
    margin: 10px 0;
    td {
      border: 1px solid var(--c-blue-gray-700);
      padding: 8px;
    }
  }
}
</style>
