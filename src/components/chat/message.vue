<template>
  <div class="messagebox" v-show="activedKey[type]!= ''">
    
      <video-player :config="playerConfig" @playerInit="onPlayerInit" ref="videoPlayer"></video-player>
      <!-- <div class="messagebox-cake"> -->
      <div class="messagebox-header">
        <div>
          <a-icon type="left" class="user-goback" v-show="broken" @click="showUserList" />
          <span>{{`${activedKey[type].name } &nbsp;&nbsp; ${activedKey[type].groupid || ''}`}}</span>
          <a-icon v-if="type=='group'" type="ellipsis" class="user-ellipsis" @click="changeMenus" />
        </div>
      </div>

      <div class="messagebox-content" ref="msgContent">
        <div class="moreMsgs" @click="loadMoreMsgs">{{loadText}}</div>
        <div
          v-for="(item,i) in msgList"
          :key="i"
          class="message-group"
          :style="{'float':item.bySelf ? 'right':'left'}"
        >
          <h4 style="text-align: left;margin:0">{{item.from}}</h4>
          <!-- 撤回消息 -->
          <div v-if="item.status == 'recall'" class="recallMsg">{{item.msg}}</div>
          <div v-if="item.status == 'recall'" class="recallMsg">{{renderTime(item.time)}}</div>
          <!-- 撤回消息 end -->
          <a-dropdown
            v-else
            :trigger="['contextmenu']"
            :style="{'float':item.bySelf ? 'right':'left'}"
            :disabled="!item.bySelf"
          >
            <span style="user-select: none">
              <!-- 图片消息 -->
              <img
                :key="item.msg"
                :src="item.msg?item.msg:''"
                v-if="item.type === 'img'"
                class="img-style"
              />
              <!-- 文件card -->
              <div
                v-else-if="item.type==='file'"
                class="file-style"
                :style="{'float':item.bySelf ? 'right':'left'}"
              >
                <el-card :body-style="{ padding: '0px' }">
                  <div style="padding: 14px;">
                    <h2>文件</h2>
                    <span>
                      <h3>{{item.filename}}</h3>
                    </span>
                    <div class="bottom clearfix">
                      <span>{{readablizeBytes(item.file_length)}}</span>
                      <a :href="item.msg" :download="item.filename">点击下载</a>
                    </div>
                  </div>
                </el-card>
              </div>
              <!-- 音频消息 -->
              <div v-else-if="item.type==='audio'" :style="{'float':item.bySelf ? 'right':'left'}">
                <audio :src="item.msg" controls></audio>
              </div>
              <!-- 视频消息 -->
              <div v-else-if="item.type==='video'">
                <video :src="item.msg" width="100%" controls></video>
              </div>
              <!-- 聊天消息 -->
              <p
                style="user-select: text"
                v-else
                v-html="renderTxt(item.msg)"
                :class="{ 'byself': item.bySelf}"
              />

            </span>

            <a-menu slot="overlay">
              <a-menu-item key="1" @click="handleCommand(item)">撤回</a-menu-item>
            </a-menu>
          </a-dropdown>

          <!-- 聊天时间 -->
          <div
            v-if="item.status !== 'recall'"
            class="time-style"
            :style="{'text-align':item.bySelf ? 'right':'left'}"
          >{{renderTime(item.time)}} {{item.bySelf?status[item.status]:''}}</div>
        </div>
      </div>
      <div class="messagebox-footer">
        <div class="footer-icon">
          <!-- 表情组件 -->
          <ChatEmoji v-on:selectEmoji="selectEmoji" :inpMessage="message" />
          <!-- 上传图片组件 -->
          <UpLoadImage :type="this.type" :chatId="activedKey[type]" />
          <!-- 上传文件组件 -->
          <UpLoadFile :type="this.type" :chatId="activedKey[type]" />
        </div>
        <div class="fotter-send">
          <a-input
            v-model="message"
            equired
            placeholder="消息"
            class="sengTxt"
            @pressEnter="onSendTextMsg"
            style="resize:none"
            ref="txtDom"
          />
          <template />
        </div>
      </div>
  </div>
</template>

<script>
import ChatEmoji from "../chatEmoji/index.vue";
import emoji from "../../config/emoji";
import UpLoadImage from "../upLoadImage/index.vue";
import UpLoadFile from "../upLoadFile/index.vue";
import "./index.less";
import { mapActions, mapGetters } from "vuex";
import moment from "moment";
import _ from "lodash";
import VideoPlayer from "../videoPlayer/videoPlayer";
let val = "";
export default {
  data() {
    return {
      activedKey: {
        contact: "",
        group: "",
        chatroom: ""
      },
      message: "",
      isHttps: window.location.protocol === "https:",
      loadText: "加载更多",
      status: {
        sending: "发送中",
        sent: "已发送",
        read: "已读"
      },
      nowIsVideo: false,
      playerConfig: {},
      rootStyle: {
        backgroundColor: "rgba(0,0,0,0.88)",
        width: 500
      },
      player: null,
      liveCdnUrl:""
    };
  },

  beforeMount() {},
  updated() {
    // console.log("数据", this.$store);
    this.scollBottom();
  },
  watch: {
    '$store.state.chat.liveCdnUrl': {
      handler (val) {
        this.liveCdnUrl = val
        this.initPlayer()
      }
    }
  },
  computed: {
    ...mapGetters({
      contact: "onGetContactUserList",
      group: "onGetGroupUserList",
      chatroom: "onGetChatroomUserList",
      msgList: "onGetCurrentChatObjMsg"
    }),
    userList() {
      return {
        contact: this.contact,
        group: this.group,
        chatroom: this.chatroom
      };
    },
    selectedKeys() {
      return [this.getKey(this.activedKey[this.type]) || ""];
    }
  },
  props: [
    "type", // 聊天类型 contact, group, chatroom
    "username", // 选中的聊天对象
    "broken", // 是否适应移动端
    "showUserList",
    "hideUserList"
  ],
  methods: {
    ...mapActions([
      "onGetContactUserList",
      "onGetGroupUserList",
      "onGetChatroomUserList",
      "onGetCurrentChatObjMsg",
      "onSendText",
      "onCallVideo",
      "onCallVoice",
      "getGroupMembers",
      "getHistoryMessage",
      "onAddBlack",
      "onDelteFirend",
      "onGetGroupinfo",
      "recallMessage",
      "onGetGroupBlack"
    ]),
    getKey(item, type) {
      let key = "";
      switch (type) {
        case "contact":
          key = item.name;
          break;
        case "group":
          key = item.groupid;
          break;
        case "chatroom":
          key = item.id;
          break;
        default:
          break;
      }
      return key;
    },
    getCurrentMsg(props) {
      this.onGetCurrentChatObjMsg({
        type: props,
        id: this.getKey(this.activedKey[props], props)
      });
    },
    select(key) {
      this.$data.activedKey[this.type] = key;
      const me = this;
      me.$data.loadText = "加载更多";
      this.initPlayer(key)

      if (this.type === "group") {
        this.$router.push({ name: this.type, params: { id: key.groupid } });
        this.onGetCurrentChatObjMsg({ type: this.type, id: key.groupid });

        setTimeout(() => {
          Vue.$store.commit("updateMessageStatus", {
            action: "oneUserReadMsgs",
            readUser: key.groupid
          });
          this.$forceUpdate();
        }, 100);

        if (!this.msgList) {
          this.getHistoryMessage({ name: key.groupid, isGroup: true });
        }
      } else if (this.type === "contact") {
        this.$router.push({ name: this.type, params: { id: key.name } });
        this.onGetCurrentChatObjMsg({ type: this.type, id: key.name });
        setTimeout(() => {
          Vue.$store.commit("updateMessageStatus", {
            action: "oneUserReadMsgs",
            readUser: key.name
          });
          this.$forceUpdate();
        }, 100);

        if (!this.msgList) {
          this.getHistoryMessage({ name: key.name, isGroup: false });
        }
      } else if (this.type === "chatroom") {
        const me = this;
        // me.roomId = key.id

        this.$router.push({ name: this.type, params: { id: key.id } });
        this.onGetCurrentChatObjMsg({ type: this.type, id: key.id });

        WebIM.conn.joinChatRoom({
          roomId: key.id, // 聊天室id
          success: function() {
            // console.log("加入聊天室成功");
            if (!me.msgList) {
              me.getHistoryMessage({ name: key.id, isGroup: true });
              setTimeout(() => {
                me.$forceUpdate();
              }, 100);
            }
          }
        });
      }
    },
    initPlayer(key){
      this.$refs.videoPlayer.destroy()
      this.playerConfig.url = this.liveCdnUrl
      this.$refs.videoPlayer.initPlayer()
    },
    loadMoreMsgs() {
      const me = this;
      const success = function(msgs) {
        if (msgs.length === 0) {
          me.$data.loadText = "已无更多数据";
        }
      };
      let name = "";
      let isGroup = false;
      if (this.type === "contact") {
        name = this.$data.activedKey[this.type].name;
      } else if (this.type === "group") {
        name = this.$data.activedKey[this.type].groupid;
        isGroup = true;
      } else if (this.type === "chatroom") {
        name = this.$data.activedKey[this.type].id;
        isGroup = true;
      }
      this.getHistoryMessage({
        name,
        isGroup,
        success
      });
    },

    changeMenus() {
      if (this.type === "contact") {
      } else if (this.type === "group") {
        this.$refs.groupInfoModel.chengeInfoModel();
        this.getGroupInfo();
      }
    },

    getGroupInfo() {
      this.onGetGroupinfo({
        select_id: this.$data.activedKey[this.type].groupid
      });
    },
    onSendTextMsg() {
      if (this.$data.message == "" || this.$data.message == "\n") {
        this.$data.message = "";
        return;
      }
      this.onSendText({
        chatType: this.type,
        chatId: this.$data.activedKey[this.type],
        message: this.$data.message
      });
      this.$data.message = "";
    },

    selectEmoji(v) {
      this.$data.message = v;
      this.$refs.txtDom.focus();
    },

    customEmoji(value) {
      return `<img src="../../../static/faces/${value}" style="width:20px"/>`;
    },

    renderTxt(txt = "") {
      let rnTxt = [];
      let match = null;
      const regex = /(\[.*?\])/g;
      let start = 0;
      let index = 0;
      while ((match = regex.exec(txt))) {
        index = match.index;
        if (index > start) {
          rnTxt.push(txt.substring(start, index));
        }
        if (match[1] in emoji.obj) {
          const v = emoji.obj[match[1]];
          rnTxt.push(this.customEmoji(v));
        } else {
          rnTxt.push(match[1]);
        }
        start = index + match[1].length;
      }
      rnTxt.push(txt.substring(start, txt.length));
      return rnTxt.toString().replace(/,/g, "");
    },

    readablizeBytes(value) {
      let s = ["Bytes", "KB", "MB", "GB", "TB", "PB"];
      let e = Math.floor(Math.log(value) / Math.log(1024));
      return (value / Math.pow(1024, Math.floor(e))).toFixed(2) + " " + s[e];
    },

    // TODO 可以抽离到utils
    renderTime(time) {
      const nowStr = new Date();
      const localStr = time ? new Date(time) : nowStr;
      const localMoment = moment(localStr);
      const localFormat = localMoment.format("MM-DD hh:mm A");
      return localFormat;
    },
    getLastMsg(item) {
      const { name, params } = this.$route;
      const chatList = this.$store.state.chat.msgList[name];
      let userId = "";
      if (name == "contact") {
        userId = item.name;
      } else if (name == "group") {
        userId = item.groupid;
      } else {
        userId = item.id;
      }
      const currentMsgs = chatList[userId] || [];
      const lastMsg = currentMsgs.length
        ? currentMsgs[currentMsgs.length - 1].msg
        : "";
      const msgTime = currentMsgs.length
        ? this.renderTime(currentMsgs[currentMsgs.length - 1].time)
        : "";
      return { lastMsg, msgTime };
    },
    scollBottom() {
      setTimeout(() => {
        const dom = this.$refs.msgContent;
        if (!dom) return;
        dom.scrollTop = dom.scrollHeight;
      }, 0);
    },
    handleCommand(item) {
      let name = "";
      if (this.type === "contact") {
        name = this.$data.activedKey[this.type].name;
      } else if (this.type === "group") {
        name = this.$data.activedKey[this.type].groupid;
      } else if (this.type === "chatroom") {
        name = this.$data.activedKey[this.type].id;
      }
      this.recallMessage({ to: name, message: item });
    },
    closeGroupMessage() {
      //退出群组或解散群组时关闭聊天框
      this.$data.activedKey["group"] = "";
    },
    closeContactMessage() {
      //删除好友时关闭当前聊天框
      this.$data.activedKey["contact"] = "";
    },
    changeIsVideoState(v) {
      v ? (this.$data.nowIsVideo = true) : (this.$data.nowIsVideo = false);
    },
    onPlayerInit(val) {
      this.player = val;
    }
  },
  components: {
    ChatEmoji,
    UpLoadImage,
    UpLoadFile,
    VideoPlayer
  }
};
</script>

<style scoped lang='less'>
.byself {
  float: right;
}
.recallMsg {
  font-size: 12px;
  color: #aaa;
  width: 100%;
  text-align: center;
}
.custom-title {
  font-weight: 500;
}
.moreMsgs {
  background: #ccc !important;
  border-radius: 8px;
  cursor: pointer;
}
.status {
  display: inline;
  position: relative;
  top: 20px;
  font-size: 12px;
  left: -6px;
  color: #736c6c;
  float: left;
}
.unreadNum {
  float: left;
  width: 100%;
}
.icon-style {
  display: inline-block;
  background-color: #f04134;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  color: white;
  line-height: 1.5;
  text-align: center;
}
.emoji-style {
  width: 22px;
  float: left;
}
.img-style {
  max-width: 350px;
}
.time-style {
  clear: both;
  margin-left: 2px;
  margin-top: 3px;
  font-size: 12px;
  color: #888c98;
}
.file-style {
  width: 240px;
  margin: 2px 2px 2px 0;
  font-size: 13px;
  h2 {
    border-bottom: 1px solid #e0e0e0;
    font-weight: 300;
    text-align: left;
  }
  h3 {
    max-width: 100%;
    font-size: 15px;
    height: 20px;
    line-height: 20px;
    font-weight: 600;
    -o-text-overflow: ellipsis;
    text-overflow: ellipsis;
    overflow: hidden;
    white-space: nowrap;
    text-align: left;
    margin-bottom: 20px;
  }
  .bottom {
    span {
      color: #999999;
      text-align: left;
    }
  }
  a {
    color: #999999;
    float: right;
    text-decoration: none;
  }
  .el-dropdown-link {
    cursor: pointer;
    color: #409eff;
  }
  .el-icon-arrow-down {
    font-size: 12px;
  }
}
</style>

