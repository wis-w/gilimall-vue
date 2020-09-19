<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>

    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      :draggable="draggable"
      :allow-drop="allowDrop"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level<=2" type="text" size="mini" @click="() => append(data)">Append</el-button>
          <el-button type="text" size="mini" @click="() => update(data)">Update</el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
        </span>
      </span>
    </el-tree>
    <!--菜单表单增改表单-->
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="categery">
        <el-form-item label="分类名称">
          <el-input v-model="categery.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="categery.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="categery.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="refresh">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      pCid: [],
      draggable: false,
      updataNodes: [],
      maxLevel: 0,
      title: "",
      dialogVisibleType: "",
      categery: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        productUnit: "",
      },
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  //计算属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //方法集合
  methods: {
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        // console.log("成功的获取到了后台系统的数据...",data.data);
        this.menus = data.data;
      });
    },
    // 批量删除
    batchDelete() {
      let checkNodes = this.$refs.menuTree.getCheckedNodes();
      let catIds = [];
      console.log("被选中的节点", checkNodes);
      for (let i = 0; i < checkNodes.length; i++) {
        catIds.push(checkNodes[i].catId);
      }
      this.$confirm(`是否批量删除【${catIds}】菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.$message({
              type: "success",
              message: "批量删除成功",
            });
            this.getMenu();
          });
        })
        .catch(() => {

        });
    },
    // 保存
    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updataNodes, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "菜单顺序等修改成功",
        });
        this.getMenu(); // 刷新页面
        this.expandedKey = this.pCid; // 设置默认展开的菜单
        this.updataNodes = [];
        this.maxLevel = 0;
        // this.pcid = [];
      });
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("handleDrop: ", dropNode, dropNode, dropType);
      // 当前节点最新的父节点ID
      let pCid = 0;
      let sibling = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        sibling = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        sibling = dropNode.childNodes;
      }
      this.pCid.push(pCid);

      // 当前拖拽的节点的最新顺序
      for (let i = 0; i < sibling.length; i++) {
        if (sibling[i].data.catId == draggingNode.data.catId) {
          // 如果遍历的时正在拖拽的节点
          let catLevel = draggingNode.level;
          if (sibling[i].level != draggingNode.level) {
            // 当前节点的层级发生了变化
            catLevel = sibling[i].level;
            // 修改子节点的层级
            this.updateChildNodeLevel(sibling[i]);
          }
          this.updataNodes.push({
            catId: sibling[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updataNodes.push({ catId: sibling[i].data.catId, sort: i });
        }
      }
      // 当前拖拽节点的层级
      console.log("updataNodes: ", this.updataNodes);
    },
    // 修改子节点的层级
    updateChildNodeLevel(node) {
      if (node != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updataNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    // 是否被允许拖拽到指定位置
    allowDrop(draggingNode, dropNode, type) {
      //1、被拖动的当前节点以及所在的父节点总层数不能大于3
      //1）、被拖动的当前节点总层数
      this.countNodeLevel(draggingNode);
      //当前正在拖动的节点+父节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;

      //   this.maxLevel
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      //找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    // 确当按钮的功能确认
    submitData() {
      if (this.dialogVisibleType == "update") {
        this.updateCategory(); // 修改菜单操作
      }
      if (this.dialogVisibleType == "append") {
        this.addCategory(); // 增加菜单操作
      }
    },
    // 修改三级菜单列表
    update(data) {
      this.title = "修改";
      this.dialogVisible = true;
      this.dialogVisibleType = "update";
      // 获取当前节点的最新数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        this.categery = data.data;
      });
    },
    updateCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(this.categery, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "修改成功",
        });
        this.refresh(); // 关闭对话框，并将菜单置为初始状态
        this.getMenu(); // 刷新页面
        this.expandedKey = [this.categery.parentCid]; // 设置默认展开的菜单
      });
    },
    // 添加三级分类
    append(data) {
      this.title = "新增";
      this.dialogVisible = true;
      this.categery.parentCid = data.catId;
      this.categery.catLevel = data.catLevel * 1 + 1;
      this.dialogVisibleType = "append";
    },
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.categery, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "添加成功",
        });
        this.refresh(); // 关闭对话框，并将菜单置为初始状态
        this.getMenu(); // 刷新页面
        this.expandedKey = [this.categery.parentCid]; // 设置默认展开的菜单
      });
    },
    // 初始化数据
    refresh() {
      this.categery = {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        productUnit: "",
      };
      this.dialogVisible = false;
    },

    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              type: "success",
              message: "删除成功",
            });
            this.getMenu(); // 刷新页面
            this.expandedKey = [node.parent.data.catId]; // 设置默认展开的菜单
          });
        })
        .catch(() => {
          this.$message("取消删除");
        });
    },
  },
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenu();
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>