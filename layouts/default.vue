<template>
  <v-app dark>
    <v-row v-if="!user" justify="center" align="center">
      <v-col cols="5">
        <v-text-field
          label="token"
          v-model="token"
        >
        </v-text-field>
        <v-btn
          @click="websocketConnect"
        >
          Login
        </v-btn>
      </v-col>
    </v-row>
    <template v-else>
      <v-navigation-drawer
        v-model="drawer"
        :mini-variant="miniVariant"
        :clipped="clipped"
        fixed
        app
      >
        <v-list>
          <v-list-item>Online ({{ items.length }})</v-list-item>
          <v-list-item @click="initPublicChat">Public Room</v-list-item>
          <v-list-item
            v-for="(item, i) in items"
            :key="i"
          >
            <v-list-item-content>
              <v-list-item-title @click="initPrivateChat(item)" v-text="item.username"/>
            </v-list-item-content>
          </v-list-item>
        </v-list>
      </v-navigation-drawer>
      <v-app-bar
        :clipped-left="clipped"
        fixed
        app
      >
        <v-app-bar-nav-icon @click.stop="drawer = !drawer"/>
        <v-btn
          icon
          @click.stop="miniVariant = !miniVariant"
        >
          <v-icon>mdi-{{ `chevron-${miniVariant ? 'right' : 'left'}` }}</v-icon>
        </v-btn>
        <v-btn
          icon
          @click.stop="clipped = !clipped"
        >
          <v-icon>mdi-application</v-icon>
        </v-btn>
        <v-btn
          icon
          @click.stop="fixed = !fixed"
        >
          <v-icon>mdi-minus</v-icon>
        </v-btn>
        <v-toolbar-title v-text="title"/>
        <v-spacer/>
        <v-btn
          icon
          @click.stop="rightDrawer = !rightDrawer"
        >
          <v-icon>mdi-menu</v-icon>
        </v-btn>
      </v-app-bar>
      <v-main>
        <v-container>
          <v-row justify="center" align="center">
            <span v-if="toName !==''"> Private chat with {{  toName }}</span>
            <span v-else> Public Room </span>
            <v-col cols="12" style="min-height: 400px;overflow-y: auto">
              <div v-for="message in messages">
                {{ message.username }} : {{ message.message }}
              </div>
            </v-col>
            <v-col cols="12">
              <v-text-field
                label="send your message"
                v-model="message"
              >
              </v-text-field>
              <v-btn
                @click="sendMessageToWebsocket"
              >
                Send Message
              </v-btn>
            </v-col>
          </v-row>
        </v-container>
      </v-main>
      <v-navigation-drawer
        v-model="rightDrawer"
        :right="right"
        temporary
        fixed
      >
        <v-list>
          <v-list-item @click.native="right = !right">
            <v-list-item-action>
              <v-icon light>
                mdi-repeat
              </v-icon>
            </v-list-item-action>
            <v-list-item-title>Switch drawer (click me)</v-list-item-title>
          </v-list-item>
        </v-list>
      </v-navigation-drawer>
      <v-footer
        :absolute="!fixed"
        app
      >
        <span>{{ status }}&copy; {{ new Date().getFullYear() }}</span>
      </v-footer>
    </template>
  </v-app>
</template>

<script>
export default {
  data() {
    return {
      clipped: false,
      drawer: false,
      fixed: false,
      items: [],
      miniVariant: false,
      right: true,
      rightDrawer: false,
      title: 'Vuetify.js',

      //// online code
      token: null,
      ws: null,
      status: 'not connected',
      user: null,
      message:'',
      messages:[],
      chatType:'public',
      to : 0,
      toName:'',

    }
  },
  methods: {
    websocketConnect() {
      this.ws = new WebSocket('ws://127.0.0.1:8080?token=' + this.token)
      this.ws.onopen = () => {
        this.status = 'connected'
      }
      this.ws.onmessage = (message) => {
        message = JSON.parse(message.data)
        if(message.type && message.type === 'user_data'){
          this.user = message.user
        }
        if(message.type && message.type === 'who_is_online'){
          this.items = message.clients
        }
        if(message.type && message.type === 'broadcast'){
          if(this.to === 0){
            this.messages.push(message)
          }
        }
        if(message.type && message.type === 'chat'){
          if(this.to !== message.user_id){
            this.messages = [];
            this.to = message.user_id;
            this.toName = message.username;
            this.chatType = 'private';
          }
          this.messages.push(message)
        }
      }
    },
    sendMessageToWebsocket(){
      let message = {
        user_id : this.user.user_id,
        message : this.message,
        username : this.user.username,
        type : 'broadcast'
      }
      if(this.chatType === 'private'){
          message["to"] = this.to;
          message["type"] = 'chat';
      }
      this.ws.send(JSON.stringify(message))
      this.messages.push(message)
      this.message = ''
    },
    initPrivateChat(user){
      this.messages = []
      this.to = user.user_id;
      this.toName = user.username;
      this.chatType = "private"
    },
    initPublicChat(){
      this.messages = []
      this.to = 0;
      this.toName = '';
      this.chatType = 'public'
    }
  }
}
</script>
