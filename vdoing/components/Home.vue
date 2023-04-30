<template>
  <div class="home-wrapper">

    <MainLayout>
      <template #mainLeft>
        <!-- 简约版文章列表 -->
        <UpdateArticle
          class="card-box"
          v-if="homeData.postList === 'simple'"
          :length="homeData.simplePostListLength || 10"
          :moreArticle="
            $themeConfig.updateBar && $themeConfig.updateBar.moreArticle
          "
        />

        <!-- 详情版文章列表 -->
        <template
          v-else-if="!homeData.postList || homeData.postList === 'detailed'"
        >
          <PostList :currentPage="currentPage" :perPage="perPage" />
          <Pagination
            :total="total"
            :perPage="perPage"
            :currentPage="currentPage"
            @getCurrentPage="handlePagination"
            v-show="Math.ceil(total / perPage) > 1"
          />
        </template>

        <Content class="theme-vdoing-content custom card-box" />
      </template>

      <template v-if="!homeData.hideRightBar" #mainRight>
        <BloggerBar v-if="$themeConfig.blogger" />
        <CategoriesBar
          v-if="
            $themeConfig.category !== false &&
            $categoriesAndTags.categories.length
          "
          :categoriesData="$categoriesAndTags.categories"
          :length="10"
        />
        <TagsBar
          v-if="$themeConfig.tag !== false && $categoriesAndTags.tags.length"
          :tagsData="$categoriesAndTags.tags"
          :length="30"
        />
        <div
          class="custom-html-box card-box"
          v-if="homeSidebarB"
          v-html="homeSidebarB"
        ></div>
      </template>
    </MainLayout>
  </div>
</template>

<script>
import NavLink from "@theme/components/NavLink";
import BScroll from "@better-scroll/core"
import Slide from "@better-scroll/slide"
import MainLayout from '@theme/components/MainLayout'
import PostList from '@theme/components/PostList'
import UpdateArticle from '@theme/components/UpdateArticle'
import Pagination from '@theme/components/Pagination'
import BloggerBar from '@theme/components/BloggerBar'
import CategoriesBar from '@theme/components/CategoriesBar'
import TagsBar from '@theme/components/TagsBar'

const MOBILE_DESKTOP_BREAKPOINT = 720 // refer to config.styl

BScroll.use(Slide)

export default {
  data() {
    return {
      isMQMobile: false,

      slide: null,
      currentPageIndex: 0,
      playTimer: 0,
      mark: 0,

      total: 0, // 总长
      perPage: 10, // 每页长
      currentPage: 1// 当前页
    }
  },
  computed: {
    homeData() {
      return {
        ...this.$page.frontmatter
      }
    },
    hasFeatures() {
      return !!(this.homeData.features && this.homeData.features.length)
    },
    homeSidebarB() {
      const { htmlModules } = this.$themeConfig
      return htmlModules ? htmlModules.homeSidebarB : ''
    },
    showBanner() { // 当分页不在第一页时隐藏banner栏
      return this.$route.query.p
        && this.$route.query.p != 1
        && (!this.homeData.postList || this.homeData.postList === 'detailed')
        ? false : true
    },
    bannerBgStyle() {
      let bannerBg = this.homeData.bannerBg
      if (!bannerBg || bannerBg === 'auto') { // 默认
        if (this.$themeConfig.bodyBgImg) { // 当有bodyBgImg时，不显示背景
          return ''
        } else { // 网格纹背景
          return 'background: rgb(40,40,45) url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACMAAAAjCAYAAAAe2bNZAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsMAAA7DAcdvqGQAAABOSURBVFhH7c6xCQAgDAVRR9A6E4hLu4uLiWJ7tSnuQcIvr2TRYsw3/zOGGEOMIcYQY4gxxBhiDDGGGEOMIcYQY4gxxBhiDLkx52W4Gn1tuslCtHJvL54AAAAASUVORK5CYII=)'
        }
      } else if (bannerBg === 'none') { // 无背景
        if (this.$themeConfig.bodyBgImg) {
          return ''
        } else {
          return 'background: var(--mainBg);color: var(--textColor)'
        }
      } else if (bannerBg.indexOf('background:') > -1) { // 自定义背景样式
        return bannerBg
      } else if (bannerBg.indexOf('.') > -1) { // 大图
        return `background: url(${this.$withBase(bannerBg)}) center center / cover no-repeat`
      }

    },
    actionLink() {
      return {
        link: this.homeData.actionLink,
        text: this.homeData.actionText
      };
    }
  },
  components: { NavLink, MainLayout, PostList, UpdateArticle, BloggerBar, CategoriesBar, TagsBar, Pagination },
  created() {
    this.total = this.$sortPosts.length
  },
  beforeMount() {
    this.isMQMobile = window.innerWidth < MOBILE_DESKTOP_BREAKPOINT ? true : false; // vupress在打包时不能在beforeCreate(),created()访问浏览器api（如window）
  },
  mounted() {
    if (this.$route.query.p) {
      this.currentPage = Number(this.$route.query.p)
    }

    if (this.hasFeatures && this.isMQMobile && (!this.$route.query.p || this.$route.query.p == 1)) {
      this.init()
    }

    if (this.hasFeatures) {
      window.addEventListener('resize', () => {
        this.isMQMobile = window.innerWidth < MOBILE_DESKTOP_BREAKPOINT ? true : false;
        if (this.isMQMobile && !this.slide && !this.mark) {
          this.mark++
          setTimeout(() => {
            this.init()
          }, 60)
        }
      })
    }
  },
  beforeDestroy() {
    clearTimeout(this.playTimer)
    this.slide && this.slide.destroy()
  },
  watch: {
    '$route.query.p'() {
      if (!this.$route.query.p) {
        this.currentPage = 1
      } else {
        this.currentPage = Number(this.$route.query.p)
      }

      if (this.hasFeatures && this.currentPage === 1 && this.isMQMobile) {
        setTimeout(() => {
          this.slide && this.slide.destroy()
          this.init()
        }, 0)
      }
    }
  },
  methods: {
    init() {
      clearTimeout(this.playTimer)
      this.slide = new BScroll(this.$refs.slide, {
        scrollX: true, // x轴滚动
        scrollY: false, // y轴滚动
        slide: {
          loop: true,
          threshold: 100
        },
        useTransition: true, // 使用css3 transition动画
        momentum: false,
        bounce: false, // 回弹
        stopPropagation: false, // 是否阻止事件冒泡
        probeType: 2,
        preventDefault: false
      })

      // user touches the slide area
      this.slide.on('beforeScrollStart', () => {
        clearTimeout(this.playTimer)
      })
      // user touched the slide done
      this.slide.on('scrollEnd', () => {
        this.autoGoNext()
      })
      this.slide.on('slideWillChange', (page) => {
        this.currentPageIndex = page.pageX
      })
      this.autoGoNext()
    },
    autoGoNext() {
      clearTimeout(this.playTimer)
      this.playTimer = setTimeout(() => {
        this.slide.next()
      }, 4000)
    },
    handlePagination(i) { // 分页
      this.currentPage = i
    },
    getScrollTop() {
      return window.pageYOffset
        || document.documentElement.scrollTop
        || document.body.scrollTop
    },
  },

};
</script>

<style lang="stylus" scoped>
.home-wrapper
  margin-top $navbarHeight+1rem
  
  .main-wrapper
    margin-top 2rem
    .main-left
      .card-box
        margin-bottom 2rem
      .pagination
        margin-bottom 3rem
      .theme-vdoing-content
        padding 0 2rem
        overflow hidden
        border none
        &>:first-child
          padding-top 2rem
        &>:last-child
          padding-bottom 2rem
    .main-right
      .custom-html-box
        padding 0
        overflow hidden
@keyframes heart
  from
    transform translate(0, 0)
  to
    transform translate(0, 8px)
// 1025px以下
@media (max-width 1025px)
  .home-wrapper
    .banner
      .banner-conent
        .hero
          h1
            font-size 2.5rem
          .description
            font-size 1rem
        .feature
          a
            h2
              font-size 1.1rem
            .feature-img
              width 9rem
              height 9rem
// 719px以下
@media (max-width $MQMobile)
  .home-wrapper
    .banner
      .banner-conent
        .features
          display none !important
// 419px以下
@media (max-width $MQMobileNarrow)
  .home-wrapper
    .banner-conent
      padding-left 1.5rem
      padding-right 1.5rem
      .hero
        img
          max-height 210px
          margin 2rem auto 1.2rem
        h1
          font-size 2rem
        h1, .description, .action
          margin 1.2rem auto
        .description
          font-size 1.2rem
        .action-button
          font-size 1rem
          padding 0.6rem 1.2rem
      .feature
        h2
          font-size 1.25rem
.theme-style-line
  .main-wrapper
    @media (max-width 719px)
      margin-top -1px
</style>
