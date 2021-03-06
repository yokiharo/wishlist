<template>
  <q-page class="flex flex-center">

    <transition appear enter-active-class="animated fadeInDown" leave-active-class="animated fadeOutUp">
      <q-chip v-if="user" class="fixed user" :avatar="user._json.picture">
        {{ user._json.name }}
        <q-popover fit>
          <q-list separator dense link>
            <q-item v-close-overlay @click.native="logout">Logout</q-item>
          </q-list>
        </q-popover>
      </q-chip>    
      <q-btn v-else-if="loaded" icon="fas fa-user" class="fixed user" color="primary" label="Sign In" @click="show_login = true" />
    </transition>

    <transition appear enter-active-class="animated fadeIn">
      <git-hub-corner v-if="loaded"/>
    </transition>

    <transition appear enter-active-class="animated fadeInDown">
      <h1 v-if="loaded" class="title fit q-mb-sm">Wishlist</h1>
    </transition>

      <draggable tag="q-list" filter=".q-item-side, .q-input-target" :list="items" :component-data="{attrs: {noBorder: true }}" class="wishlist" v-if="items.length && loaded" @change="save_items">
        <wishlist-item v-for="(item, i) in items" :key="item.id" :item="item" :i="i" @delete="delete_item" @update="save_items"/>
      </draggable>
      <transition appear enter-active-class="animated fadeInUp">
        <h2 v-if="!items.length && loaded" class="fit q-display-1 text-weight-thin content-start">Your wishlist is empty! Click the Add button on the lower right corner or click <a href="javascript:;" style="text-decoration:none;" @click="add_item">here</a> to add a new item!</h2>
      </transition>

    <transition appear enter-active-class="animated fadeInUp">
      <q-fab v-if="loaded" class="fixed menu" color="primary" icon="menu" direction="up">
        
        <q-fab-action v-if="items.filter(i => !i.check).length" color="green" icon="check_circle_outline" @click="checking_items(true)">
          <q-tooltip :disable="$q.platform.is.mobile" anchor="center right" self="center left" :offset="[10, 0]">Check all items</q-tooltip>
        </q-fab-action>

        <q-fab-action v-if="items.filter(i => i.check).length" color="purple-13" icon="remove_circle_outline" @click="checking_items(false)">
          <q-tooltip :disable="$q.platform.is.mobile" anchor="center right" self="center left" :offset="[10, 0]">Uncheck all items</q-tooltip>
        </q-fab-action>

        <q-fab-action v-if="items.filter(i => i.check).length > 0 && items.length !== items.filter(i => i.check).length" color="orange-7" icon="delete_sweep" @click="delete_checked_items">
          <q-tooltip :disable="$q.platform.is.mobile" anchor="center right" self="center left" :offset="[10, 0]">Delete checked items</q-tooltip>
        </q-fab-action>

        <q-fab-action v-if="items.length" color="red" icon="delete_forever" @click="delete_all_items">
          <q-tooltip :disable="$q.platform.is.mobile" anchor="center right" self="center left" :offset="[10, 0]">Delete all items</q-tooltip>
        </q-fab-action>

        <install-app/>

        <q-fab-action color="info" icon="info" text-color="grey-11" @click="show_about = true">
          <q-tooltip :disable="$q.platform.is.mobile" anchor="center right" self="center left" :offset="[10, 0]">About Wishlist</q-tooltip>
        </q-fab-action>
      </q-fab>
    </transition>

    <transition appear enter-active-class="animated fadeInUp">
    <q-btn v-if="loaded" round color="primary" @click="add_item" class="fixed add-item" icon="add">
      <q-tooltip :disable="$q.platform.is.mobile" anchor="center left" self="center right" :offset="[10, 0]">New Item</q-tooltip>
    </q-btn>
    </transition>

    <q-dialog v-model="show_about">
      <span slot="title">About Wishlist</span>
      <p slot="message" class="text-weight-light">
        Wishlist is a basic list made in Quasar Framework!
        <br>
        You can get more information about Wishlist <a href="https://github.com/yokiharo/wishlist" target="_blank" style="text-decoration:none;">here</a>.
        <br><br>
        Thank you for using Wishlist!
      </p>
      <div slot="body">
        <span>1.19.05.24.1800</span>
      </div>
    </q-dialog>

    <q-dialog v-model="show_login" :ok="false">
      <span slot="title">Wishlist</span>
      <div slot="body" class="text-center q-pa-md">
        Sign in to save and load your wishlist!<br>
        <div class="q-pa-md">
          <q-btn icon="fab fa-google" label="Google" color="red" text-color="white" @click="auth_google"/><br>
        </div>
        Or continue offline, you can always sign in later<br>
        <div class="q-pa-md">
          <q-btn icon="fas fa-plug" label="Offline" color="dark" text-color="white" @click="show_login = false"/>
        </div>
      </div>
    </q-dialog>
  </q-page>
</template>

<script>
import WishlistItem from 'components/WishlistItem';
import GitHubCorner from 'components/GitHubCorner';
import InstallApp from 'components/InstallApp';

import Draggable from 'vuedraggable';

import socket from 'socket.io-client';

import { uid } from 'quasar';

export default {
  name: 'PageIndex',
  components: {
    WishlistItem,
    GitHubCorner,
    InstallApp,
    Draggable
  },
  data: () => ({
    items: [],
    loaded: false,
    show_about: false,
    user: undefined,
    show_login: false
  }),
  methods: {
    auth_google() {
      window.location.href = 'https://wishlist-quasar-api.herokuapp.com/auth/google';
    },
    logout() {
      var app = this;
      app.$axios.post('/logout').then(() => {
        app.items = [];
        app.user = undefined;
        app.$q.localStorage.remove('nunogois_wishlist');
        app.$q.localStorage.remove('nunogois_wishlist_token');
      });      
    },
    add_item() {
      this.$q.dialog({
        title: 'Wishlist',
        message: 'Please add your new item.',
        prompt: {
          model: '',
          type: 'text'
        },
        cancel: true,
        color: 'primary',
        ok: {
          label: 'Add'
        }
      }).then(item => {
        if (item !== '') {
          this.items.push({ text: item, check: false, id: uid() });
          this.save_items();
        }
      }).catch(() => {
      })
    },
    delete_item(i) {
      if (this.items[i] === undefined)
        this.items = [];
      else
        this.items.splice(i, 1);

      this.save_items();
    },
    save_items() {
      this.$q.localStorage.set('nunogois_wishlist', { items: this.items, updated: new Date().toISOString() });
      if (this.user)
        this.$axios.post('/save', { items: this.items });
    },
    checking_items(condition) {
      this.items.forEach(item => {
        item.check = condition;
      });
      this.save_items();
    },
    delete_checked_items() {
      var app = this;
      this.$q.dialog({
        title: 'Wishlist',
        message: 'Are you sure you wish to delete all checked items?',
        cancel: true,
        color: 'primary',
        ok: {
          label: 'Delete',
          color: 'negative'
        }
      }).then(() => {
        let new_app_items = [];
        app.items.forEach(item => {
          if (!item.check)
            new_app_items.push(item);
        });
        app.items = new_app_items;
        app.save_items();
      }).catch(() => {

      })
    },
    delete_all_items() {
      this.$q.dialog({
        title: 'Wishlist',
        message: 'Are you sure you wish to delete all items?',
        cancel: true,
        color: 'primary',
        ok: {
          label: 'Delete',
          color: 'negative'
        }
      }).then(() => {
        this.items = [];
        this.save_items();
      }).catch(() => {

      })
    }
  },
  mounted () {
    var app = this;
    if (!app.loaded) {
      window.addEventListener('keyup', function(e) {
        var originalElement = e.srcElement || e.originalTarget;
        if (e.keyCode === 13 && originalElement.tagName === 'BODY')
          app.add_item();
      })

      var offline_list = app.$q.localStorage.get.item('nunogois_wishlist');
      if (offline_list !== null) {
        if (!offline_list.updated) { // is this a old list? let's upgrade it to a new list that saves the save date
          offline_list = { items: offline_list, updated: new Date().toISOString() };
          app.$q.localStorage.set('nunogois_wishlist', offline_list);
        }
        
        app.items = offline_list.items;
      }

      var token;
      if (app.$route.query.token) {
        token = app.$route.query.token;
        app.$router.replace('/');
        app.$q.localStorage.set('nunogois_wishlist_token', token);
      }

      if (!token)
        token = app.$q.localStorage.get.item('nunogois_wishlist_token');

      if (token && token !== null) {
        const io = socket.connect('https://wishlist-quasar-api.herokuapp.com', { query: 'token=' + token });
        io.on('save', (items) => {
          app.items = items;
        })

        app.$axios.defaults.headers.common['Authorization'] = 'Bearer ' + token;
        app.$axios.get('/load').then((response) => {
          app.user = response.data.user;
          var user_list = response.data.user_list;
          if (user_list !== null && (offline_list === null || new Date(user_list.updated) > new Date(offline_list.updated))) {
            app.items = user_list.items;
            this.$q.localStorage.set('nunogois_wishlist', { items: app.items, updated: new Date().toISOString() });
          }

          app.loaded = true;
        }).catch((e) => {
          app.$q.localStorage.remove('nunogois_wishlist_token');
        })
      }
      else {
        if (!app.user)
          app.show_login = true;

        app.loaded = true;
      }
    }
  }
}
</script>

<style lang="stylus">
.title
  font-family 'Dancing Script', cursive;

.wishlist
  width 100%
  max-width 500px
  transition flex 0.3s ease-out
  flex 1

.menu
  font-size 20px
  left 18px
  bottom 18px

.add-item
  font-size 20px
  right 18px
  bottom 18px

.user
  top 18px
  left 18px
  cursor pointer

.q-layout-page
  padding-bottom 80px
  text-align center

.q-item+.q-item:before
    content ""
    height 1px
    background linear-gradient(to right, rgba(0, 0, 0, 0) 0%, rgba(217, 217, 217, 1) 50%, rgba(0, 0, 0, 0) 100%)
    position absolute
    width 100%
    top 0
    left 0

@media (min-width: 600px)
  .menu
    font-size 34px
    left 38px
    bottom 38px

  .add-item
    font-size 34px
    right 38px
    bottom 38px
    
  .q-layout-page
    padding-bottom 160px
</style>