<template>
  <div class="explorer" ref="root">
    <!-- mode normal -->
    <div
      class="explorer-normal"
      v-if="mode === 'normal' && dataArr.length">
      <div
        class="explorer-normal-item"
        :class="{ active: selectedKeyArr.indexOf(item.key) !== -1 }"
        v-for="(item, i) in dataArr"
        :key="item.key"
        draggable
        @click="clickItem(item)"
        @contextmenu="openAction($event, item)"
        @dragstart="($event) => onDragstart($event, item)"
        @dragover.prevent
        @drop="($event) => onDrop($event, item)">
        <!-- 选中按钮 -->
        <img
          class="explorer-normal-item-selected"
          v-if="selection"
          src="./images/icon-selected.png"
          alt=""
          @click.stop="normalToggleSelected(item)">
        <!-- 图标 -->
        <img
          class="explorer-normal-item-icon"
          :src="typeIconMap[ item.type ] || typeIconMap['none']"
          alt="">
        <!-- 名称 -->
        <p class="explorer-normal-item-name">
          {{ item.name || '' }}
        </p>
      </div>
    </div>

    <!-- mode table -->
    <el-table
      class="explorer-table"
      ref="table"
      v-else-if="mode === 'table' && dataArr.length"
      :data="dataArr"
      @selection-change="tableToggleSelected"
      @row-click="(row, column, event) => clickItem(row)"
      @row-contextmenu="(row, column, event) => openAction(event, row)">
      <!-- 多选列 -->
      <el-table-column v-if="selection" type="selection"/>
      <!-- 名称列 -->
      <el-table-column label="名称">
        <template slot-scope="scope">
          <div class="explorer-table-name">
            <img :src="typeIconMap[ scope.row.type ]" alt="">
            {{ scope.row.name }}
          </div>
        </template>
      </el-table-column>
      <!-- 自定义的列 -->
      <el-table-column
        v-for="column in otherColumnArr"
        :key="column.label"
        :label="column.label">
        <template slot-scope="scope">
          {{ column.filter ? column.filter(scope.row.data[column.key]) : scope.row.data[column.key] }}
        </template>
      </el-table-column>
    </el-table>

    <!-- empty -->
    <img
      class="explorer-empty"
      src="./images/empty.png"
      v-else
      alt="">

    <!-- action -->
    <div
      class="explorer-action"
      v-show="showAction && actionArr.length"
      :style="{ left: actionX + 'px', top: actionY + 'px' }">
      <div
        class="explorer-action-item"
        v-for="action in actionArr"
        :key="action.label"
        @click="action.handler(selectedArr[0])">
        {{ action.label }}
      </div>
    </div>
  </div>
</template>

<script>
  export default {
    name: 'vue-explorer',
    props: {
      /**
       *  文件数据，一维数组
       *  每一项数据如下:
       *    {
       *      // 唯一值
       *      key: [String, Number],
       *
       *      // 显示名称
       *      name: String,
       *
       *      // 类型
       *      type: String[folder/docx/excel/image/mp3/pdf/ppt/txt/video/zip/none]
       *
       *      // 额外数据
       *      data: Object
       *    }
       **/
      dataArr: {
        type: Array,
        default () {
          return []
        }
      },

      /**
       *  是否开启选择
       **/
      selection: {
        type: Boolean,
        default: true
      },

      /**
       *  被选中的项数据集合
       *  注意：
       *    selection 要为 true
       *    table 模式有引用地址的问题
       *    所以要保证项对象在 dataArr 中能找到并且全等
       **/
      selectedArr: {
        type: Array,
        default () {
          return []
        }
      },

      /**
       *  显示模式
       *  normal - 正常模式
       *  table  - 表格模式
       **/
      mode: {
        type: String,
        default: 'normal'
      },

      /**
       *  table 模式显示新的列
       *  normal 该 prop 无效
       *  第一列选中，第二列名字
       *  是必有的，可以考虑通过 css 的方式隐藏这两个固有列
       *  来显示纯显示
       *  {
       *    label: String,
       *    key: String,
       *    filter: Function
       *  }
       **/
      otherColumnArr: {
        type: Array
      },

      /**
       *  对于每一项的单独操作
       *  右键项即可弹出操作列表
       *  [
       *    {
       *      label: String（显示的名字）
       *      handler: Function（处理函数，参数为当前项）
       *    }
       *  ]
       **/
      actionArr: {
        type: Array,
        default: () => {
          return []
        }
      }
    },
    data () {
      return {
        // 各个类型的图标数据
        typeIconMap: {
          doc: require('./images/icon-doc.png'),
          xls: require('./images/icon-xls.png'),
          folder: require('./images/icon-folder.png'),
          image: require('./images/icon-image.png'),
          mp3: require('./images/icon-mp3.png'),
          none: require('./images/icon-none.png'),
          pdf: require('./images/icon-pdf.png'),
          ppt: require('./images/icon-ppt.png'),
          video: require('./images/icon-video.png'),
          zip: require('./images/icon-zip.png'),
          rar: require('./images/icon-rar.png'),
          txt: require('./images/icon-txt.png')
        },

        // 显示 action
        showAction: false,

        // action 位置
        actionX: 0,
        actionY: 0
      }
    },
    computed: {
      selectedKeyArr () {
        let result = []
        this.selectedArr.forEach(item => {
          result.push(item.key)
        })
        return result
      }
    },
    methods: {
      // 点击文件或者文件夹
      clickItem (item) {
        this.$emit(
          item.type === 'folder' ?
            'clickFolder' :
            'clickFile',
          item
        )
      },

      // 打开操作菜单
      openAction (e, data) {
        if (this.actionArr.length === 0) {
          return null
        }

        let target = e.target
        let offsetX = e.offsetX
        let offsetY = e.offsetY

        while (target !== this.$refs.root) {
          offsetX += target.offsetLeft
          offsetY += target.offsetTop
          target = target.offsetParent
        }

        this.showAction = true
        this.actionX = offsetX
        this.actionY = offsetY

        e.preventDefault()

        // 选中当前的项
        this.$emit('update:selectedArr', [ data ])

        // 点击后隐藏 action
        document.addEventListener('click', (() => {
          let handler = () => {
            this.showAction = false
            document.removeEventListener('click', handler)
          }

          return handler
        })())
      },

      // 拖拽
      onDragstart (e, item) {
        let selectedArr = this.selectedArr
        let hasSelected = selectedArr.length
        let isSelected = (() => {
          for (let i = 0; i < selectedArr.length; i++) {
            if (selectedArr[ i ].key === item.key) {
              return true
            }
          }
          return false
        })()

        /**
         *  用户体验
         *    若当前没有选中项目
         *    或
         *    若当前已选中了项目，但是不包含当前拖拽的这一个
         *    则只选中当前这一个
         **/
        if (!hasSelected || (hasSelected && !isSelected)) {
          let newSelectedArr = selectedArr = [ item ]
          this.$emit('update:selectedArr', newSelectedArr)
        }

        e.dataTransfer.setData('text', JSON.stringify(
          selectedArr
        ))
      },
      onDrop (e, item) {
        let targetFolderData = item
        let selectedArr = JSON.parse(
          e.dataTransfer.getData('text')
        )
        let includeSelf = (() => {
          for (let i = 0; i < selectedArr.length; i++) {
            let selectedItem = selectedArr[ i ]
            if (selectedItem.key === targetFolderData.key) {
              return true
            }
          }
          return false
        })()

        /**
         *  ! 目标必须是个文件夹
         *  ! 不能自己移动自己
         **/
        if (
          targetFolderData.type !== 'folder' ||
          includeSelf
        ) {
          return null
        }

        this.$emit('dragMove', targetFolderData, selectedArr)
      },

      // normal 模式切换选中状态
      normalToggleSelected (item) {
        let arr = [ ...this.selectedArr ]
        let i = -1

        for (let j = 0; j < arr.length; j++) {
          if (arr[ j ].key === item.key) {
            i = j
            break
          }
        }

        if (i === -1) {
          arr.push(item)
        }
        else {
          arr.splice(i, 1)
        }

        this.$emit('update:selectedArr', arr)
      },

      // table 模式切换选中状态（el-table 自带全选功能）
      tableToggleSelected (arr) {
        this.$emit('update:selectedArr', [ ...arr ])
      },

      // 根据 prop selectedArr 同步 el-table 的选中状态
      syncElTableSelectedArr (newSelectedArr) {
        this.$refs.table.clearSelection()

        newSelectedArr.forEach(item => {
          this.$refs.table.toggleRowSelection(item)
        })
      }
    },
    watch: {
      selectedArr: {
        immediate: true,
        handler (newArr, oldArr) {
          if (
            this.mode !== 'table' ||
            this.dataArr.length === 0 ||
            JSON.stringify(newArr) === JSON.stringify(oldArr)
          ) {
            return null
          }
          else if (this.$refs.table) {
            this.syncElTableSelectedArr(newArr)
          }
          else {
            setTimeout(() => {
              this.syncElTableSelectedArr(newArr)
            }, 20)
          }
        }
      }
    }
  }
</script>

<style lang="scss" scoped>
  .explorer {
    position: relative;
    color: #000;
    box-sizing: border-box;
    font-size: 14px;
    background-color: #fff;
    line-height: 1.25;
    &-normal {
      display: flex;
      flex-wrap: wrap;
      &-item {
        position: relative;
        display: flex;
        align-items: center;
        justify-content: center;
        flex-direction: column;
        width: 100px;
        border-radius: 5px;
        cursor: pointer;
        padding: 10px;
        &-selected {
          position: absolute;
          top: 0;
          left: 0;
          z-index: 5;
          color: #4b4b4b;
          opacity: 0;
          cursor: pointer;
          width: 18px;
          margin: 5px;
        }
        &-icon {
          display: block;
        }
        &-name {
          color: #8e8c8c;
          margin: 10px 0 0 0;
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
          width: 100%;
          text-align: center;
        }

        // 鼠标移入状态
        &, &-selected {
          transition: all .15s;
        }
        &:hover {
          background-color: rgba(#000, 0.05);
        }
        &:hover &-selected {
          opacity: 0.4;
        }
        & &-selected:hover {
          opacity: 0.7;
        }

        // 选中时的状态
        &.active &-selected {
          color: #409EFF;
          opacity: 1;
        }
      }
    }
    &-table {
      font-size: inherit;
      &-name {
        display: flex;
        align-items: center;
        cursor: pointer;
        img {
          width: 30px;
          margin-right: 12px;
        }
      }

      /deep/ .el-table__row {
        cursor: pointer;
      }
    }
    &-empty {
      display: block;
      margin: 0 auto;
      max-width: 240px;
    }
    &-action {
      position: absolute;
      z-index: 5;
      border-radius: 5px;
      border: 1px solid #ebeef5;
      min-width: 100px;
      background-color: #fff;
      box-shadow: 0 2px 12px 0 rgba(0, 0, 0, .1);
      &-item {
        padding: 12px;
        & {
          transition: all .2s;
          cursor: pointer;
        }
        &:hover {
          background-color: rgba(#000, 0.05);
        }
      }
    }
  }
</style>