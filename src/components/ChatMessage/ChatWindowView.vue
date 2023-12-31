<script>
import axios from 'axios'
import EmojiPicker from 'vue3-emoji-picker'
import 'vue3-emoji-picker/css'

import MessageView from '@/components/ChatMessage/MessageView.vue'
import { normalizeMsgDate } from '@/utils'


export default {
  name: 'ChatWindowView',
  data() {
    return {
      userInfo: null,
      inputMessageValue: '',
      chatObj: null,
      chatType: null,
      isInterlocutorOnline: false,
      isInterlocutorTyping: false,

      webSocket: null,

      allMessageCount: 0,
      messageLimit: 15,
      messageOffset: 0,
      isMessageLoading: false,
      showSearchMessageInput: false,
      messageSearchValue: '',
      isFetchingMessages: false,
      msgLstDiv: null,
      messages: [],

      amITyping: false,
      typingTimeOut: null,
      showEmojiPicker: false,
    }
  },
  components: {
    MessageView,
    EmojiPicker,
  },
  methods: {
    handleSendMessage(event) {
      event.preventDefault();
      if (this.inputMessageValue.trim() === '') return;
      if (this.chatType === "PRIVATE" || this.chatType === "GROUP") {
        this.handleSocketSendChatMessage()
      } else if (this.chatType === "CHANNEL") {
        this.handleSocketSendChatMessage();
      } else return

      // this.$refs.messageInput.focus()
      this.inputMessageValue = ''
    },
    handleSocketSendChatMessage() {

      let now = new Date()
      var msg = {
        "id": 0,
        "content": this.inputMessageValue,
        "is_own_message": true,
        "created_at": now.toLocaleDateString("uz", { year: "numeric", month: "numeric", day: "2-digit" }) + " " + now.toLocaleTimeString("uz-UZ", { hour: "2-digit", minute: "2-digit", second: "2-digit" })
      }
      this.messages.unshift(msg)

      const message = {
        EVENT_TYPE: "private_chat_send_message",
        receiver_id: this.chatObj.chat.user ? this.chatObj.chat.user.id : null,
        message_type: "TEXT",
        message_text: this.inputMessageValue
      }

      if (this.webSocket.readyState == WebSocket.OPEN) {
        this.webSocket.send(JSON.stringify(message))
      }
    },
    createWebsocketConnection(chatId) {
      const access = this.$store.state.access
      const socketUrl = process.env.VUE_APP_BASE_WEBSOCKET_URL + "/chat/" + chatId + "/?token=" + access
      this.webSocket = new WebSocket(socketUrl)

      this.webSocket.addEventListener("open", this.handleSocketOpen)
      this.webSocket.addEventListener("message", this.handleSocketMessage)
      this.webSocket.addEventListener("close", this.handleSocketClose)
      this.webSocket.addEventListener("error", this.handleSocketError)
    },
    closeOldConnection(oldId) {
      console.log(oldId)
      if (this.webSocket) {
        this.webSocket.close()
      }
      this.webSocket = null
    },
    checkUserOnlineStatus(userId) {
      const message = {
        EVENT_TYPE: "check_private_chat_user_online",
        user_id: userId,
      }
      if (this.webSocket.readyState == WebSocket.OPEN) {
        this.webSocket.send(JSON.stringify(message))
      }
    },
    NotifyUserTypingStatus() { },
    handleSocketOpen() {
      if (this.chatObj && this.chatType === 'PRIVATE') {
        this.checkUserOnlineStatus(this.chatObj.chat.user.id)
      }
    },
    handleSocketMessage(event) {

      try {
        const event_data = JSON.parse(event.data)
        const event_type = event_data.EVENT_TYPE

        if (event_type == "private_chat_online_status") {
          const user_id = event_data.user_id

          if (user_id == this.chatObj.chat.user.id) {
            this.isInterlocutorOnline = event_data.is_online
          }
        } else if (event_type == "private_chat_send_message") {
          const message = event_data.message

          if (message.sender.id == this.$store.state.user.id) {
            message.is_own_message = true;
            for (let i = 0; i < this.messages.length; i++) {
              if (!this.messages[i].id && this.messages[i].content === message.content) {
                this.messages[i] = message;
                break;
              }
            }
          } else {
            message.is_own_message = false;
            this.messages.unshift(message);
          }

          this.scrollToLastMessage()

          if (event_data.message.sender.id === this.$store.state.user.id) {
            // this.$refs.messageAudio.play()
            console.log()
          } else {
            // this.$refs.messageReceiveAudio.play()
            console.log()
          }
        } else if (event_type == "private_chat_user_typing_status") {
          if (event_data.user_id !== this.$store.state.user.id) {
            this.isInterlocutorTyping = event_data.is_typing
          }
        } else if (event_type == "private_chat_edit_message") {
          for (let i = 0; i < this.messages.length; i++) {
            if (this.messages[i].id === event_data.message.id) {
              this.messages[i] = event_data.message;
              if (this.messages[i].sender.id === this.$store.state.user.id) {
                this.messages[i].is_own_message = true;
              } else {
                this.messages[i].is_own_message = false;
              }
              break;
            }
          }
        } else if (event_type == "private_chat_message_delete") {
          this.messages = this.messages.filter(message => message.id !== event_data.msg_id)
        } else if (event_type == "private_chat_see_message") {
          setTimeout(() => {
            for (let i = 0; i < this.messages.length; i++) {
              if (event_data.message && this.messages[i].id === event_data.message.id) {
                this.messages[i] === event_data.message;
              }
            }
          }, 500)
        }
      } catch {
        console.log("ERROR PARSING WEBSOCKET MESSAGE!")
      }

    },
    handleSocketClose(event) {
      console.log(event)
    },
    handleSocketError(event) {
      console.log(event)
    },
    fetchChat(chatId) {
      axios.get(
        "chat/chatDetail/" + chatId + "/"
      )
        .then(response => {
          if (response.status == 200) {
            return response.data
          }
        })
        .then(data => {
          this.chatObj = data;
          this.chatType = this.chatObj.chat.type;
        })
        .catch(error => {
          console.log(error)
        })
    },
    fetchMessages(chatId, loadMore) {
      axios.get(
        "/chat/" + chatId + "/messages/?limit=" + this.messageLimit + "&offset=" + this.messageOffset + "&search=" + this.messageSearchValue,
      )
        .then(response => {
          if (response.status === 200) {
            return response.data
          }
        })
        .then(data => {
          if (loadMore) {
            this.messages.push(...data.results);
          } else {
            this.messages = data.results;
          }
          this.allMessageCount = data.count
        })
        .catch(error => {
          console.log(error)
        })
    },
    handleMessageSearch() {
      this.messageLimit = 15;
      this.messageOffset = 0;
      this.fetchMessages(this.$route.params.id, false);
    },
    normalizeMsgDate,
    handleTyping(e) {
      e.preventDefault();
      this.startTyping()

      if (this.typingTimeOut) {
        clearTimeout(this.typingTimeOut)
      }
      this.typingTimeOut = setTimeout(() => {
        this.stopTyping()
      }, 1000)  // 1s
    },
    startTyping() {
      if (this.amITyping) return

      const message = {
        EVENT_TYPE: "private_chat_user_typing_status",
        user_id: this.$store.state.user.id,
        is_typing: true
      }
      if (this.webSocket.readyState == WebSocket.OPEN) {
        this.webSocket.send(JSON.stringify(message))
      }
      this.amITyping = true
    },
    stopTyping() {
      this.amITyping = false
      const message = {
        EVENT_TYPE: "private_chat_user_typing_status",
        user_id: this.$store.state.user.id,
        is_typing: false,
      }
      if (this.webSocket.readyState == WebSocket.OPEN) {
        this.webSocket.send(JSON.stringify(message))
      }
    },
    loadMore(msgLst) {
      if (-msgLst.scrollTop + msgLst.clientHeight + 1 >= msgLst.scrollHeight) {
        if (this.isFetchingMessages) {
          return
        } else {
          this.messageOffset = this.messageOffset + this.messageLimit
          if (this.messageOffset >= this.allMessageCount) {
            return
          }
        }
        this.isFetchingMessages = true
        this.isMessageLoading = true
        setTimeout(() => {
          this.fetchMessages(this.$route.params.id, true)
          this.isMessageLoading = false
          this.isFetchingMessages = false
        }, 1000)
      }
    },
    handleBackToChats() {
      this.$emit('backToChats')
    },
    scrollToLastMessage() {
      this.$refs.msgLstDiv.scrollTop = this.$refs.msgLstDiv.scrollHeight
    },
    openUserProfile() {
      this.$store.commit("toggleUserProfile", this.chatObj.chat.user.id);

      // this.$emit("openProfile", this.chatObj.chat.user.id)
    },
    editMessage(msgId, newMessageContent) {
      const message = {
        EVENT_TYPE: "private_chat_edit_message",
        message_id: msgId,
        message_text: newMessageContent,
      }

      if (this.webSocket.readyState == WebSocket.OPEN) {
        this.webSocket.send(JSON.stringify(message))
      }
    },
    seeMessage(msgId) {
      const message = {
        EVENT_TYPE: "private_chat_see_message",
        message_id: msgId,
      }

      if (this.webSocket.readyState == WebSocket.OPEN) {
        this.webSocket.send(JSON.stringify(message))
      }
    },
    deleteMessage(msgId) {
      const message = {
        EVENT_TYPE: "private_chat_message_delete",
        message_id: msgId,
      }

      if (this.webSocket.readyState == WebSocket.OPEN) {
        this.webSocket.send(JSON.stringify(message))
      }
    },
    onSelectEmoji(emoji) {
      this.inputMessageValue = this.inputMessageValue + emoji.i
      // this.$refs.messageInput.focus()
    },
    toggleEmojiPicker() {
      this.showEmojiPicker = !this.showEmojiPicker;
    }
  },
  created() {
    this.$watch(
      () => this.$route.params,
      (toParams, previousParams) => {
        this.closeOldConnection(previousParams.id);
        this.messageOffset = 0;
        this.allMessageCount = 0;
        this.messages = [];
        this.isInterlocutorOnline = false;
        this.isInterlocutorTyping = false;

        if (toParams) {
          this.fetchChat(this.$route.params.id);
          this.fetchMessages(toParams.id, false);
          this.createWebsocketConnection(toParams.id);
        }
      }
    )
  },
  beforeMount() {
    this.fetchChat(this.$route.params.id);
  },
  mounted() {
    this.userInfo = this.$store.state.user;

    this.fetchMessages(this.$route.params.id, true)
    this.createWebsocketConnection(this.$route.params.id)

    setTimeout(() => {
      const msgLst = this.$refs.msgLstDiv
      if (msgLst) {
        msgLst.addEventListener("scroll", () => this.loadMore(msgLst))
      }
    }, 1000)
  },
  beforeUnmount() {
    if (this.webSocket) {
      this.webSocket.close();
    }
  }
}
</script>


<template>
  <div id="chat-area" class="is-flex is-flex-direction-column">

    <div class="chat-profile-header is-flex px-3">
      <div class="is-flex is-justify-content-space-between" style="width: 100%">
        <div class="is-flex">
          <div class="is-flex">
            <div class="p-3 mr-2">
              <i @click="$router.push('/')" class="fa-solid fa-arrow-left is-cursor-pointable"></i>
            </div>

            <figure class="image is-48x48 is-cursor-pointable" @click="openUserProfile">
              <img v-if="chatType == 'PRIVATE'" class="is-rounded"
                :src="chatObj.chat.user.avatar ? chatObj.chat.user.avatar : '/default_avatar.png'">
              <img v-else-if="chatType == 'GROUP'" class="is-rounded"
                :src="chatObj.chat.image ? chatObj.chat.image : '/default_group_avatar.svg'">
              <img v-else-if="chatType == 'CHANNEL'" class="is-rounded"
                :src="chatObj.chat.image ? chatObj.chat.image : '/channel_default_avatar.svg'">
            </figure>

          </div>

          <div class="has-text-left px-4">
            <h4 class="is-size-6 is-cursor-pointable" v-if="chatType == 'PRIVATE'" @click="openUserProfile">
              {{ chatObj.chat.user.first_name }} {{ chatObj.chat.user.last_name }}
            </h4>

            <h4 class="is-size-6 is-cursor-pointable" v-else-if="chatType === 'GROUP' || chatType === 'CHANNEL'">
              {{ chatObj.chat.name }}
            </h4>

            <div v-if="chatType === 'PRIVATE'">
              <p class="is-size-7 has-text-info typing-text" v-if="isInterlocutorOnline && isInterlocutorTyping">
                typing ...
              </p>
              <p class="is-size-7 has-text-success" v-else-if="isInterlocutorOnline">online</p>
              <p class="is-size-7" v-else>last seen at {{
                normalizeMsgDate(chatObj.chat.user.last_seen_at) }}</p>
            </div>
            <div v-else-if="chatType === 'GROUP' || chatType === 'CHANNEL'">
              <p class="is-size-7">312 members</p>
            </div>

          </div>
        </div>

        <div class="is-flex">
          <div class="py-2 px-3">
            <div v-if="showSearchMessageInput" class="messageSearch control has-icons-right">
              <input type="text" class="input is-small" v-model="messageSearchValue" @input="handleMessageSearch"
                placeholder="Search messages ..." />
              <span class="icon is-right">
                <i class="fa-solid fa-xmark is-clickable has-text-dark"
                  @click="showSearchMessageInput = !showSearchMessageInput"></i>
              </span>
            </div>
            <i class="is-size-7 fas fa-search is-cursor-pointable is-clickable"
              @click="showSearchMessageInput = !showSearchMessageInput"></i>
          </div>
          <div class="py-2 px-3">
            <i class="is-size-7 fa-solid fa-ellipsis-vertical is-cursor-pointable"></i>
          </div>
        </div>
      </div>
    </div>

    <div class="chat-container is-flex is-flex-direction-column is-justify-content-space-between">
      <div class="chat-body is-relative">
        <div v-if="isMessageLoading" class="load-more">
          <i class="fa-solid fa-spinner"></i>
        </div>

        <div v-if="chatObj" id="msgLstDiv" class="message-list is-flex is-flex-direction-column-reverse" ref="msgLstDiv">
          <MessageView v-for="(message, index) in messages" :key="index" :message="message" :chatObj="chatObj"
            @editMessage="editMessage" @deleteMessage="deleteMessage" @seeMessage="seeMessage" />
        </div>
      </div>

      <div v-if="chatType === 'PRIVATE' || chatType === 'GROUP' || chatType === 'CHANNEL' && chatObj.chat.is_own_channel"
        class="message-input-container is-flex is-flex-direction-column">
        <form @submit.prevent="handleSendMessage">

          <div class="control has-icons-left has-icons-right pb-3">
            <input ref="messageInput" class="input is-medium is-rounded" type="text" spellcheck="false"
              placeholder="Send message ..." v-model="inputMessageValue" @input="handleTyping"
              style="width: 100% !important;" />
            <span class="icon is-small is-left">
              <EmojiPicker v-if="showEmojiPicker" class="emoji-picker is-clickable" :native="true"
                @select="onSelectEmoji" />
              <i class="messageIcon is-clickable has-text-dark far fa-smile is-size-5 is-size-6-mobile"
                @click="toggleEmojiPicker"></i>
            </span>
            <span class="icon is-small is-right">
              <i class="messageIcon far fa-paper-plane is-size-5 is-size-6-mobile"
                :class="{ 'has-text-light disabled': !inputMessageValue, 'has-text-success is-clickable': inputMessageValue }"
                @click="handleSendMessage"></i>
            </span>

            <audio hidden="true" ref="messageAudio">
              <source src="../../assets/audio/notification-sound-7062-pixabay.mp3" type="audio/mpeg">
            </audio>
            <audio hidden="true" ref="messageReceiveAudio">
              <source src="../../assets/audio/whatsapp_message_sent.mp3" type="audio/mpeg">
            </audio>
          </div>

        </form>

      </div>

    </div>
  </div>
</template>

<style>
#chat-area {
  height: 100%;
}

.chat-container {
  height: calc(100% - 48px);
}

.chat-body {
  flex: 1;
  height: calc(100% - 74px);
}

#msgLstDiv {
  height: 100%;
  overflow-y: scroll;
}

/* ::-webkit-scrollbar {
  display: none;
} */

@keyframes typingAnimation {
  0% {
    color: red;
  }

  33% {
    color: aqua;
  }

  66% {
    color: blue;
  }

  100% {
    color: yellowgreen;

  }
}

form {
  position: sticky;
  bottom: 0;
}

.emoji-picker {
  position: absolute;
  bottom: 3em;
  left: 1em;
}

.messageIcon {
  color: dark !important;
  pointer-events: all;
}

.messageIcon:hover {
  cursor: pointer;
  color: none;
}

.messageSearch {
  position: absolute !important;
  right: 2em;
  top: 3em;
  z-index: 100;
}
</style>
