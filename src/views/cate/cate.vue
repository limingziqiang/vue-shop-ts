<template lang="pug">
  .tabbar-page
    //- 导航栏
    Nav(tabbar title="分类")
    //- 接口调用失败
    Abnormal(v-if="errMsg" :err="errMsg" tabbar @callback="getCategories")
    //- 主体
    .c-main(v-else)
      //- 搜索栏
      .c-search(:style="`top: ${navHeight}px`")
        .e-bgGray.e-flex_left.e-c9
          span.icon.icon-search.e-font20
          ul: li.e-font12 你好
      //- 左侧分类栏
      ul#menu.c-left.scroll-view(@toucmove.stop.prevent="")
        li.e-hidden(v-for="(item, index) in cateList" :key="index" @click="selectCates(index, item.id)")
          p.c-left_item(:class="{ on: selectCateID === item.id }")
            span.ctx {{ item.name }}
      //- 右侧商品橱窗
      .c-right.scroll-view(@toucmove.stop.prevent="")
        //- loading页
        .loading-page(v-if="loadingStatus"): spinner(:type="1" color="#E50112")
        //- 无分类情况
        Abnormal(v-else-if="noCateGoods" err="该分类下暂无商品哦~" icon="ico-state cate" :nav="false" :btn="false")
        //- 分类列表
        .c-right_wrapper(v-else)
          .e-flex.e-flex_center
            .c-right_wrapper_product(v-for="(item, index) in productInfoList" :key="index" :class="{ last: index > productInfoList.length - 3 }")
              proTemplate(direction="vertical" :brief="false" size="small" :info="item")
    //- tabbar
    tabbar(selected="cate")
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import { namespace } from 'vuex-class'
import tabbar from '@/components/tabbar.vue'
import proTemplate from '@/components/pro-template.vue'
import spinner from '@/components/spinner/spinner.vue'
import goodsApi from '@/api/goods'

// 声明文件
type productListOptions = {
  brief: string;
  has_store: number;
  image: string;
  name: string;
  price: string;
  product_id: string;
  goods_id: string;
}

const userModule = namespace('user')
const appModule = namespace('app')

@Component({
  components: {
    tabbar,
    proTemplate,
    spinner
  }
})
export default class Cate extends Vue {
  cateList: {id: number; name: string}[] = []
  selectCateID = 0 // 选中的分类项请求ID
  scrollTop = 0 // 左侧滚动值
  rightScrollTop = 0 // 右侧滚动值
  startScrollIndex = 6 // 开始需要自动滚动的分类下标
  scrollLock = false // 右侧的滚动锁
  upperTipFlag = false // 下拉提示的控制开关
  loadingStatus = 0 // 右侧加载开关
  productInfoList: productListOptions[] = []
  noSubCateFlag = false // 无二级分类情况
  noCateGoods = false // 无分类商品情况
  errMsg = '' // 加载失败文案

  @userModule.State cateId!: number
  @appModule.State navHeight!: number

  @userModule.Mutation('SET_CATE_ID') setCartId!: Function

  created () {
    this.getCategories()

    if (this.cateId > 0 && this.cateId !== this.selectCateID) {
      this.selectCateID = this.cateId
      this.getGoods()
    }
  }

  // 获取分类数据
  getCategories () {
    goodsApi.getCategories().then(res => {
      this.errMsg = ''
      this.cateList = res.data
      // 初次加载分类默认选中第一个分类项，如果某次加载失败也重置为第一个分类项
      if (this.cateId === 0 || this.errMsg) {
        this.selectCateID = this.cateList[0].id
        this.setCartId(this.selectCateID)
        this.getGoods()
      }
    }).catch(err => {
      this.errMsg = err.msg || '加载商品分类失败'
    })
  }

  // 获取分类商品数据，获取二级分类数据
  getGoods (id?: number) {
    const paramID = id || this.selectCateID

    // 增加loading过渡
    this.loadingStatus = 1
    goodsApi.getCategoryGoods(paramID).then(res => {
      this.errMsg = ''
      this.productInfoList = res.data
      this.noCateGoods = res.data.length <= 0

      // 处理价格为0时显示暂无报价，后期做在全局过滤器里
      this.productInfoList.forEach(ele => {
        ele.price = parseFloat(ele.price).toFixed(2)
      })
    }).catch(err => {
      this.errMsg = err.msg || '加载分类明细失败'
    }).finally(() => {
      this.loadingStatus = 0
    })
  }

  // 选择分类
  selectCates (index: number, id: number) {
    if (this.selectCateID === id) return false
    this.selectCateID = id
    this.setCartId(id)
    this.getGoods()

    // 设置滚动位置
    if (index > this.startScrollIndex) {
      const el = document.querySelector(`#menu li:nth-child(${index - this.startScrollIndex})`) as HTMLElement
      el.scrollIntoView({ behavior: 'smooth' })
    } else {
      const el = document.querySelector('#menu li:nth-child(1)') as HTMLElement
      el.scrollIntoView({ behavior: 'smooth' })
    }
  }
}
</script>

<style lang="scss" scoped>
.c-main {
  display: flex;
  height: 100%;
  padding-top: 88px;
}

.c-search {
  position: fixed;
  left: 0;
  z-index: 1;
  width: 100%;
  height: 44px;
  padding: 7px 10px;
  background-color: $e-f;
  border-bottom: 1px solid $e-line;
  .e-bgGray {
    height: 30px;
    border-radius: 35px;
    padding-left: 15px;
  }
}

.c-left {
  width: 80px;
  height: 100%;
  font-size: 12px;
  background-color: $e-gray;

  &_item {
    display: block;
    width: 80px;
    padding: 10px 0;
    text-align: center;

    .ctx {
      display: block;
      height: 24px;
      padding: 5px;
      border-left: 3px solid transparent;
    }
    &.on {
      color: $e-lightRed;
      background-color: $e-f;
    }
  }
}

.c-right {
  position: relative;
  flex: 1;
  height: 100%;
  background-color: $e-f;
  text-align: center;

  &_wrapper {
    padding: 15px 30px;

    &_product {
      width: 105px;
      margin-bottom: 15px;
      &.last {
        margin-bottom: 0;
      }
    }
  }
}

// loading页
.loading-page {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 130;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  background: $e-f;
  text-align: center;
}
</style>
