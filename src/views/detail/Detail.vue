<template>
  <div id="detail">
    <detail-nav-bar ref="nav" class="detail-nav" @titleClick="titleClick" />
    <scroll class="content" ref="scroll" :probe-type="3" @scroll="contentScroll">
      <detail-swiper :top-images="topImages" />
      <detail-base-info :goods="goods" />
      <detail-shop-info :shop="shop" />
      <detail-goods-info :detail-info="detailInfo" @detailImageLoad="detailImageLoad" />
      <detail-param-info ref="params" :param-info="itemParams" />
      <detail-comment-info ref="comment" :comment-info="commentInfo" />
      <goods-list ref="recommend" :goods="recommends" />
    </scroll>
    <detail-bottom-bar @addCart="addToCart" />
    <back-top @click.native="backClick" v-show="isShowBackTop" />
    <!-- <toast :show="show" :message="message" /> -->
  </div>
</template>

<script>
import DetailNavBar from './childComps/DetailNavBar'
import DetailSwiper from './childComps/DetailSwiper'
import DetailBaseInfo from './childComps/DetailBaseInfo'
import DetailShopInfo from './childComps/DetailShopInfo'
import DetailGoodsInfo from './childComps/DetailGoodsInfo'
import DetailParamInfo from './childComps/DetailParamInfo'
import DetailCommentInfo from './childComps/DetailCommentInfo'
import DetailBottomBar from './childComps/DetailBottomBar'

import Scroll from 'components/common/scroll/Scroll'
import GoodsList from 'components/content/goods/GoodsList'
// import Toast from 'components/common/toast/Toast'

import { debounce } from 'common/utils'
import { itemListenerMixin, backTopMixin } from 'common/mixin'

import {
  getDetail,
  getRecommend,
  Goods,
  Shop,
  GoodsParam
} from 'network/detail'

export default {
  name: 'Detail',
  components: {
    DetailNavBar,
    DetailSwiper,
    DetailBaseInfo,
    DetailShopInfo,
    DetailGoodsInfo,
    DetailParamInfo,
    DetailCommentInfo,
    DetailBottomBar,
    Scroll,
    GoodsList
    // Toast
  },
  mixins: [itemListenerMixin, backTopMixin],
  data() {
    return {
      iid: null,
      topImages: [],
      goods: {},
      shop: {},
      detailInfo: {},
      itemParams: {},
      commentInfo: {},
      recommends: [],
      themeTopYs: [],
      getThemeTopY: null,
      currentIndex: 0
      // message: '',
      // show: false
    }
  },
  created() {
    // 1、保存存入的iid
    this.iid = this.$route.params.iid

    // 2、根据iid请求详情数据
    getDetail(this.iid).then(res => {
      const data = res.result
      // 1、获取顶部的图片轮播数据
      this.topImages = data.itemInfo.topImages
      // 2、获取商品信息
      this.goods = new Goods(
        data.itemInfo,
        data.columns,
        data.shopInfo.services
      )
      // 3、获取店铺信息的对象
      this.shop = new Shop(data.shopInfo)
      // 4、获取商品详情
      this.detailInfo = data.detailInfo
      // 5、获取商品参数信息
      this.itemParams = new GoodsParam(
        data.itemParams.info,
        data.itemParams.rule
      )
      // 6、获取评论信息
      if (data.rate.cRate !== 0) {
        this.commentInfo = data.rate.list[0]
      }
    })
    // 3、请求推荐数据
    getRecommend().then(res => {
      this.recommends = res.data.list
    })

    // 4、给getThemeTopY赋值 对this.themeTopYs赋值进行防抖
    this.getThemeTopY = debounce(() => {
      this.themeTopYs = []
      this.themeTopYs.push(0)
      this.themeTopYs.push(this.$refs.params.$el.offsetTop - 44)
      this.themeTopYs.push(this.$refs.comment.$el.offsetTop - 44)
      this.themeTopYs.push(this.$refs.recommend.$el.offsetTop - 44)
      this.themeTopYs.push(Number.MAX_VALUE)
    }, 100)
  },
  mounted() {
    // const refresh = debounce(this.$refs.scroll.refresh, 50)
    // this.itemImgListener = () => {
    //   refresh()
    // }
    // this.$bus.$on('itemImageLoad', this.itemImgListener)
  },
  destroyed() {
    this.$bus.$off('itemImgLoad', this.itemImgListener)
  },
  methods: {
    detailImageLoad() {
      this.newRefresh()
      this.getThemeTopY()
    },
    titleClick(index) {
      this.$refs.scroll.scrollTo(0, -this.themeTopYs[index], 200)
    },
    contentScroll(position) {
      // 是否显示回到顶部
      this.isShowBackTop = -position.y > 1000
      // 1、获取y值
      const positionY = -position.y
      // 2、与主题中的值进行对比
      let length = this.themeTopYs.length
      for (let i = 0; i < length - 1; i++) {
        if (
          this.currentIndex !== i &&
          (positionY >= this.themeTopYs[i] &&
            positionY < this.themeTopYs[i + 1])
        ) {
          this.currentIndex = i
          this.$refs.nav.currentIndex = this.currentIndex
        }
        // if (
        //   this.currentIndex !== i &&
        //   ((i < length - 1 &&
        //     positionY >= this.themeTopYs[i] &&
        //     positionY < this.themeTopYs[i + 1]) ||
        //     (i === length - 1 && positionY > this.themeTopYs[i]))
        // ) {
        //   this.currentIndex = i
        //   this.$refs.nav.currentIndex = this.currentIndex
        // }
      }
    },
    addToCart() {
      // 1、获取购物车需要展示的信息
      const product = {}
      product.image = this.topImages[0]
      product.title = this.goods.title
      product.desc = this.goods.desc
      product.price = this.goods.nowPrice
      product.iid = this.iid

      // 2、将商品添加到购物车
      this.$store.dispatch('addCart', product).then(res => {
        // this.show = true
        // this.message = res

        // setTimeout(() => {
        //   this.show = false
        //   this.message = ''
        // }, 1500)
        this.$toast.show(res)
      })
    }
  }
}
</script>

<style scoped>
#detail {
  position: relative;
  z-index: 9;
  background-color: #fff;
  height: 100vh;
}
.detail-nav {
  position: relative;
  z-index: 9;
  background-color: #fff;
}
.content {
  height: calc(100% - 44px - 49px);
  overflow: hidden;
}
</style>