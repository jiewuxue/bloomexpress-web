<template>
  <s-layout
    navbar="normal"
    :leftWidth="0"
    :rightWidth="0"
    tools="search"
    :defaultSearch="state.keyword"
    @search="onSearch"
  >
    <!-- 筛选 -->
    <su-sticky bgColor="#fff">
      <view class="ss-flex">
        <view class="ss-flex-1">
          <su-tabs
            :list="state.tabList"
            :scrollable="false"
            @change="onTabsChange"
            :current="state.currentTab"
          />
        </view>
        <view class="list-icon" @tap="state.iconStatus = !state.iconStatus">
          <text v-if="state.iconStatus" class="sicon-goods-list" />
          <text v-else class="sicon-goods-card" />
        </view>
      </view>
    </su-sticky>

    <!-- 弹窗 -->
    <su-popup
      :show="state.showFilter"
      type="top"
      round="10"
      :space="sys_navBar + 38"
      backgroundColor="#F6F6F6"
      :zIndex="10"
      @close="state.showFilter = false"
    >
      <view class="filter-list-box">
        <view
          class="filter-item"
          v-for="(item, index) in state.tabList[state.currentTab].list"
          :key="item.value"
          :class="[{ 'filter-item-active': index === state.curFilter }]"
          @tap="onFilterItem(index)"
        >
          {{ item.label }}
        </view>
      </view>
    </su-popup>

    <!-- 情况一：单列布局 -->
    <view v-if="state.iconStatus && state.pagination.total > 0" class="goods-list ss-m-t-20">
      <view
        class="ss-p-l-20 ss-p-r-20 ss-m-b-20"
        v-for="item in state.pagination.list"
        :key="item.id"
      >
        <s-goods-column
          class=""
          size="lg"
          :data="item"
          :topRadius="10"
          :bottomRadius="10"
          @click="sheep.$router.go('/pages/goods/index', { id: item.id })"
        >
          <template v-slot:cart>
            <button
              class="ss-reset-button list-cart-btn list-cart-btn-lg"
              @tap.stop="onQuickAddCart(item)"
            >
              <image
                class="list-cart-icon"
                src="/static/img/shop/tools/cart.png"
                mode="aspectFit"
              />
            </button>
          </template>
        </s-goods-column>
      </view>
    </view>
    <!-- 情况二：双列布局 -->
    <view
      v-if="!state.iconStatus && state.pagination.total > 0"
      class="ss-flex ss-flex-wrap ss-p-x-20 ss-m-t-20 ss-col-top"
    >
      <view class="goods-list-box">
        <view class="left-list" v-for="item in state.leftGoodsList" :key="item.id">
          <s-goods-column
            class="goods-md-box"
            size="md"
            :data="item"
            :topRadius="10"
            :bottomRadius="10"
            @click="sheep.$router.go('/pages/goods/index', { id: item.id })"
            @getHeight="mountMasonry($event, 'left')"
          >
            <template v-slot:cart>
              <button class="ss-reset-button list-cart-btn" @tap.stop="onQuickAddCart(item)">
                <image
                  class="list-cart-icon"
                  src="/static/img/shop/tools/cart.png"
                  mode="aspectFit"
                />
              </button>
            </template>
          </s-goods-column>
        </view>
      </view>
      <view class="goods-list-box">
        <view class="right-list" v-for="item in state.rightGoodsList" :key="item.id">
          <s-goods-column
            class="goods-md-box"
            size="md"
            :topRadius="10"
            :bottomRadius="10"
            :data="item"
            @click="sheep.$router.go('/pages/goods/index', { id: item.id })"
            @getHeight="mountMasonry($event, 'right')"
          >
            <template v-slot:cart>
              <button class="ss-reset-button list-cart-btn" @tap.stop="onQuickAddCart(item)">
                <image
                  class="list-cart-icon"
                  src="/static/img/shop/tools/cart.png"
                  mode="aspectFit"
                />
              </button>
            </template>
          </s-goods-column>
        </view>
      </view>
    </view>
    <uni-load-more
      v-if="state.pagination.total > 0"
      :status="state.loadStatus"
      :content-text="{
        contentdown: '上拉加载更多',
      }"
      @tap="loadMore"
    />
    <s-empty v-if="state.pagination.total === 0" icon="/static/soldout-empty.png" text="暂无商品" />

    <!-- 多规格商品：选择规格弹窗 -->
    <s-select-sku
      v-if="state.showSelectSku && state.skuGoods"
      :key="state.skuGoods.id"
      :goodsInfo="state.skuGoods"
      :show="true"
      @addCart="onSelectSkuAddCart"
      @close="state.showSelectSku = false"
    />
  </s-layout>
</template>

<script setup>
  import { reactive, ref } from 'vue';
  import { onLoad, onReachBottom } from '@dcloudio/uni-app';
  import sheep from '@/sheep';
  import { concat } from 'lodash-es';
  import { resetPagination } from '@/sheep/helper/utils';
  import SpuApi from '@/sheep/api/product/spu';
  import OrderApi from '@/sheep/api/trade/order';
  import { appendSettlementProduct } from '@/sheep/hooks/useGoods';
  import { showAuthModal } from '@/sheep/hooks/useModal';

  const sys_navBar = sheep.$platform.navbar;
  const emits = defineEmits(['close', 'change']);

  const state = reactive({
    pagination: {
      list: [],
      total: 0,
      pageNo: 1,
      pageSize: 6,
    },
    currentSort: undefined,
    currentOrder: undefined,
    currentTab: 0, // 当前选中的 tab
    curFilter: 0, // 当前选中的 list 筛选项
    showFilter: false,
    iconStatus: false, // true - 单列布局；false - 双列布局
    keyword: '',
    categoryId: 0,
    tabList: [
      {
        name: '综合推荐',
        list: [
          {
            label: '综合推荐',
          },
          {
            label: '价格升序',
            sort: 'price',
            order: true,
          },
          {
            label: '价格降序',
            sort: 'price',
            order: false,
          },
        ],
      },
      {
        name: '销量',
        sort: 'salesCount',
        order: false,
      },
      {
        name: '新品优先',
        value: 'createTime',
        order: false,
      },
    ],
    loadStatus: '',
    leftGoodsList: [], // 双列布局 - 左侧商品
    rightGoodsList: [], // 双列布局 - 右侧商品
    skuGoods: null, // 多规格商品 - 选择规格弹窗使用的 SPU 详情
    showSelectSku: false, // 多规格商品 - 是否展示规格弹窗
  });

  // 加载瀑布流
  let count = 0;
  let leftHeight = 0;
  let rightHeight = 0;

  // 处理双列布局 leftGoodsList + rightGoodsList
  function mountMasonry(height = 0, where = 'left') {
    if (where === 'left') {
      leftHeight += height;
    } else {
      rightHeight += height;
    }
    if (!state.pagination.list[count]) {
      return;
    }

    if (leftHeight <= rightHeight) {
      state.leftGoodsList.push(state.pagination.list[count]);
    } else {
      state.rightGoodsList.push(state.pagination.list[count]);
    }
    count++;
  }

  // 清空列表
  function emptyList() {
    resetPagination(state.pagination);
    state.leftGoodsList = [];
    state.rightGoodsList = [];
    count = 0;
    leftHeight = 0;
    rightHeight = 0;
  }

  // 搜索
  function onSearch(e) {
    state.keyword = e;
    emptyList();
    getList(state.currentSort, state.currentOrder);
  }

  // 点击
  function onTabsChange(e) {
    // 如果点击的是【综合推荐】，则直接展开或者收起筛选项
    if (state.tabList[e.index].list) {
      state.currentTab = e.index;
      state.showFilter = !state.showFilter;
      return;
    }
    state.showFilter = false;

    // 如果点击的是【销量】或者【新品优先】，则直接切换 tab
    if (e.index === state.currentTab) {
      return;
    }

    state.currentTab = e.index;
    state.currentSort = e.sort;
    state.currentOrder = e.order;
    emptyList();
    getList(e.sort, e.order);
  }

  // 点击 tab 的 list 筛选项
  const onFilterItem = (val) => {
    // 如果点击的是当前的筛选项，则直接收起筛选项，不要加载数据
    // 这里选择 tabList[0] 的原因，是目前只有它有 list
    if (
      state.currentSort === state.tabList[0].list[val].sort &&
      state.currentOrder === state.tabList[0].list[val].order
    ) {
      state.showFilter = false;
      return;
    }
    state.showFilter = false;

    // 设置筛选条件
    state.curFilter = val;
    state.tabList[0].name = state.tabList[0].list[val].label;
    state.currentSort = state.tabList[0].list[val].sort;
    state.currentOrder = state.tabList[0].list[val].order;
    // 清空 + 加载数据
    emptyList();
    getList();
  };

  async function getList() {
    state.loadStatus = 'loading';
    const { code, data } = await SpuApi.getSpuPage({
      pageNo: state.pagination.pageNo,
      pageSize: state.pagination.pageSize,
      sortField: state.currentSort,
      sortAsc: state.currentOrder,
      categoryId: state.categoryId,
      keyword: state.keyword,
    });
    if (code !== 0) {
      return;
    }
    // 拼接结算信息（营销）
    await OrderApi.getSettlementProduct(data.list.map((item) => item.id).join(',')).then((res) => {
      if (res.code !== 0) {
        return;
      }
      appendSettlementProduct(data.list, res.data);
    });
    state.pagination.list = concat(state.pagination.list, data.list);
    state.pagination.total = data.total;
    state.loadStatus = state.pagination.list.length < state.pagination.total ? 'more' : 'noMore';
    mountMasonry();
  }

  // 加载更多
  function loadMore() {
    if (state.loadStatus === 'noMore') {
      return;
    }
    state.pagination.pageNo++;
    getList(state.currentSort, state.currentOrder);
  }

  // 列表卡片：快速加入购物车
  async function onQuickAddCart(item) {
    if (item.stock != null && item.stock <= 0) {
      sheep.$helper.toast('该商品无库存');
      return;
    }
    // 未登录：弹出登录框
    if (!sheep.$store('user').isLogin) {
      showAuthModal();
      return;
    }
    // 加载 SPU 详情（含 SKU）
    const { code, data } = await SpuApi.getSpuDetail(item.id);
    if (code !== 0 || !data) {
      return;
    }
    const skus = data.skus || [];
    if (skus.length === 1) {
      // 单规格：直接加入
      if (skus[0].stock <= 0) {
        sheep.$helper.toast('该商品无库存');
        return;
      }
      sheep.$store('cart').add({ id: skus[0].id, goods_num: 1 });
    } else {
      // 多规格：弹出选择规格弹窗
      state.skuGoods = data;
      state.showSelectSku = true;
    }
  }

  // 规格弹窗：加入购物车
  function onSelectSkuAddCart(e) {
    if (!e.id) {
      sheep.$helper.toast('请选择商品规格');
      return;
    }
    sheep.$store('cart').add(e);
    state.showSelectSku = false;
  }

  onLoad((options) => {
    state.categoryId = options.categoryId;
    state.keyword = options.keyword;
    getList(state.currentSort, state.currentOrder);
  });

  // 上拉加载更多
  onReachBottom(() => {
    loadMore();
  });
</script>

<style lang="scss" scoped>
  // 列表卡片 - 快速加购物车按钮
  .list-cart-btn {
    width: 54rpx;
    height: 54rpx;
    background: linear-gradient(90deg, #fe8900, #ff5e00);
    border-radius: 50%;
    position: absolute;
    bottom: 50rpx;
    right: 20rpx;
    z-index: 2;
  }

  .list-cart-btn-lg {
    bottom: 20rpx;
  }

  .list-cart-icon {
    width: 30rpx;
    height: 30rpx;
  }

  .goods-list-box {
    width: 50%;
    box-sizing: border-box;

    .left-list {
      margin-right: 10rpx;
      margin-bottom: 20rpx;
    }

    .right-list {
      margin-left: 10rpx;
      margin-bottom: 20rpx;
    }
  }

  .goods-box {
    &:nth-last-of-type(1) {
      margin-bottom: 0 !important;
    }

    &:nth-child(2n) {
      margin-right: 0;
    }
  }

  .list-icon {
    width: 80rpx;

    .sicon-goods-card {
      font-size: 40rpx;
    }

    .sicon-goods-list {
      font-size: 40rpx;
    }
  }

  .goods-card {
    margin-left: 20rpx;
  }

  .list-filter-tabs {
    background-color: #fff;
  }

  .filter-list-box {
    padding: 28rpx 52rpx;

    .filter-item {
      font-size: 28rpx;
      font-weight: 500;
      color: #333333;
      line-height: normal;
      margin-bottom: 24rpx;

      &:nth-last-child(1) {
        margin-bottom: 0;
      }
    }

    .filter-item-active {
      color: var(--ui-BG-Main);
    }
  }

  .tab-item {
    height: 50px;
    position: relative;
    z-index: 11;

    .tab-title {
      font-size: 30rpx;
    }

    .cur-tab-title {
      font-weight: $font-weight-bold;
    }

    .tab-line {
      width: 60rpx;
      height: 6rpx;
      border-radius: 6rpx;
      position: absolute;
      left: 50%;
      transform: translateX(-50%);
      bottom: 10rpx;
      background-color: var(--ui-BG-Main);
      z-index: 12;
    }
  }
</style>
