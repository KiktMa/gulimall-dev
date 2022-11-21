<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="treeMenu"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
          <el-button type="text" size="mini" @click="edit(data)">
            Edit
          </el-button>
        </span>
      </span></el-tree
    >

    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%">
      <el-form :model="categroy">
        <el-form-item label="商品类别">
          <el-input v-model="categroy.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="categroy.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="categroy.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="bianji">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import Vue from "vue";
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxlevel: 0,
      leibie: "",
      title: "",
      menus: [],
      expandedKey: [],
      dialogVisible: false,
      categroy: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: 0,
        icon: null,
        productUnit: "",
      },
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },

  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("获取到列表数据", data.data);
        this.menus = data.data;
      });
    },

    // 批量删除
    batchDelete() {
      let catIds = [];
      let checkbox = this.$refs.treeMenu.getCheckedNodes();
      for (let i = 0; i < checkbox.length; i++) {
        catIds.push(checkbox[i].catId);
      }
      this.$confirm(`是否删除这【${catIds.length}个节点】?`, "提示", {
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
              message: "批量删除成功",
              typez: "success",
            });
            this.getMenus();
          });
        })
        .catch(() => {});
    },

    // 拖拽节点后的信息处理
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("handleDrop: ", dropNode.label, dropType);
      // 获取节点的最新的父节点
      let pCid = 0;
      let siblings = null;
      if (dropNode == "before" || dropNode == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
        siblings = dropNode.childNodes;
      }

      this.pCid.push(pCid);

      // 当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        // 如果遍历的是当前正在拖拽的节点
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            // 当前节点的层级发生变化
            catLevel = siblings[i].level;
            // 修改它子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].catId, sort: i });
        }
      }

      // 当前拖拽节点的最新层级
      console.log("updateNodes:", this.updateNodes);
    },

    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.getMenus();
        this.expandedKey = this.pCid;
        this.updateNodes = [];
        this.maxlevel = 0;
        // this.pCid = 0;
      });
    },

    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: cNode.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    // 判断是修改还是添加
    bianji() {
      if (this.leibie == "add") {
        this.addCategory();
      }
      if (this.leibie == "edit") {
        this.editCategory();
      }
    },

    // 修改类别
    editCategory() {
      var { catId, name, icon, productUnit } = this.categroy;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.categroy.catId];
      });
    },

    // 添加类别
    addCategory() {
      console.log("提交的三级分类数据", this.categroy);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.categroy, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [node.parent.data.catId];
      });
    },

    // 编辑前获取修改后的最新数据
    edit(data) {
      console.log("修改的最新数据：", data);
      this.dialogVisible = true;
      this.title = "修改";
      this.leibie = "edit";
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        this.categroy.name = data.data.name;
        this.categroy.catId = data.data.catId;
        this.categroy.icon = data.data.icon;
        this.categroy.productUnit = data.data.productUnit;
        this.categroy.parentCid = data.data.parentCid;
      });
    },

    // 添加类别
    append(data) {
      console.log("append", data);
      this.title = "添加";
      this.leibie = "add";
      this.dialogVisible = true;
      this.categroy.parentCid = 0;
      this.categroy.catLevel = 0;
      this.categroy.sort = 0;
      this.categroy.productUnit = "";
      this.categroy.icon = null;
      this.categroy.showStatus = 1;
      this.categroy.name = "";
    },

    // 删除类别
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}菜单】?`, "提示", {
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
              message: "菜单删除成功",
              type: "success",
            });
            this.getMenus();
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
      console.log("remove:", node, data);
    },

    // 层级拖拽功能
    allowDrop(draggingNode, dropNode, type) {
      console.log("draggingNode:", draggingNode, dropNode, type);
      this.countLevel(draggingNode.data);
      console.log("深度：", this.maxlevel);
      let deep = Math.abs(this.maxlevel - draggingNode.level) + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },

    // 获取当前拖拽节点的最大层级
    countLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxlevel) {
            this.maxlevel = node.childNodes[i].catLevel;
          }
          this.countLevel(node.childNodes[i]);
        }
      }
    },
  },

  //计算属性 类似于 data 概念
  computed: {},
  //监控 data 中的数据变化
  watch: {},
  //方法集合

  //生命周期 - 创建完成（可以访问当前 this 实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有 keep-alive 缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>