<template>
  <div>
    <!--
      节点被点击时的回调 node-click
      共三个参数，依次为：传递给 data 属性的数组中该节点所对应的对象、节点对应的 Node、节点组件本身。
    -->
    <el-tree
      :data="menus"
      :props="defaultProps"
      node-key="catId"
      @node-click="nodeclick"
      ref="menuTree">
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
    nodeclick (data, node, component) {
      //console.log('子组件被点击', data, node, component)
      this.$emit('tree-node-click', data, node, component)
    },
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
