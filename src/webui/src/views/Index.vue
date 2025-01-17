<script setup>
import SideBar from './components/SideBar.vue'
import ChatItem from './components/ChatItem.vue'
import Welcome from './components/Welcome.vue'
import useDatabase from '@/hooks/useDatabase'
import useSSE from '../hooks/useSSE'
import useAppStore from '@/stores/app'
import {
  getConversationList,
  getMessagesByConversionId,
  addMessage,
  updateMessage,
  addConversation,
  deleteConversation as service_deleteConversation,
  editConversationName as service_editConversationName
} from '@/services'

const { db } = useDatabase()

const currentConversationId = ref('')

watch(
  () => currentConversationId.value,
  async () => {
    messages.value = await getMessagesByConversionId(currentConversationId.value)
    nextTick(() => {
      scrollToBottom()
    })
  }
)

// 对话列表
const conversations = ref([])

// 消息列表
const messages = ref([])

const RefChatList = ref(null)

function scrollToBottom() {
  nextTick(() => {
    RefChatList.value.scrollTop = RefChatList.value.scrollHeight
  })
}

const init = async () => {
  try {
    await db.open()

    // 初始化对话和对话内容表
    await initConversations()
    await initMessages()
  } catch (error) {
    console.error('Error initializing tables:', error)
  }
}

init()

const initConversations = async () => {
  conversations.value = await getConversationList()

  // 如果对话表为空，则创建一个示例对话
  if (conversations.value.length === 0) {
    // 将示例对话插入对话表
    await addConversation({ name: generateConversationName() })
    // 更新对话列表
    conversations.value = await getConversationList()
  }

  // 更新当前对话id为第一条对话
  currentConversationId.value = conversations.value[0].id
}

// 初始化消息表
const initMessages = async () => {
  // 查询当前对话的消息
  if (conversations.value.length > 0) {
    messages.value = await getMessagesByConversionId(currentConversationId.value)
    scrollToBottom()
  }
}

// 生成对话名称的逻辑
const generateConversationName = () => {
  return 'New Chat'
}

const message = ref('')

function fetchResult(params) {
  return new Promise((resolve) => {
    const modelParams = useAppStore().modelParams
    useSSE('/sse/subscribe', {
      params: Object.assign({}, params, toRaw(modelParams)),
      onmessage(ev) {
        console.log('ev', ev)
        try {
          const res = JSON.parse(ev.data)
          const { flag, resData } = res
          if (flag) {
            const { message } = resData
            resolve(message)
          }
        } catch (error) {
          console.error(error)
        }
      }
    })
  })
}

const AUTO_RENAME_LETTER_COUNT = 15

// 发送消息时自动重命名对话名称
function autoRenameConversation(id, name) {
  editConversationName(id, name)
}

// 发送消息
async function sendMessage() {
  if (!message.value) return
  // 创建message记录
  const msgItem = {
    conversation_id: currentConversationId.value,
    sender: 'USER',
    content: message.value
  }

  // 插入message
  await addMessage(msgItem)

  // 刷新messages列表
  messages.value = await getMessagesByConversionId(currentConversationId.value)

  const isFirstMessage = messages.value.length === 1
  if (isFirstMessage) {
    autoRenameConversation(
      currentConversationId.value,
      message.value.substring(0, AUTO_RENAME_LETTER_COUNT)
    )
  }

  // 清空输入框
  message.value = ''

  messages.value.push({
    isLoading: true
  })
  scrollToBottom()
  const MESSAGES_COUNT = useAppStore().maxMultiTurns || 0
  console.log('MESSAGES_COUNT', MESSAGES_COUNT)
  const _messages = messages.value.slice(-2 * (MESSAGES_COUNT + 1)).reduce((acc, cur) => {
    if (cur.sender) {
      if (cur.sender === 'USER') {
        acc.push({ question: cur.content, answer: '' })
      } else {
        if (acc.length) acc[acc.length - 1].answer = cur.content
      }
    }
    return acc
  }, [])
  const params = {
    messages: _messages
  }
  status.value = 'LOADING'
  const resultMsg = await fetchResult(params)
  // 构建result对象
  const result = {
    conversation_id: currentConversationId.value,
    sender: 'BOT',
    content: resultMsg
  }
  await addMessage(result)
  messages.value = await getMessagesByConversionId(currentConversationId.value)
  scrollToBottom()
  messages.value[messages.value.length - 1].isNew = true
  status.value = ''
}

function messageUpdate() {
  scrollToBottom()
}

function typingStopped(renderedResult) {
  const id = messages.value[messages.value.length - 1].id
  updateMessage({ id, content: renderedResult })
}

async function createConversation() {
  // 检查最新一条对话是否为空 如果没有内容则拒绝新建对话
  const firstConversationId = conversations.value[0].id
  const firstConvMessages = await getMessagesByConversionId(firstConversationId)
  if (!firstConvMessages.length) return

  const newConversationId = await addConversation({ name: generateConversationName() })
  currentConversationId.value = newConversationId
  conversations.value = await getConversationList()
}

function changeConversation(id) {
  currentConversationId.value = id
}

async function deleteConversation(id) {
  await service_deleteConversation(id)
  conversations.value = await getConversationList()

  // 如果删除后没有对话
  if (conversations.value.length === 0) {
    // 将示例对话插入对话表
    await addConversation({ name: generateConversationName() })
    // 更新对话列表
    conversations.value = await getConversationList()
  }
  if (id === currentConversationId.value) {
    currentConversationId.value = conversations.value[0].id
  }
}

async function editConversationName(id, name) {
  await service_editConversationName(id, name)
  conversations.value = await getConversationList()
}

const status = ref('')

function keydown(e) {
  if (!e.shiftKey && e.keyCode === 13) {
    e.cancelBubble = true //ie阻止冒泡行为
    e.stopPropagation() //Firefox阻止冒泡行为
    e.preventDefault() //取消事件的默认动作*换行
    if ('LOADING' === status.value) return
    sendMessage()
  }
}

// 输入推荐问题
function useRecommend(content) {
  message.value = content
}
</script>

<template>
  <el-container class="chat-page_wrapper">
    <el-aside width="280px">
      <SideBar
        :conversations="conversations"
        :currentConversationId="currentConversationId"
        @createConversation="createConversation"
        @changeConversation="changeConversation"
        @deleteConversation="deleteConversation"
        @editConversationName="editConversationName"
      />
    </el-aside>
    <el-main class="chat-main_wrapper">
      <div class="chat-container">
        <div class="chat-list_container" ref="RefChatList">
          <template v-if="messages.length">
            <div class="chat-item_container" v-for="msg in messages" :key="msg.id">
              <ChatItem
                :content="msg.content"
                :sender="msg.sender"
                :isLoading="msg.isLoading"
                :is-new="msg.isNew"
                @update="messageUpdate"
                @typingStopped="typingStopped"
              />
            </div>
          </template>
          <template v-else>
            <Welcome @useRecommend="useRecommend" />
          </template>
        </div>
        <div class="chat-input_container">
          <el-input
            class="chat-input"
            v-model="message"
            type="textarea"
            resize="none"
            :autosize="{ minRows: 2, maxRows: 4 }"
            :placeholder="$t('lang.inputPlaceholder')"
            @keydown="keydown($event)"
          />
          <div class="chat-input-btn_container">
            <el-button
              type="primary"
              class="chat-input-btn"
              @click="sendMessage"
              :disabled="!message || 'LOADING' === status"
            >
              {{ $t('lang.send') }}
            </el-button>
          </div>
        </div>
      </div>
    </el-main>
  </el-container>
</template>

<style scoped lang="less">
.chat-page_wrapper {
  background-color: @color-white-2;
  overflow: auto;

  .chat-main_wrapper {
    display: flex;
    flex-direction: column;
  }

  .chat-container {
    overflow: hidden;
    overflow-x: auto;
    position: relative;
    width: 100%;
    margin: 0 auto;
    flex: 1;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    align-items: center;

    .chat-list_container {
      width: 960px;
      margin: 0 auto;
      flex: 1;
      overflow-y: auto;
      scrollbar-width: thin;
      padding-bottom: 24px;
      //scroll-behavior: smooth;

      &::-webkit-scrollbar {
        width: 0;
      }
    }

    .chat-input_container {
      width: 980px;
      flex-shrink: 0;
      flex-grow: 0;
      position: relative;
      margin: 0 auto;

      .chat-input {
        display: flex;
        justify-content: center;
        align-items: center;
        max-width: 1135px;
        margin: 0 auto;
        box-shadow: 3px 3px 7px 1px rgba(0, 75, 174, 0.15);
        border-radius: 15px;

        :deep(.el-textarea__inner) {
          background-color: @color-white;
          border: none;
          border-radius: 15px;
          padding-right: 86px;

          &:focus {
            border: solid 1px #0070e0;
          }
        }
      }

      .chat-input-btn_container {
        align-items: center;
        display: flex;
        flex-direction: column;
        position: absolute;
        bottom: 5px;
        right: 10px;

        .chat-input-btn {
          width: 70px;
          height: 40px;
          border-radius: 25px;
          text-align: center;
          background-image: linear-gradient(90deg, @color-blue-1 0%, @color-blue-2 100%);

          &.is-disabled {
            background-image: linear-gradient(90deg, @color-grey-2 0%, @color-grey-2 100%);
            border-color: @color-white-3;
            color: @color-grey-1;
          }
        }
      }
    }
  }
}
</style>
