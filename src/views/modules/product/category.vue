<template>
  <div>
    <!--
      node-key
      每个树节点用来作为唯一标识的属性，整棵树应该是唯一的
      default-expanded-keys 默认展开的节点的 key 的数组
      draggable结点可拖拽
      allow-drop判定可拖拽的业务
    -->
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button @click="batchSave" v-if="draggable" size="mini">批量保存</el-button>
    <el-button type="danger" @click="batchDelete" size="mini">批量删除</el-button>
    <el-tree
      @node-drop="handleDrop"
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      ref="menuTree"
    >
      <!--自定义结点内容-->
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <!--
            只有当前结点没有子菜单的时候才显示删除按钮，
            只有当前结点是一级、二级菜单的时候才显示添加按钮
            返回数据中的level可以表示当前是几级菜单
            返回数据的childrenNode可以表示是否有子节点
          -->
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            style="color: #409EFF"
            @click="() => append(data)"
          >
            添加
          </el-button>
          <el-button
            type="text"
            style="color: #67C23A"
            size="mini" @click="() => edit(data)">
            修改
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            style="color: #F56C6C"
            @click="() => remove(node, data)">
            删除
          </el-button>
        </span>
      </span>
    </el-tree>

    <!--
      对话框组件
      点击添加结点的时候，对话框会弹出
      :visible.sync="dialogVisible"控制对话框是否弹出
      在添加子结点的时候要展示当前结点的子节点信息
    -->
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false">

      <!--对话框中嵌套表单-->
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: 'category',
  components: {},
  directives: {},
  data () {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: '',
      dialogType: '',
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      /**
       * category:添加子节点时需要的信息
       *
       * name：子节点名称
       * parentCid：父节点id
       * catLevel：结点菜单等级、层级
       * showStatus：是否展示
       * sort：当前排序字段
       */
      category: {
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: '',
        productUnit: '',
        catId: null
      },
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  mounted () {
  },
  methods: {
    // 批量删除
    batchDelete () {
      let catIds = []
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('被选中的元素', checkedNodes)
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
      }

      this.$confirm(`是否批量删除【${catIds}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          })
            .then(({data}) => {
              this.$message({
                type: 'success',
                message: '菜单批量删除成功!'
              })
              // 刷新出新的菜单
              this.getMenus()
            })
            .catch(() => {
            })
        })
        .catch(() => {
        })
    },
    // 批量修改
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      })
        .then(({data}) => {
          this.$message({
            type: 'success',
            message: '菜单顺序修改成功!'
          })
          // 刷新出新的菜单
          this.getMenus()
          // 设置需要默认展开的菜单
          this.expandedKey = this.pCid
          this.updateNodes = []
          this.maxLevel = 0
          // this.pCid = 0;
        })
        .catch(() => {
        })
    },

    /**
     *
     * @param draggingNode 当前拖拽的结点
     * @param dropNode 进入到哪个结点层级
     * @param dropType 进入到结点什么位置，前，后，里面
     * @param ev
     */
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType)

      // 1 当前节点最新的父节点
      let pCid = 0
      let siblings = null
      if (dropType === 'before' || dropType === 'after') {
        pCid =
          dropNode.parent.data.catId === undefined
            ? 0
            : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid.push(pCid)
      // 2 当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level
          if (siblings[i].level !== draggingNode.level) {
            // 当前节点的层级发生变化
            catLevel = siblings[i].level
            // 修改他子节点的层级
            this.updateChildNodeLevlel(siblings[i])
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel
          })
        } else {
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i})
        }
      }
      // 3 当前拖拽节点的最新层级
      console.log('updateNodes', this.updateNodes)
    },
    updateChildNodeLevlel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          })
          this.updateChildNodeLevlel(node.childNodes[i])
        }
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      // 1 被拖动的当前节点以及所在的父节点总层数不能大于3

      // 1 被拖动的当前节点总层数
      console.log('allowDrop:', draggingNode, dropNode, type)

      // var level = this.countNodeLevel(draggingNode)
      this.countNodeLevel(draggingNode)
      // 当前正在拖动的节点+父节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      console.log('深度:', deep)

      // this.maxLevel
      if (type === 'innner') {
        // console.log(
        //   `this.maxLevel: ${this.maxLevel}; draggingNode.data.catLevel:${draggingNode.data.catLevel};dropNode.level: ${dropNode.level}`
        // );
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },
    countNodeLevel (node) {
      // 找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes)
        }
      }
    },
    append (data) {
      console.log('append----', data)
      this.dialogType = 'add'
      this.title = '添加分类'
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.category.catId = null
      this.category.name = null
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
      this.dialogVisible = true
    },
    // 添加三级分类
    addCategory () {
      console.log('提交的三级分类数据', this.category)
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      })
        .then(({data}) => {
          this.$message({
            type: 'success',
            message: '菜单保存成功!'
          })
          this.dialogVisible = false
          // 刷新出新的菜单
          this.getMenus()
          this.expandedKey = [this.category.parentCid]
        })
        .catch(() => {
        })
    },
    // 修改三级分类数据
    editCategory () {
      var {catId, name, icon, productUnit} = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      })
        .then(({data}) => {
          this.$message({
            type: 'success',
            message: '菜单修改成功!'
          })
          // 关闭对话框
          this.dialogVisible = false
          // 刷新出新的菜单
          this.getMenus()
          // 设置需要默认展开的菜单
          this.expandedKey = [this.category.parentCid]
        })
        .catch(() => {
        })
    },
    edit (data) {
      console.log('要修改的数据', data)
      this.dialogType = 'edit'
      this.title = '修改分类'
      // 发送请求获取节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({data}) => {
        // 请求成功
        console.log('要回显得数据', data)
        this.category.name = data.data.name
        this.category.catId = data.data.catId
        this.category.icon = data.data.icon
        this.category.productUnit = data.data.productUnit
        this.category.parentCid = data.data.parentCid
        this.dialogVisible = true
      })
    },
    submitData () {
      if (this.dialogType === 'add') {
        this.addCategory()
      }
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },
    remove (node, data) {
      console.log('remove---', node)
      console.log('data---', data)
      var ids = [data.catId]

      this.$confirm(`是否删除【${data.name}】当前菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          })
            .then(({data}) => {
              this.$message({
                type: 'success',
                message: '菜单删除成功!'
              })
              // 刷新出新的菜单
              this.getMenus()
              this.expandedKey = [node.parent.data.catId]
            })
            .catch(() => {
            })
        })
        .catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
        })
    },
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        this.menus = data.data
      })
    }
  },
  created () {
    this.getMenus()
  }
}
</script>

<style scoped>
</style>
