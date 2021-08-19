<template>
  <div class="flex-direction-column">
    <!-- 表格 -->
    <div class="table-container" :style="tableStyle">
      <el-table
        ref="tableEle"
        :data="tableData"
        v-bind="tableConfig"
        class="table-custom"
        tooltip-effect="dark"
        header-cell-class-name="table-header-cell"
        @current-change="handleCurrentChange"
        @select="handleSelect"
        @select-all="handleSelectAll"
        @selection-change="handleSelectionChange"
        @sort-change="handleSortChange"
        @cell-mouse-enter="handleCellMouseEnter"
        @cell-mouse-leave="handleCellMouseLeave"
        @cell-click="handleCellClick"
        @cell-dblclick="handleCellDblclick"
        @row-click="handleRowClick"
        @row-contextmenu="handleRowContextmenu"
        @row-dblclick="handleRowDblclick"
        @header-click="handleHeaderClick"
        @header-contextmenu="handleHeaderContextmenu"
      >
        <template v-for="(item, index) in columnConfig">
          <!-- selection -->
          <el-table-column
            v-if="item.config.type === 'selection'"
            :key="JSON.stringify(item.config.prop)"
            v-bind="item.config"
          >
          </el-table-column>

          <!-- index -->
          <el-table-column
            v-else-if="item.config.type === 'index'"
            :key="JSON.stringify(item.config.prop)"
            v-bind="item.config"
          >
          </el-table-column>

          <!-- normal -->
          <el-table-column v-else :key="item.config.prop" v-bind="item.config">
            <template slot-scope="scope">
              <!-- 插槽 -->
              <slot
                v-if="item.isSlot"
                :name="item.config.prop"
                :td="scope.row[item.config.prop]"
                :row="scope.row"
                :index="index"
              >
              </slot>

              <!-- tag -->
              <template v-else-if="item.formate && item.formate.type === 'tag'">
                <!-- 多个tag需要遍历 -->
                <template v-if="Array.isArray(scope.row[item.config.prop])">
                  <template v-if="scope.row[item.config.prop].length !== 0">
                    <el-tooltip
                      :disabled="tagOutRange[scope.$index]"
                      placement="top-start"
                      :content="
                        getTagData(item, scope.row[item.config.prop]).join('，')
                      "
                    >
                      <span
                        class="span-tags"
                        @mouseover="onTagHover(scope.$index)"
                      >
                        <span>
                          <el-tag
                            v-for="(tag, i) in getTagData(
                              item,
                              scope.row[item.config.prop]
                            )"
                            :key="i"
                            disable-transitions
                            :type="
                              item.status &&
                              item.status.method(tag, ...item.status.params) !==
                                'primary'
                                ? item.status.method(tag, ...item.status.params)
                                : undefined
                            "
                            size="small"
                            >{{ tag }}</el-tag
                          >
                        </span>
                      </span>
                    </el-tooltip>
                  </template>
                  <span v-else>{{ getDefaultValue(item.formate) }}</span>
                </template>

                <!-- 单个tag -->
                <template v-else>
                  <el-tag
                    v-if="scope.row[item.config.prop]"
                    disable-transitions
                    :type="
                      item.status &&
                      item.status.method(tag, ...item.status.params) !==
                        'primary'
                        ? item.status.method(tag, ...item.status.params)
                        : undefined
                    "
                    size="small"
                    >{{ getTagData(item, scope.row[item.config.prop]) }}</el-tag
                  >
                  <span v-else>{{
                    getDefaultValue(item.formate, scope.row[item.config.prop])
                  }}</span>
                </template>
              </template>

              <!-- 时间格式化 -->
              <span
                v-else-if="item.formate && item.formate.type === 'time'"
                :class="getStatus(item, scope.row[item.config.prop])"
                >{{
                  scope.row[item.config.prop]
                    | parseTime(item.formate.params && item.formate.params[0])
                }}</span
              >

              <!-- 内存格式化 -->
              <span
                v-else-if="item.formate && item.formate.type === 'memory'"
                :class="getStatus(item, scope.row[item.config.prop])"
                >{{ scope.row[item.config.prop] | sizeFilter }}</span
              >

              <!-- 单位格式化 -->
              <span
                v-else-if="item.formate && item.formate.type === 'unit'"
                :class="getStatus(item, scope.row[item.config.prop])"
                >{{
                  scope.row[item.config.prop]
                    | unitFilter(item.formate.params && item.formate.params[0])
                }}</span
              >
              <!-- 文本自定义格式化 -->
              <span
                v-else-if="item.formate && item.formate.type === 'customize'"
                :class="getStatus(item, scope.row[item.config.prop])"
                >{{ getCustomizeRst(item, scope.row[item.config.prop]) }}</span
              >

              <!-- 默认 -->
              <span
                v-else
                :class="getStatus(item, scope.row[item.config.prop])"
                >{{
                  scope.row[item.config.prop]
                    ? scope.row[item.config.prop]
                    : getDefaultValue(item.formate, scope.row[item.config.prop])
                }}</span
              >
            </template>
          </el-table-column>
        </template>
      </el-table>
    </div>

    <!-- 翻页 -->
    <div v-if="pageConfig.isShow" class="pagination-container">
      <el-pagination
        :current-page.sync="currentPage"
        :page-size.sync="pageSize"
        :total="totalCounts"
        :background="true"
        :page-sizes="pageSizes"
        :layout="layout"
        v-bind="$attrs"
        @size-change="handlePaginationChange"
        @current-change="handlePaginationChange"
      />
    </div>
  </div>
</template>

<script>
import { scrollTo } from "./scrollTo";
import eventMethods from "./eventMethods.js";
export default {
  name: "TableContainer",
  filters: {
    //存储单位格式化
    sizeFilter(bytes) {
      if (!bytes) return "0 B";
      const k = 1000, // or 1024
        sizes = ["B", "KB", "MB", "GB", "TB", "PB", "EB", "ZB", "YB"],
        i = Math.floor(Math.log(bytes) / Math.log(k));
      return (bytes / Math.pow(k, i)).toPrecision(3) + " " + sizes[i];
    },
    //时间格式化
    parseTime(time, cFormat) {
      if (!time) {
        return "--";
      }
      const formate = cFormat || "{y}-{m}-{d} {h}:{i}:{s}";
      let date;
      if (typeof time === "object") {
        date = time;
      } else {
        if (typeof time === "string" && /^[0-9]+$/.test(time)) {
          time = parseInt(time);
        }
        if (typeof time === "number" && time.toString().length === 10) {
          time = time * 1000;
        }
        date = new Date(time);
      }
      const formatObj = {
        y: date.getFullYear(),
        m: date.getMonth() + 1,
        d: date.getDate(),
        h: date.getHours(),
        i: date.getMinutes(),
        s: date.getSeconds(),
        a: date.getDay()
      };
      const time_str = formate.replace(/{(y|m|d|h|i|s|a)+}/g, (result, key) => {
        let value = formatObj[key];
        // Note: getDay() returns 0 on Sunday
        if (key === "a") {
          return ["日", "一", "二", "三", "四", "五", "六"][value];
        }
        if (result.length > 0 && value < 10) {
          value = "0" + value;
        }
        return value || 0;
      });
      return time_str;
    },
    // 单位格式化
    unitFilter(val, unit) {
      if (!val && val !== 0) return "--";
      if (unit) return `${val} ${unit}`;
      return val;
    }
  },
  mixins: [eventMethods],
  props: {
    // 表格数据
    tableInfo: {
      type: Object,
      default: () => ({
        data: [],
        currentPage: 1,
        totalPages: 0,
        totalCounts: 0
      })
    },
    // table Ele配置内容
    tableConfig: {
      type: Object,
      default: () => ({})
    },
    // 表格配置信息
    columnConfig: {
      type: Array,
      default: () => []
    },
    pageConfig: {
      default: () => ({}),
      validator: function(value) {
        // 校验pageConfig的数据类型
        if (Object.prototype.toString.call(value) === "[object Object]") {
          const { limit, pageSizes, layout, autoScroll, isShow } = value;
          // 校验pageConfig配置项数据格式
          if (
            (limit !== undefined && typeof limit !== "number") ||
            (layout !== undefined && typeof layout !== "string") ||
            (autoScroll !== undefined && typeof autoScroll !== "boolean") ||
            (isShow !== undefined && typeof isShow !== "boolean") ||
            (pageSizes !== undefined && !Array.isArray(pageSizes))
          ) {
            return false;
          } else {
            return true;
          }
        } else {
          return false;
        }
      }
    },
    //表格包裹层样式
    tableStyle: {
      type: Object,
      default: () => ({})
    }
  },
  data() {
    return {
      tableData: [],
      pageSize: 10,
      currentPage: 1,
      totalCounts: 0,
      totalPages: 0,
      // 翻页配置信息
      limit: 10,
      pageSizes: [10, 20, 30, 40],
      layout: "total, sizes, prev, pager, next, jumper",
      autoScroll: true,
      tagOutRange: [] // tag是否超出范围
    };
  },
  computed: {
    tableEle() {
      return this.$refs.tableEle; // 导出 el-table Dom元素
    }
  },
  watch: {
    tableInfo: {
      handler(val) {
        const { currentPage, totalCounts } = val;
        // 过滤时
        if (currentPage === 1) this.currentPage = 1;

        this.totalCounts = totalCounts;
        // 如果没有pageSize, 手动补全
        if (!val.pageSize) {
          let tableInfo = this.deepClone(this.tableInfo);
          tableInfo.pageSize = this.pageSize;
          // 更新父组件 tableInfo
          this.$emit("update:tableInfo", this.jsonClone(tableInfo));
        }

        // 页码不为1且数据为空, 则不去更新表格数据
        if (!(val.currentPage !== 1 && !val.data[0]))
          this.tableData = this.deepClone(val.data);
      },
      deep: true
    },
    columnConfig: {
      handler(val) {
        val.map(item => {
          // 初始化 formate.params
          if (
            item.formate &&
            (item.formate.type === "customize" ||
              item.formate.type === "tag") &&
            !Array.isArray(item.formate.params)
          ) {
            item.formate.params = [];
          }
          // 取消 tag类型的show-overflow-tooltip
          if (item.formate && item.formate.type === "tag") {
            item.config["show-overflow-tooltip"] = false;
          }
          if (item.status && !item.status.params) {
            // 初始化 status.params
            item.status.params = [];
          }
        });
      },
      immediate: true,
      deep: true
    },
    currentPage(val) {
      // 无数据跳转前一页
      if (
        this.tableInfo.currentPage > 1 &&
        (!this.tableInfo.data || !this.tableInfo.data[0])
      ) {
        let tableInfo = this.deepClone(this.tableInfo);
        tableInfo.currentPage = val;
        tableInfo.pageSize = this.pageSize;
        // 更新父组件 tableInfo
        this.$emit("update:tableInfo", this.jsonClone(tableInfo));
        this.$emit("pagination", {
          currentPage: val,
          pageSize: this.pageSize
        });
      }
    },
    pageConfig: {
      handler(val) {
        let { limit, pageSizes, layout, autoScroll, isShow } = val;
        // limit有值且pageSizes未配置
        if (limit && !pageSizes) {
          this.pageSizes = [];

          this.limit = limit;
          [1, 2, 3, 4].map(num => {
            this.pageSizes.push(num * limit);
          });
        }
        if (Array.isArray(pageSizes) && pageSizes[0])
          this.pageSizes = pageSizes;
        if (layout) this.layout = layout;
        if (autoScroll !== undefined) this.autoScroll = autoScroll;
        if (isShow === undefined) val.isShow = true; // 默认显示
      },
      immediate: true,
      deep: true
    }
  },
  created() {
    let { data, currentPage, totalCounts } = this.deepClone(this.tableInfo);
    const limit = this.pageConfig.limit;
    if (limit) this.pageSize = limit;
    // 初始化 tableInfo
    let tableInfo = {
      data,
      pageSize: limit ? limit : 10,
      currentPage: currentPage ? currentPage : 1,
      totalCounts: totalCounts ? totalCounts : 0,
      totalPages: 1
    };
    // 更新父组件 tableInfo
    this.$emit("update:tableInfo", this.jsonClone(tableInfo));
  },
  methods: {
    handlePaginationChange() {
      let tableInfo = this.deepClone(this.tableInfo);
      tableInfo.currentPage = this.currentPage;
      tableInfo.pageSize = this.pageSize;
      // 更新父组件 tableInfo
      this.$emit("update:tableInfo", this.jsonClone(tableInfo));

      this.$emit("pagination", {
        currentPage: this.currentPage,
        pageSize: this.pageSize
      });
      if (this.autoScroll) {
        scrollTo(0, 800);
      }
    },
    onTagHover(index) {
      let parent = event.currentTarget;
      let children = event.currentTarget.children[0];
      if (parent.offsetWidth > children.offsetWidth) {
        this.$set(this.tagOutRange, index, true);
      } else {
        this.$set(this.tagOutRange, index, false);
      }
    },

    // 获取值为空的默认值
    getDefaultValue(formate, value) {
      let str = "--";
      if (value === 0) str = value;
      if (
        formate &&
        formate.defaultValue !== undefined &&
        formate.defaultValue !== null &&
        formate.defaultValue !== ""
      ) {
        str = formate.defaultValue;
      }
      return str;
    },
    // 获取自定义格式化结果
    getCustomizeRst(item, data) {
      let rst = item.formate.method(data, ...item.formate.params);

      // 格式化结果为空值, 重新赋默认空值
      if (!rst && rst !== 0) rst = this.getDefaultValue(item.formate, data);
      return rst;
    },
    // 获取状态
    getStatus(item, data) {
      return item.status && item.status.method(data, ...item.status.params);
    },
    // 获取tag格式化内容
    getTagData(item, data) {
      const params = item.formate.params;
      if (item.formate && item.formate.method) {
        return item.formate.method(data, ...params);
      }
      return data;
    },
    // 深拷贝
    deepClone(obj, cache = []) {
      if (obj === null || typeof obj !== "object") {
        return obj;
      }
      const hit = cache.filter(c => c.original === obj)[0];
      if (hit) {
        return hit.copy;
      }
      const copy = Array.isArray(obj) ? [] : {};
      cache.push({
        original: obj,
        copy
      });
      Object.keys(obj).forEach(key => {
        copy[key] = this.deepClone(obj[key], cache);
      });
      return copy;
    },
    jsonClone(data) {
      return JSON.parse(JSON.stringify(data));
    }
  }
};
</script>

<style scoped lang="scss">
// insight状态色
.info {
  color: var(--color-info, #909399);
}
.primary {
  color: var(--color-primary, #188cff);
}
.success {
  color: var(--color-success, #67c23a);
}
.warning {
  color: var(--color-warning, #faad14);
}
.danger {
  color: var(--color-danger, #f13a30);
}
.pagination-container {
  padding: 32px 16px;
}
.span-tags {
  white-space: nowrap;
  width: 100%;
  overflow: hidden;
  display: block;
  text-overflow: ellipsis;
}
</style>
