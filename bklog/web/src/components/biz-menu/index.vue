<!--
* Tencent is pleased to support the open source community by making
* 蓝鲸智云PaaS平台 (BlueKing PaaS) available.
*
* Copyright (C) 2021 THL A29 Limited, a Tencent company.  All rights reserved.
*
* 蓝鲸智云PaaS平台 (BlueKing PaaS) is licensed under the MIT License.
*
* License for 蓝鲸智云PaaS平台 (BlueKing PaaS):
*
* ---------------------------------------------------
* Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
* documentation files (the "Software"), to deal in the Software without restriction, including without limitation
* the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and
* to permit persons to whom the Software is furnished to do so, subject to the following conditions:
*
* The above copyright notice and this permission notice shall be included in all copies or substantial portions of
* the Software.
*
* THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO
* THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
* AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF
* CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
* IN THE SOFTWARE.
-->

<template>
  <div v-bk-clickoutside="handleClickOutSide" :class="['biz-menu-select', { 'light-theme': theme === 'light' }]">
    <!-- 图标+业务名称 -->
    <div class="menu-select">
      <!-- 图标 -->
      <span :style="`backgroundColor: ${spaceBgColor}`" class="menu-title">{{ bizNameIcon }}</span>
      <!-- 图标旁边的字 -->
      <span v-if="isExpand" class="menu-select-name" tabindex="{0}" @mousedown="handleClickBizSelect">
        {{ bizName }}
        <i :style="{ transform: `rotate(${!showBizList ? '0deg' : '-180deg'})` }"
          class="bk-select-angle bk-icon icon-angle-up-fill select-icon" />
      </span>
    </div>
    <!-- 下拉框内容 -->
    <div v-if="isExpand" :style="{ display: showBizList ? 'flex' : 'none' }" class="menu-select-list">
      <!-- 业务搜索框 -->
      <bk-input ref="menuSearchInput" class="menu-select-search" :clearable="false" :placeholder="$t('搜索')"
        :value="keyword" left-icon="bk-icon icon-search" @change="handleBizSearchDebounce"
        @clear="handleBizSearchDebounce">
      </bk-input>
      <!-- space空间选择栏 -->
      <ul v-if="showSpaceTypeIdList" id="space-type-ul" class="space-type-list">
        <li v-for="item in spaceTypeIdList" :style="{
          ...item.styles,
          borderColor: item.id === searchTypeId ? item.styles.color : 'transparent',
        }" class="space-type-item" :key="item.id" @click="handleSearchType(item.id)">
          {{ item.name }}
        </li>
      </ul>
      <!-- 业务列表 -->
      <div ref="bizListRef" :style="`width: ${bizBoxWidth}px`" class="biz-list" @scroll="handleScroll">
        <template v-if="groupList.length">
          <slot name="list-top"></slot>
          <template>
            <List
              :canSetDefaultSpace="canSetDefaultSpace"
              :checked="localValue"
              :list="generalList"
              :theme="theme"
              @handleClickOutSide="handleClickOutSide"
              @handleClickMenuItem="handleClickMenuItem"
            />
          </template>
        </template>
        <li v-else class="list-empty">
          {{ $t('无匹配的数据') }}
        </li>
      </div>
      <!-- 体验demo按钮 -->
      <div class="menu-select-extension">
        <!-- <div class="menu-select-extension-item">
          <span class="icon bk-icon icon-plus-circle"></span>
          {{ $t('申请业务权限') }}
        </div> -->
        <div v-if="!isExternal && demoUid" class="menu-select-extension-item" @mousedown.stop="experienceDemo">
          <span class="icon bklog-icon bklog-app-store"></span>
          {{ $t('体验DEMO') }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { Storage } from '@/common/util';
import navMenuMixin from '@/mixins/nav-menu-mixin';
import { SPACE_TYPE_MAP } from '@/store/constant';
import { debounce } from 'throttle-debounce';
import { mapState, mapGetters } from 'vuex';

import * as authorityMap from '../../common/authority-map';
import List from '../../global/biz-select/list';

const SPACE_COLOR_LIST = [
  '#7250A9',
  '#3563BE',
  '#3799BA',
  '#4FB17F',
  '#86AF4A',
  '#E9AE1D',
  '#EB9258',
  '#D36C68',
  '#BC4FB3',
];

  export default {
    components: {
      List,
    },
    mixins: [navMenuMixin],
    props: {
      isExpand: {
        type: Boolean,
        default: true,
      },
      theme: {
        type: String,
        default: 'dark',
      },
      handlePropsClick: {
        type: Function,
      },
      isExternalAuth: {
        type: Boolean,
        default: false,
      },
      canSetDefaultSpace: {
        type: Boolean,
        default: true,
      },
      // localValue: {
      //   type: Number,
      //   default: 0,
      // }
    },
    data() {
      return {
        bizId: '',
        keyword: '',
        showBizList: false,
        storage: null,
        BIZ_SELECTOR_COMMON_IDS: 'BIZ_SELECTOR_COMMON_IDS', // 常用的 的key
        BIZ_SELECTOR_COMMON_MAX: 5, // 常用的的最大长度
        spaceTypeIdList: [],
        commonListIds: [],
        groupList: [],
        searchTypeId: '',
        spaceBgColor: '#3799BA',
        bizBoxWidth: 418,
        exterlAuthSpaceName: '', // 用于授权外部版选择器显示
        generalList: [],
        localValue: 0,
        pagination: {
          current: 1,
          count: 0,
          limit: 20,
          data: [],
        },
      };
    },
    computed: {
      ...mapState(['isExternal']),
      ...mapGetters({
        demoUid: 'demoUid',
      }),
      bizName() {
        if (this.isExternalAuth && !!this.exterlAuthSpaceName) return this.exterlAuthSpaceName;
        return this.mySpaceList.find(item => item.space_uid === this.spaceUid)?.space_name;
      },
      bizNameIcon() {
        return this.bizName?.[0]?.toLocaleUpperCase() ?? '';
      },
      showSpaceTypeIdList() {
        // 外部版不展示空间分类
        return !this.isExternal && this.spaceTypeIdList.length > 1;
      },
    },
    watch: {
      async showBizList(val) {
        if (val) {
          await this.$nextTick();
          const el = document.querySelector('#space-type-ul');
          this.bizBoxWidth = Math.max(394, el?.clientWidth ?? 394) + 24;
        } else {
          this.$refs.bizListRef.scrollTop = 0;
        }
      },

      // 监听路由变化，解析spaceUid并赋值给localValue
      $route(to) {
        const spaceUid = to.query.spaceUid || to.params.spaceUid;
        if (spaceUid) {
          this.localValue = spaceUid;
        }
      },
    },
    created() {
      this.handleBizSearchDebounce = debounce(300, this.handleBizSearch);

      const spaceTypeMap = {};
      this.mySpaceList.forEach(item => {
        spaceTypeMap[item.space_type_id] = 1;
        if (item.space_type_id === 'bkci' && item.space_code) {
          spaceTypeMap.bcs = 1;
        }
      });
      this.spaceTypeIdList = Object.keys(spaceTypeMap).map(key => ({
        id: key,
        name: SPACE_TYPE_MAP[key]?.name || this.$t('未知'),
        styles: (this.theme === 'dark' ? SPACE_TYPE_MAP[key]?.dark : SPACE_TYPE_MAP[key]?.light) || {},
      }));
    },
    methods: {
      getRandomColor() {
        const color = SPACE_COLOR_LIST[Math.floor(Math.random() * SPACE_COLOR_LIST.length)];
        this.$store.commit('setSpaceBgColor', color);
        return color;
      },
      initGroupList() {
        this.storage = new Storage();
        this.commonListIds = this.storage.get(this.BIZ_SELECTOR_COMMON_IDS) || [];
        const generalList = [];
        this.mySpaceList.slice(0, 100).forEach(item => {
          let show = false;
          const keyword = this.keyword.trim().toLocaleLowerCase();
          if (this.searchTypeId) {
            show =
              this.searchTypeId === 'bcs'
                ? item.space_type_id === 'bkci' && !!item.space_code
                : item.space_type_id === this.searchTypeId;
          }
          if ((show && keyword) || (!this.searchTypeId && !show)) {
            show =
              item.space_name.toLocaleLowerCase().indexOf(keyword) > -1 ||
              item.py_text.toLocaleLowerCase().indexOf(keyword) > -1 ||
              item.space_uid.toLocaleLowerCase().indexOf(keyword) > -1 ||
              `${item.bk_biz_id}`.includes(keyword) ||
              `${item.space_code}`.includes(keyword);
          }
          if (show) {
            // 无权限 直接不显示
            if (!item.permission[authorityMap.VIEW_BUSINESS]) return;
            generalList.push(item);
          }
        });
        this.generalList = generalList;
        this.setPaginationData(true);
        // 只返回一个分组，全部数据都在generalList
        return [
          {
            id: 'general',
            name: this.$t('有权限的'),
            children: this.pagination.data,
          },
        ];
      },

      setPaginationData(isInit) {
        const showData = this.generalList;
        this.pagination.count = showData.length;
        if (isInit) {
          this.pagination.current = 1;
          this.pagination.data = showData.slice(0, this.pagination.limit);
        } else {
          if (this.pagination.current * this.pagination.limit < this.pagination.count) {
            this.pagination.current += 1;
            const temp = showData.slice(
              (this.pagination.current - 1) * this.pagination.limit,
              this.pagination.current * this.pagination.limit,
            );
            this.pagination.data.push(...temp);
          }
        }
      },
      handleScroll(event) {
        const el = event.target;
        const { scrollHeight, scrollTop, clientHeight } = el;
        if (Math.ceil(scrollTop) + clientHeight >= scrollHeight) {
          this.setPaginationData(false);
          const generalData = this.groupList.find(item => item.id === 'general');
          if (generalData?.children) {
            generalData.children = this.pagination.data;
          }
        }
      },
      handleClickBizSelect() {
        console.log('01---handleClickBizSelect', new Date().getTime());
        this.showBizList = !this.showBizList;
        if (this.showBizList) this.groupList = this.initGroupList();
        setTimeout(() => {
          this.$refs.menuSearchInput.focus();
          console.log('02---handleClickBizSelect', new Date().getTime());
        }, 100);
      },
      debounceUpdateRouter() {
        return debounce(60, space => {
          if (`${space.bk_biz_id}` !== this.$route.query.bizId || space.space_uid !== this.$route.query.spaceUid) {
            this.$router.push({
              params: {
                ...(this.$route.params ?? {}),
                indexId: undefined,
              },
              query: {
                ...(this.$route.query ?? {}),
                bizId: space.bk_biz_id,
                spaceUid: space.space_uid,
              },
            });
          }
        });
      },
      /**
       * @desc: 点击下拉框的空间选项
       * @param {Object} space 点击的空间
       * @param {String} type 点的是哪个分组的空间
       */
      handleClickMenuItem(space, type) {
        this.debounceUpdateRouter()(space);

        try {
          if (this.isExternalAuth) {
            this.exterlAuthSpaceName = space.space_name;
            this.localValue = space.space_uid;
            this.$emit('space-change', space.space_uid);
            return;
          }
          if (typeof this.handlePropsClick === 'function') return this.handlePropsClick(space); // 外部function调用
          if (type === 'general') this.commonAssignment(space.space_uid); // 点击有权限的业务时更新常用的ul列表
          this.checkSpaceChange(space.space_uid); // 检查是否有权限然后进行空间切换
          this.localValue = space.space_uid;
        } catch (error) {
          console.warn(error);
      } finally {
          this.showBizList = false;
      }
    },
    /**
     * @desc: 常用的分配
     * @param {Number} id 点击的space_uid
     * @param {Array} filterList 基于哪个数组进行过滤
     */
    commonAssignment(id = null) {
      const leng = this.commonListIds.length;
      if (!!id) {
        const isExist = this.commonListIds.includes(id);
        let newIds = [...this.commonListIds];
        if (isExist) newIds = newIds.filter(item => item !== id);
        newIds.unshift(id);
        this.commonListIds = newIds;
      }
      leng >= this.BIZ_SELECTOR_COMMON_MAX && (this.commonListIds.length = this.BIZ_SELECTOR_COMMON_MAX);
      this.storage.set(this.BIZ_SELECTOR_COMMON_IDS, this.commonListIds);
    },
    handleClickOutSide() {
      this.showBizList = false;
    },
    handleBizSearch(v) {
      this.keyword = v;
      this.groupList = this.initGroupList();
    },
    experienceDemo() {
      this.checkSpaceChange(this.demoUid);
    },
    handleSearchType(typeId) {
      this.searchTypeId = typeId === this.searchTypeId ? '' : typeId;
      this.groupList = this.initGroupList();
    },
  },
};
</script>

<style lang="scss">
@import '../../scss/mixins/flex.scss';
@import '../../scss/mixins/ellipsis.scss';

/* stylelint-disable no-descending-specificity */
.biz-menu-select {
  padding-left: 8px;

  .menu-select {
    position: relative;
    display: flex;
    flex: 1;
    align-items: center;
    height: 32px;
    padding: 0 4px 0 8px;
    border-radius: 2px;

    &-name {
      position: relative;
      flex: 1;
      padding: 0 26px 0 8px;
      overflow: hidden;
      font-size: 12px;
      line-height: 30px;
      color: #acb2c6;
      text-overflow: ellipsis;
      white-space: nowrap;
      cursor: pointer;

      .select-icon {
        position: absolute;
        top: 8px;
        right: 8px;
        color: #c4c6cc;
        transition:
          transform 0.3s cubic-bezier(0.4, 0, 0.2, 1),
          -webkit-transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
      }

      .icon-angle-up-fill {
        top: 8px;
        color: #96a2b9;
      }
    }

    &-list {
      position: fixed;
      top: 100px;
      left: 0;
      z-index: 3000;
      display: flex;
      flex-direction: column;
      overflow: auto;
      background-color: #38455f;
      border-radius: 2px;
      box-shadow: 0px 2px 6px 0px rgba(0, 0, 0, 0.2);

      .biz-list {
        display: flex;
        flex-direction: column;
        max-height: 240px;
        padding: 6px 0;
        overflow: auto;

        .group-title {
          display: inline-block;
          margin: 0 0 7px 12px;
          font-size: 12px;
          color: #66768e;
        }

        .list-empty,
        %list-empty {
          flex: 0 0 32px;
          height: 32px;
          padding: 0 9px 0 12px;
          font-size: 12px;
          color: #c3d0e7;

          @include flex-center;
        }

        .list-item {
          justify-content: space-between;

          @extend %list-empty;
          @include ellipsis;
          @include flex-align(left);

          &.is-select,
          &%is-select {
            color: #fff;
            background-color: #2c354d;
          }

          &:hover {
            cursor: pointer;
            background-color: #323c53;

            @extend %is-select;
          }

          &.is-disable {
            color: #66768e;
            cursor: not-allowed;
          }

          .text {
            width: 100%;
            overflow: hidden;
            text-overflow: ellipsis;
          }

          .apply-text {
            display: none;
            color: #3a84ff;
            cursor: pointer;
          }

          &:hover .apply-text {
            display: flex;
          }

          .list-item-left {
            /* stylelint-disable-next-line declaration-no-important */
            display: inline-flex !important;
            flex: 1;
            flex-wrap: nowrap;
            margin-right: 8px;

            @include ellipsis();

            .list-item-name {
              @include ellipsis();
            }

            .list-item-id {
              margin-left: 8px;

              @include ellipsis();
            }
          }
        }

        &::-webkit-scrollbar {
          width: 4px;
          background: #38455f;
        }

        &::-webkit-scrollbar-thumb {
          background: #ddd;
          border-radius: 20px;
          box-shadow: inset 0 0 6px rgba(204, 204, 204, 0.3);
        }
      }
    }

    &-search {
      flex: 1;
      width: inherit;
      padding: 0 12px;

      .left-icon {
        color: #63656e;
      }

      .bk-form-input {
        color: #acb5c6;
        background-color: #38455f;
        border: 0;
        border-bottom: 1px solid rgba(240, 241, 245, 0.16);
        border-radius: 0;

        &::placeholder {
          /* stylelint-disable-next-line declaration-no-important */
          color: #66768e !important;
          background-color: #39455f;
        }

        &:focus {
          /* stylelint-disable-next-line declaration-no-important */
          background-color: #39455f !important;

          /* stylelint-disable-next-line declaration-no-important */
          border-bottom-color: #434e68 !important;
        }
      }
    }

    &-extension {
      display: flex;
      padding: 10px 0;
      font-size: 12px;
      color: #c3d0e7;
      cursor: pointer;
      background-color: #323c53;
      border-top: 1px solid #434e68;

      &-item {
        flex-grow: 1;
        width: 50%;
        text-align: center;

        &:hover {
          color: #fff;
        }

        &:first-child {
          border-right: 1px solid #434e68;
        }

        &:last-child {
          border: 0;
        }

        .icon {
          font-size: 14px;
        }
      }
    }
  }

  .menu-title {
    flex: 1;
    width: 20px;
    min-width: 20px;
    max-width: 20px;
    height: 20px;
    font-size: 12px;
    font-weight: 700;
    color: #fff;
    background: #a09e21;
    border-radius: 2px;

    @include flex-center;
  }
}

.light-theme {
  padding: 0;

  .menu-select {
    background: transparent;
    border: 0;

    .menu-select-name {
      font-size: 14px;
      color: #313238;
    }

    .select-icon {
      /* stylelint-disable-next-line declaration-no-important */
      right: 2px !important;
    }

    &-list {
      top: 106px;
      left: 16px;
      min-width: 418px;
      background-color: #fff;
      outline: 1px solid #dcdee5;

      .biz-list {
        min-width: 418px;
        padding: 6px 0;

        .group-title {
          display: inline-block;
          margin: 0 0 7px 12px;
          font-size: 12px;
          color: #c4c6cc;
        }

        .list-empty,
        %list-empty {
          color: #63656e;
        }

        .list-item {
          max-width: 100%;

          @extend %list-empty;

          &.is-select,
          &%is-select {
            color: #3a84ff;
            background-color: #f5f7fa;
          }

          &:hover {
            @extend %is-select;
          }

          &.is-disable {
            color: #c4c6cc;
          }
        }

        &::-webkit-scrollbar {
          background: #fff;
        }

        &::-webkit-scrollbar-thumb {
          background: #dcdee5;
        }
      }
    }

    &-name {
      font-size: 12px;
      color: #63656e;
    }

    &-search {
      .bk-form-input {
        color: #63656e;
        background-color: #fff;
        border-bottom: 1px solid #eaebf0;

        &::placeholder {
          background-color: #fff;
        }

        &:focus {
          /* stylelint-disable-next-line declaration-no-important */
          background-color: #fff !important;

          /* stylelint-disable-next-line declaration-no-important */
          border-color: #eaebf0 !important;
        }
      }
    }

    &-extension {
      color: #63656e;
      background-color: #fafbfd;
      border-top: 1px solid #dcdee5;

      &-item {
        &:hover {
          color: #3a84ff;
        }

        &:first-child {
          border-color: #dcdee5;
        }
      }
    }
  }

  .select-icon {
    color: #c4c6cc;
  }

  .space-type-list {
    border-color: #eaebf0;
  }
}

.space-type-list {
  display: flex;
  align-items: center;
  padding: 8px 0;
  margin: 0 12px;
  border-bottom: 1px solid #434e68;

  .space-type-item {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 22px;
    padding: 0 10px;
    margin-right: 4px;
    font-size: 12px;
    cursor: pointer;
    border: 1px solid transparent;
    border-radius: 2px;
  }
}
</style>
