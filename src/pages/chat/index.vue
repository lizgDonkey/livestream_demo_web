<template>
  <a-layout style="position: absolute;
	width: 100%;
	overflow: hidden;
	height: 100%;">
    <a-layout-header class="layout-header">
      <div class="header">
        <span class="setting">
          <a-dropdown>
            <span class="ant-dropdown-link" href="#">
              <a-icon type="setting" />
              <span class="username">{{userName}}</span>
            </span>
            <a-menu slot="overlay">
              <a-menu-item @click="toLogout">
                <a href="javascript:;">退出</a>
              </a-menu-item>
            </a-menu>
          </a-dropdown>
        </span>
      </div>

      <a-menu
        v-model="current"
        mode="horizontal"
        :defaultSelectedKeys="['chatroom']"
        :style="{ lineHeight: '50px', background: '#434648', color: '#fff', textAlign: 'left'}"
        @click="contactTypeChange"
      >
        <a-menu-item key="chatroom">
          <a-icon type="usergroup-add" class="navMenu-icon" />
          <span class="navMenu-text">直播聊天室</span>
        </a-menu-item>
      </a-menu>
    </a-layout-header>

    <a-layout>
      <a-layout-sider
        style="background: #fff"
        :width="broken ? '100%' : 350"
        breakpoint="lg"
        collapsedWidth="0"
        :trigger="null"
        v-model="collapsed"
        @collapse="onCollapse"
        @breakpoint="onBreakpoint"
      >
        <MessageBox :type="activeKey" :select="select" ref="messageBox" />
      </a-layout-sider>
      <a-layout-content style="overflow: visible">
        <Message
          :type="activeKey"
          :broken="broken"
          :hideUserList="hideUserList"
          :showUserList="showUserList"
          ref="messageList"
        />
      </a-layout-content>
    </a-layout>
  </a-layout>
</template>

<script>
import Vue from "vue";
import MessageBox from "../../components/chat/index.vue";
import Message from "../../components/chat/message.vue";
import "./index.less";
import { mapState, mapActions } from "vuex";
export default {
  data() {
    return {
      groupRead: false,
      contactRead: false,
      showSettingOptions: false,
      activeKey: "chatroom",
      selectedItem: "",
      showAddOptions: false,
      userName:
        localStorage.getItem("userInfo") &&
        JSON.parse(localStorage.getItem("userInfo")).userId,
      collapsed: false,
      broken: false,
      current: ["chatroom"]
    };
  },
  computed: {
    chatList() {
      return this.$store.state.chat.msgList;
    }
  },
  methods: {
    ...mapActions(["onLogout","getLiveCdnUrl"]),
    toLogout() {
      this.onLogout();
    },
    onCollapse(collapsed, type) {
      if (type != "responsive") {
        this.$data.collapsed = true;
      } else {
        this.$data.collapsed = false;
      }
    },
    onBreakpoint(broken) {
      this.$data.broken = broken;
    },
    hideUserList() {
      this.$data.collapsed = true;
    },
    showUserList() {
      this.$data.collapsed = false;
    },
    select(i) {
      let payload = {
        roomId: i.id
      };
      this.getLiveCdnUrl(payload)
      this.$refs.messageList.select(i);
      if (this.broken) {
        this.$data.collapsed = true;
      }
    },
    optionsVisibleChange() {
      this.$data.showSettingOptions = !this.$data.showSettingOptions;
    },
    contactTypeChange(type) {
      this.$data.activeKey = type.key;
      this.$router.push(`/${type.key}`);
      if (this.broken && this.collapsed) {
        this.$data.collapsed = false;
      }

      switch (type.key) {
        case "contact":
          this.$refs.messageBox.onGetContactUserList();
          break;
        case "group":
          this.$refs.messageBox.onGetGroupUserList();
          break;
        case "chatroom":
          this.$refs.messageBox.onGetChatroomUserList();
          break;
        default:
          break;
      }
      this.$refs.messageList.getCurrentMsg(type.key);
    },
    addModalChange() {
      this.$data.showAddOptions = !this.$data.showAddOptions;
    },
  },
  components: {
    MessageBox,
    Message,
  }
};
</script>
