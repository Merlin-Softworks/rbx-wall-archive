<template>
<div>
  <div id="sidebar">
    <img :class="`groupPicker ${ gid == 85654 && 'groupPicker--active' }`" src="assets/images/nfc.png" @click="changeGid(85654)">
    <img :class="`groupPicker ${ gid == 80738 && 'groupPicker--active' }`" src="assets/images/uaf.png" @click="changeGid(80738)">
    <img :class="`groupPicker ${ gid == 18 && 'groupPicker--active' }`" src="assets/images/ucr.png" @click="changeGid(18)">
  </div>

  <h1 id="header">Roblox Wall Archive</h1>
  <h2 class="selectedGroupName">
    <a class="outsideLink" target="_blank" :href="`https://www.roblox.com/groups/${gid}`">{{ groupNames[gid] }}</a>
  </h2>

  <a :class="`resetLink ${ canResetSearch && 'resetLink--active' }`" @click="reset">Reset</a>
  <input type="text" v-model="userQuery" @keyup="page = 1" placeholder="Search for a user"/>
  <input type="text" v-model="bodyQuery" @keyup="page = 1" placeholder="Search for a specific post"/>

  <div id="resultsInfoContainer">
    <h3 v-if="numResults > 0 && !isLoading">showing {{ posts ? posts.length : 0 }} results of {{ numResults }}</h3>
    <h3 v-if="numResults == 0 && !isLoading">No results</h3>
    <button v-if="numResults > 0 && !isLoading" @click="page = 1; changeSortOrder()">Sort: {{ sortOrder == -1 ? 'New to old' : 'Old to new' }}</button>
  </div>

  <div v-if="!isLoading" v-for="post in posts" v-bind:key="post.id" v-bind:id="post.id" :class="`post ${ (userQuery || bodyQuery) && 'post--promptClick' }`" @click.self="getContext(post.id)">
    <a target="_blank" :href="`https://www.roblox.com/users/${post.poster.user.userId}/profile`">
      <img class="avatar" :src="`https://www.roblox.com/headshot-thumbnail/image?userId=${post.poster.user.userId}&width=420&height=420&format=png`">
    </a>
    <p class="postContainer">
      <a class="searchLink" @click="searchUser(post.poster.user.username)">{{ post.poster.user.username }}</a>
      <br><br>
      <span @click.self="getContext(post.id)" v-html="post.body"></span>
      <br><br>
      <span @click.self="getContext(post.id)" class="date">{{ new Date(post.created).toLocaleString('en-US') }}</span>
    </p>
  </div>

  <div v-if="isLoading" style="margin-top: 60px"></div>
  <vue-loading v-if="isLoading" type="bars" color="#75787c" :size="{ width: '50px', height: '50px' }"></vue-loading>

  <div v-if="numResults > 0 && !isLoading" id="pageNavigation">
    <h5>Page {{ page }} of {{ numPages }}</h5>
    <button @click="prevPage" v-if="page > 1">Prev Page</button>
    <button @click="nextPage" v-if="numPages > page">Next Page</button>
  </div>
</div>
</template>

<script>
  const apiOrigin = (
    window.location.hostname === 'localhost' ? 'http://localhost:3000' : 'https://rbx-wall-api.herokuapp.com'
  )

  const state = {
    sortOrder: -1,
    numResults: 0,
    userQuery: '',
    bodyQuery: '',
    isLoading: true,
    page: 1,
    gid: 85654,
    spotlightMsg: null,
    groupNames: {
      85654: 'Nightfall Clan',
      80738: 'Urban Assault Forces',
      18: 'United Clan of ROBLOX'
    }
  }

  function sanitizePostsResponse(response) {
    response.data.posts.forEach(post => {
      if (!post.poster) {
        post.poster = {
          user: {
            username: 'Banned User',
            userId: 1
          }
        }
      }
      post.body = post.body.linkify()
    })
  }

  import axios from 'axios'
  import { VueLoading } from 'vue-loading-template'

  export default {
    data () {
      return state
    },
    components: {
      VueLoading
    },
    computed: {
      numPages () {
        return Math.ceil((state.numResults/10))
      },
      canResetSearch () {
        return state.userQuery || state.bodyQuery || state.page > 1
      }
    },
    updated () {
      if (state.spotlightMsg) {
        const msg = document.getElementById(state.spotlightMsg)
        if (msg) {
          msg.scrollIntoView({behavior: 'smooth', block: 'center'})
          msg.classList.add('flash')

          setTimeout(() => {
            state.spotlightMsg = null
          }, 500)
        }
      }
    },
    asyncComputed: {
      posts () {
        const apiUrl = `${apiOrigin}/${state.gid}/${state.page}?user=${state.userQuery}&body=${state.bodyQuery}&sortOrder=${state.sortOrder}`
        state.isLoading = true

        const userQueryAtTheTime = state.userQuery
        const bodyQueryAtTheTime = state.bodyQuery

        return new Promise((resolve, reject) => {
          axios.get(apiUrl)
            .then(response => {
              if (state.userQuery === userQueryAtTheTime && state.bodyQuery === bodyQueryAtTheTime) {
                sanitizePostsResponse(response)
                state.numResults = response.data.count
                state.isLoading = false
                resolve(response.data.posts)
                mixpanel.track('Results returned')
              } else {
                reject()
              }
            })
            .catch(err => {
              reject(err)
            })
        })
      }
    },
    methods: {
      submit (e) {
        console.log(e.target.value)
      },
      changeSortOrder () {
        state.sortOrder = state.sortOrder*-1
      },
      searchUser (user) {
        state.page = 1
        state.userQuery = user
        document.body.scrollTop = document.documentElement.scrollTop = 0
      },
      prevPage () {
        if (!state.page > 1) return
        state.page--
      },
      nextPage () {
        if (!state.page < state.numPages) return
        state.page++
      },
      getContext (id) {
        if (state.userQuery == '' && state.bodyQuery == '') return

        state.isLoading = true
        const apiUrl = `${apiOrigin}/${state.gid}?getContext=${id}&sortOrder=${state.sortOrder}`

        axios.get(apiUrl)
          .then(response => {
            state.spotlightMsg = id
            state.page = response.data.page
            state.sortOrder = -1
            state.userQuery = ''
            state.bodyQuery = ''
          })
          .catch(err => {
            throw new Error(err)
          })
      },
      changeGid(gid) {
        state.gid = gid
        state.page = 1
        state.spotlightMsg = null
      },
      reset () {
        state.page = 1
        state.sortOrder = -1
        state.userQuery = ''
        state.bodyQuery = ''
        state.spotlightMsg = null
      }
    }
  }
</script>

<style lang="scss">
$lightGray: #484b51;
$medGray: #2f3136;
$darkGray: #202225;
$lightText: #cfcfcf;
$medText: #75787c;
$grayAccent: #3e4147;
$blue: #64a3d6;

html {
  font-family: Arial, Helvetica, sans-serif;
  overflow-y: overlay;
}

body {
  background: $medGray;
}

* {
  transition: all 150ms;
  color: $lightText;
}

html input:focus, html button:focus {
  outline: none;
  // box-shadow: #7dcdea52 0 0 0px 4px;
}

a {
  cursor: pointer;
  text-decoration: none;
  color: $blue;
  font-weight: bold;

  &.outsideLink::after,
  &.resetLink::after,
  &.searchLink::after {
    text-decoration: none;
    opacity: 0;
    position: relative;
    left: 6px;
    transition: all 150ms;
  }

  &.outsideLink::after {
    content: '🡥';
    bottom: 6px;
    font-size: 75%;
  }

  &.resetLink::after {
    content: '⭮';
  }

  &.searchLink::after {
    content: '+';
  }

  &.outsideLink:hover,
  &.resetLink:hover,
  &.searchLink:hover {
    &::after {
      text-decoration: none;
      opacity: 1;
    }
  }
}

a:hover {
  text-decoration: none;
}

.resetLink {
  margin: -21px auto 0 auto;
  width: 500px;
  display: block;
  text-align: right;
  opacity: 0;

  &--active {
    opacity: 1;
  }
}

h3, input, .post {
  display: block;
  margin: 10px auto;
  width: 500px;
  padding: 10px;
  border: 2px solid $grayAccent;
}

h3 {
  text-align: center;
  color: $lightText;
  border: none;
  display: inline;
}

input {
  border-radius: 5px;
  border: 1px solid $grayAccent;
  background: $lightGray;
  color: $lightText;
}

#header {
  text-align: center;
  color: $blue;
  font-weight: bold;
}

.post, .flashPost {
  border-radius: 10px;
  border-radius: 0px;
  border-width: 1px;
  border-color: transparent transparent $grayAccent transparent;
}

.post--promptClick {
  cursor: pointer;
}

.date {
  font-size: 0.7em;
}

.avatar {
  width: 75px;
  height: 75px;
  margin: 5px 20px 5px 5px;
  float: left;
  border: 1px solid $grayAccent;
  border-radius: 100%;
}

#resultsInfoContainer {
  text-align: center;
  margin: 20px 0 0 10px;
}

.postContainer {
  display: table;
}

button {
  cursor: pointer;
  position: relative;
  top: -4px;
  font-weight: bold;
  color: $lightText;
  border: 2px solid $blue;
  border-radius: 5px;
  width: 150px;
  height: 30px;
  line-height: 15px;
  background: $blue;
}

button:hover {
  color: $blue;
  background: $medGray;
}

#pageNavigation {
  text-align: center;
  margin: 20px;
}

#sidebar {
  height: 100%;
  width: 50px;
  border-right: 1px solid $grayAccent;
  position: fixed;
  top: 0;
  left: 0;
  padding: 10px 0 0 10px;
  background: $darkGray;
}

.groupPicker {
  width: 50px;
  height: 50px;
  display: block;
  margin: 10px 0;
  cursor: pointer;
  padding: 5px;
  position: relative;
  right: 10px;
}

.groupPicker:first-child  {
  margin-top: 0px;
}

.groupPicker--active {
  background-color: #2f3136;
  border-radius: 5px;
  right: 0;
}

.selectedGroupName {
  text-align: center;

  a {
    color: white;
  }
}

@keyframes flash {
  0% {
    border-color: transparent transparent $grayAccent transparent;
    background-color: transparent;
  }
  50% {
    border-color: #ffe600;
    background-color: rgba(255, 255, 0, 0.122);
  }
  100% {
    border-color: transparent transparent $grayAccent transparent;
    background-color: transparent;
  }
}

.flash {
  animation: 0.75s ease-in-out 0s 2 flash;
}
</style>

