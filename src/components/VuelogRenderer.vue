<template>
  <div class="renderer">
    <transition name="loading" mode="out-in" appear>
      <vuelog-spinner class="spinner" v-if="!content" key="spinner" :pattern="config.spinnerPattern"></vuelog-spinner>
      <div class="content-body" v-if="content" key="content">
        <h1 class="content-title" v-if="type !== 'posts' && !metadata.titleless" v-text="i18nify(metadata.title)"></h1>
        <h2 class="content-title" v-if="type === 'posts'">
          <router-link :to="{name: 'post', params: {category: metadata.category, slug: metadata.slug, year: metadata.year}}" v-text="i18nify(metadata.title)"></router-link>
        </h2>
        <transition name="fade" mode="out-in" appear>
          <h4 class="content-meta" v-if="type !== 'page'" :key="active">
            <span v-text="$t(time.key, time.values)"></span>
            <span> / </span>
            <router-link :to="{name: 'category', params: {category: metadata.category}}" v-text="i18nify(metadata.categoryTitle)"></router-link>
          </h4>
        </transition>
        <!-- used in posts view -->
        <div v-if="type === 'posts'">
          <div class="content-container" v-html="content[0]"></div>
          <div class="continue-reading">
            <router-link :to="{name: 'post', params: {category: metadata.category, slug: metadata.slug, year: metadata.year}}" v-text="$t('reading.continued')"></router-link>
          </div>
        </div>
        <!-- used in page/post view -->
        <div v-if="type !== 'posts'">
          <div class="content-container">
            <transition name="fade" mode="out-in" @before-leave="closeSideMenu" @before-enter="resetScroll" appear>
              <router-view :key="routeKey" :markups="content"></router-view>
            </transition>
          </div>
          <vuelog-pagination v-if="content.length > 1" :total="content.length" :type="type"></vuelog-pagination>
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
  import marked from 'marked'
  import { meaningfulTime, retrieveByLanguage } from '../helpers'
  import hljs from '../helpers/highlight'
  import VuelogPagination from './VuelogPagination'
  import VuelogSpinner from './VuelogSpinner'

  export default {
    name: 'vuelog-renderer',

    props: ['type', 'metadata'],

    components: {
      VuelogPagination,
      VuelogSpinner
    },

    computed: {
      active () {
        return this.$store.getters.lang
      },

      config () {
        return this.$store.getters.config
      },

      languages () {
        return this.$store.getters.languages
      },

      content () {
        if (!this.contentByLang) {
          return null
        }
        if (this.contentByLang[this.active]) {
          return this.contentByLang[this.active]
        }
        return this.contentByLang['**-**']
      },

      time () {
        return meaningfulTime(this.metadata.date)
      },

      routeKey () {
        // Avoid router-view reload when routing from default match to part 1 or vise versa.
        var path = this.$route.path.replace(/\/$/, '')
        if (this.$route.name === 'page' || this.$route.name === 'post') {
          return path + '/1@' + this.active
        }
        return path + '@' + this.active
      }
    },

    data () {
      return {
        contentByLang: null,
        xhr: null
      }
    },

    methods: {
      oops () {
        this.$router.replace('/oops')
      },

      i18nify (content) {
        return retrieveByLanguage(content, this.active, this.config.lang)
      },

      closeSideMenu () {
        this.$store.dispatch('sideMenu', false)
      },

      resetScroll () {
        window.scrollTo(0, 0)
      },

      promiseRequest (method, url, header, body) {
        var self = this
        return new Promise((resolve, reject) => {
          /* global XMLHttpRequest */
          var xhr = self.xhr = new XMLHttpRequest()
          xhr.onload = () => {
            self.xhr = null
            if (xhr.status === 200) {
              resolve(xhr.responseText)
            } else {
              reject(xhr.status)
            }
          }
          xhr.onerror = () => {
            reject(xhr.status)
          }
          xhr.open(method, url, true)
          Object.keys(header || {}).forEach(key => {
            xhr.setRequestHeader(key, header[key])
          })
          xhr.send(body)
        })
      },

      loadMarkdown (path) {
        return this.promiseRequest('GET', path)
      },

      preProcess (md) {
        const metadataDelimiter = this.config.metadataDelimiter
        const metadataPosition = md.indexOf(metadataDelimiter)
        const contentMd = md.slice(metadataPosition + metadataDelimiter.length)
        var mdByLang = {}
        Object.keys(this.languages).forEach(lang => {
          const startTag = `<!-- ${lang}:+ -->`
          const endTag = `<!-- ${lang}:- -->`
          const startPosition = contentMd.indexOf(startTag)
          const endPosition = contentMd.indexOf(endTag)
          if (startPosition > -1 && endPosition > -1) {
            mdByLang[lang] = contentMd.substring(startPosition + startTag.length, endPosition).trim()
            // Default language is set for fall back
            if (this.config.lang === lang) {
              mdByLang['**-**'] = mdByLang[lang]
            }
          }
        })
        var providedLangs = Object.keys(mdByLang)
        // The content is multilingua, but none is in default language
        // Use the language that comes the first for fall back
        if (providedLangs.length > 0 && providedLangs.indexOf('**-**') === -1) {
          mdByLang['**-**'] = mdByLang[providedLangs[0]]
        }
        // The content is not multilingua
        if (Object.keys(mdByLang).length === 0) {
          mdByLang['**-**'] = contentMd.trim()
        }
        return mdByLang
      },

      renderMarkdown (mdByLang) {
        marked.setOptions({
          highlight (code, lang) {
            try {
              return hljs.highlight(lang, code).value
            } catch (e) {
              return hljs.highlightAuto(code).value
            }
          }
        })
        var markupByLang = {}
        Object.keys(mdByLang).forEach(lang => {
          markupByLang[lang] = marked(mdByLang[lang])
        })
        return markupByLang
      },

      // Ever thought of the GitHub API [Markdown](https://developer.github.com/v3/markdown/)?
      // Well, it may not be a good idea. The API will **silently** eat some tags, like <audio>, <video>. Do it at your own risk.
      // Want the GitHub look and feel too? Check out [sindresorhus/github-markdown-css](https://github.com/sindresorhus/github-markdown-css)
      renderGitHubMarkdown (md) {
        const header = { 'Accept': 'application/vnd.github.v3+json', 'Content-Type': 'application/json' }
        const body = JSON.stringify({ text: md, mode: 'markdown' })
        return this.promiseRequest('POST', 'https://api.github.com/markdown', header, body)
      },

      getDelimiterPosition (markup, delimiter) {
        if (!delimiter) {
          return markup.length
        }
        var position = markup.indexOf(delimiter)
        return position > -1 ? position : markup.length
      },

      sliceMarkup (markup) {
        if (this.type === 'posts') {
          // For posts view, if both excerptDelimiter and contentDelimiter are inserted, display the shorter part.
          var excerptPosition = this.getDelimiterPosition(markup, this.config.excerptDelimiter)
          var contentPosition = this.getDelimiterPosition(markup, this.config.contentDelimiter)
          return [markup.substring(0, Math.min(excerptPosition, contentPosition))]
        }
        // For page or post view, show part of content if contentDelimiter is inserted, otherwise full content.
        return this.config.contentDelimiter ? markup.split(this.config.contentDelimiter).filter(m => m.trim().length) : [markup]
      },

      postProcess (markupByLang) {
        var markupsByLang = {}
        Object.keys(markupByLang).forEach(lang => {
          markupsByLang[lang] = this.sliceMarkup(markupByLang[lang])
        })
        return markupsByLang
      }
    },

    created () {
      this.loadMarkdown(this.metadata.markdown)
        .then(this.preProcess)
        .then(this.renderMarkdown)
        .then(this.postProcess)
        .then(markupsByLang => {
          this.contentByLang = markupsByLang
        })
        .catch(exception => {
          if (this.type !== 'posts') {
            this.oops()
          }
        })
    },

    beforeDestroy () {
      if (this.xhr && this.xhr.readyState < 4) {
        this.xhr.abort()
      }
    }
  }
</script>

<style lang="stylus">
  @import '../assets/styles/content';
  @import '../assets/styles/highlight';
</style>

<style lang="stylus" scoped>
  .spinner
    display block
    height 48px
    width 48px
    margin 100px auto

  .loading-enter-active
  .loading-leave-active
    transition opacity .3s ease

  .loading-enter
  .loading-leave-active
    opacity 0

  .content-body
    flex 1
    display flex
    flex-direction column

  .content-title
    color #34495e
    margin-bottom 0

  .content-title a
    color #34495e
    transition color .3s ease

    &:hover
      color #4fc08d
      text-decoration none

  .content-meta
    color #7f8c8d
    margin-top .5em
    margin-bottom 0

  .content-container
    flex 1

  .continue-reading a:hover
    text-decoration none

  .fade-enter-active
  .fade-leave-active
    transition opacity .3s ease

  .fade-enter
  .fade-leave-active
    opacity 0

  @media screen and (max-width: 999px)
    h1.content-title
      font-size 1.75em

    .continue-reading a:after
      opacity 1
</style>