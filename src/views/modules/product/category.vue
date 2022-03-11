<template>
  <div>
    <!--
      node-key
      每个树节点用来作为唯一标识的属性，整棵树应该是唯一的
      default-expanded-keys 默认展开的节点的 key 的数组
    -->
    <el-tree :data="menus"
             :props="defaultProps"
             show-checkbox
             node-key="catId"
             :default-expanded-keys="expandedKey"
             :expand-on-click-node="false">

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
            type="text"
            size="mini"
            style="color: #67C23A"
            v-if="node.level<=2"
            @click="() => append(data)">
            添加
          </el-button>
          <el-button
            type="text"
            size="mini"
            v-if="node.childNodes.length==0"
            style="color: #F56C6C"
            @click="() => remove(node, data)">
            删除
          </el-button>
        </span>
      </span>

    </el-tree>
  </div>
</template>

<script>
export default {
  name: 'category',
  data () {
    return {
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  created () {
    this.getMenus()
  },
  methods: {
    // 添加结点
    append (data) {

    },

    // 删除结点
    remove (node, data) {
      // 获取当前结点的id值
      let ids = [data.catId]
      this.$confirm('是否删除【' + data.name + '】菜单, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          this.$message({
            message: '恭喜你，这是一条成功消息',
            type: 'success'
          })
          // 刷新出新的菜单
          this.getMenus()
          // 刷出需要默认展开的菜单--被删除菜单的父菜单,找到当前结点的父节点
          this.expandedKey = [node.parent.data.catId]
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
    },

    // 获取菜单
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        this.menus = data.data
      })
    }
  }
}
</script>

<style scoped>

</style>
