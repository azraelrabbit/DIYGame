<template>
  <div id="app">
    <header>
      <nav>
        <router-link to="/">{{lang===2?"Home":"主页"}}</router-link>
        <router-link to="/previewPage">{{lang===2?"Example":"实例"}}</router-link>
        <router-link to="/entireGame">{{lang===2?"Console":"设计总台"}}</router-link>
        <router-link to="/help">{{lang===2?"Guide":"帮助"}}</router-link>
        <a href="https://github.com/Lawliet01/diyGame" target='_blank'>GITHUB</a>
        <a @click="changeLang(2)" :class="lang===2?'selected':'unselected'">En</a>
        |<a @click="changeLang(1)" :class="lang===1?'selected':'unselected'">中文</a>
      </nav>
    </header>
    <transition name="pageTransition">
      <router-view/>
    </transition>
  </div>
</template>

<script>
// import { mapState, mapMutations} from "vuex";

export default {
  computed:{
    lang(){
      return this.$store.state.lang
    }
  },
  created(){
    let date = (new Date()).toString()
    let inChina = /CST|China/.test(date)
    if (inChina) {
      this.changeLang(1)
    }

  },
  methods:{
    changeLang(lang){
      this.$store.commit('changeLanguage',lang )
    }
  }
}

</script>
<style lang="scss">
@import "./lib/rangeInputStyle.css";

* {
  box-sizing: border-box;
}

//动画效果
.pageTransition-enter-active {
  transition: all 0.06s;
}
.pageTransition-enter {
  transform: scale(0);
}

html {
  background-color: #f8f9fa;
  max-width:100%;
  overflow-x:hidden;
}
body {
  margin: 0;
  max-width:100%;
  overflow-x:hidden;
}
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
header {
  text-align: left;
  $navHeight: 50px;
  padding: 10px;
  &::before {
    content: "DIYGAME";
    line-height: $navHeight;
    font-weight: bold;
    font-size: 30px;
    padding: 0px 20px;
  }

  nav {
    float:right;
    a {
      text-decoration: none;
      line-height: $navHeight;
      font-size: 15px;
      margin: 0px 15px;
      color: #3eb0dd;
      font-weight: 600;
      &.router-link-exact-active {
        color: #2c3e50;
      }
      &:hover {
        text-decoration: underline;
      }
    }

    .selected {
      margin: 5px;
      // @extend a;

      color: #2c3e50;
    }

    .unselected {
      margin: 5px;
      &:hover {
        // text-decoration: underline;
        cursor: pointer;
      }
    }
  }
  
}

@media screen and (max-width: 550px) {
  header{
    text-align: center;
    &::before{
      position: absolute;
      content:''
    }
    nav{
      display: block;
      float:none;
      margin: auto;
    }
  }
}

</style>
