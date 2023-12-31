<script>
import axios from 'axios';
import ChatListBlockView from '@/components/Chat/ChatListBlockView.vue'


export default {
  name: 'SidebarView',
  data() {
    return {
      isArchivedOpen: false,
      isArchiveButtonLoading: false,
      selectedChatId: 0,
      searchInput: '',
      chatLimit: 15,
      chatOffset: 0,
      allChatsCount: 0,
      chats: [],
    }
  },
  props: ["myId"],
  components: {
    ChatListBlockView,
  },
  methods: {
    toggleArchived() {
      this.isArchivedOpen = !this.isArchivedOpen;
      this.chatOffset = 0;
      this.fetchChats();
    },
    handleChatSearch() {
      this.fetchChats()
    },
    chatSelected(chatId) {
      this.selectedChatId = chatId
      this.$emit("selectedChat", chatId)
    },
    fetchChats(loadMore = false) {

      this.isArchiveButtonLoading = true;

      axios.get(
        "chat/chatList/",
        {
          params: {
            limit: this.chatLimit,
            offset: this.chatOffset,
            is_archived: this.isArchivedOpen,
            search: this.searchInput,
          }
        }
      )
        .then(response => {
          if (response.status == 200) {
            return response.data
          }
        })
        .then(data => {
          if (loadMore) {
            this.chats.push(...data.results)
          } else {
            this.chats = data.results
            this.allChatsCount = data.count;
          }

          setTimeout(() => {
            this.isArchiveButtonLoading = false;
          }, 50)
        })
        .catch(error => {
          console.log("ERROR: ", error)
        })

    },
    handleChatListScroll(e) {

      let scrollTop = e.target.scrollTop;
      let clientHeight = e.target.clientHeight;
      let scrollHeight = e.target.scrollHeight;

      if (scrollTop + clientHeight >= scrollHeight) {

        if (this.chatOffset >= this.allChatsCount) {
          return
        }

        this.chatOffset = this.chatOffset + this.chatLimit;
        this.fetchChats(true);
      }
    },
  },
  mounted() {
    this.fetchChats();
  }
}
</script>


<template>
  <div class="sidebar-container">
    <div class="field searchbar">
      <p class="control has-icons-left is-size-6">
        <input class="input is-size-6" type="text" placeholder="Search" v-model="searchInput" spellcheck="false"
          @input="handleChatSearch" />
        <span class="icon is-small is-left">
          <i class="fa-solid fa-magnifying-glass"></i>
        </span>
      </p>
    </div>

    <div class="chats-container">
      <div id="chat-list" ref="chatListContainer" @scroll="handleChatListScroll">
        <button class="archive-button button is-medium is-danger" :class="{ 'is-loading': isArchiveButtonLoading }"
          @click="toggleArchived" v-if="isArchivedOpen">
          <p class="is-size-7">close</p>
        </button>

        <button v-else class="archive-button button is-medium has-background-grey"
          :class="{ 'is-loading': isArchiveButtonLoading }" @click="toggleArchived">
          <div v-if="!isArchiveButtonLoading" class="px-4 is-flex is-justify-content-center">
            <div
              class="is-size-7-mobile has-background-light has-text-grey is-flex is-justify-content-center is-align-items-center"
              style="width: 30px; height: 30px; border-radius: 50%;">
              <i class="fa-solid fa-box-archive"></i>
            </div>
          </div>
          <div v-if="!isArchiveButtonLoading" class="is-size-7 has-text-white">
            <p class="has-text-center">Archived conversations</p>
            <p class="has-text-center">2 conversations</p>
          </div>
        </button>

        <div v-for="chat in chats" :key="chat.chat.id">
          <router-link :to="{ 'name': 'chatWindow', 'params': { 'id': chat.chat.id } }" class="has-text-dark">
            <ChatListBlockView :chatObj="chat" :myId="myId" />
          </router-link>
        </div>

        <div v-if="!chats.length" class="p-5">No chats</div>
      </div>

    </div>
  </div>
</template>

<style scoped>
.sidebar-container {
  width: 100% !important;
  height: 100% !important;
}

.searchbar {
  min-height: 40px;
  margin: 0 !important;
}

.chats-container {
  height: calc(100% - 40px);
  margin: 0 !important;
}

.archive-button {
  width: 100%;
}

#chat-list {
  width: 100% !important;
  margin: 0;
  height: 100%;
  overflow-y: scroll;
}

@media screen and (min-width: 1024px) {
  #chat-list {
    height: 93%;
  }
}

.router-link-exact-active .chat-block {
  background: rgb(226, 225, 225) !important;
}
</style>
