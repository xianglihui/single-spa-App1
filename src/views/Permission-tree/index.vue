<template>
  <div>
    <!-- header -->
    <common-page-header :title="title" :showSearchHandle="true">
      <!-- <div class="headerBtnWrap">
          <el-button class="el-icon-upload" size="mini">
            导入权限文件
          </el-button>
      </div> -->
    </common-page-header>
    <!-- main -->
    <div class="auth-tree-con">
      <div class="auth-tree">
        <el-input
          placeholder="输入关键字进行过滤"
          v-model="filterText"
          size="mini"
          clearable
        ></el-input>
        <div class="tree-control">
          <el-button
            @click="collapseAll"
            type="text"
            size="mini"
            icon="el-icon-arrow-up"
            >折叠</el-button
          >
          <el-button
            @click="unFoldAll"
            type="text"
            size="mini"
            icon="el-icon-arrow-down"
            >展开</el-button
          >
        </div>
        <el-tree
          class="tree"
          :data="treeData"
          :props="defaultProps"
          :filter-node-method="filterNode"
          @node-click="handleNodeClick"
          highlight-current
          node-key="id"
          ref="tree"
        ></el-tree>
      </div>
      <!-- btn -->
      <div class="middle">
        <el-button
          size="mini"
          @click="handleAddRole"
          type="primary"
          icon="el-icon-user"
          >授权给角色</el-button
        >
        <el-button
          size="mini"
          @click="handleCreate"
          type="primary"
          icon="el-icon-plus"
          >新增权限</el-button
        >
        <el-button
          size="mini"
          @click="handleDelete"
          type="danger"
          icon="el-icon-delete"
          >删除权限</el-button
        >
        <br />
        <!-- tip -->
        <span
          :class="
            operationType === 'edit'
              ? 'edit hint'
              : operationType === 'create'
              ? 'create hint'
              : ''
          "
        >
          {{
            operationType === "edit"
              ? "修改【" + currentNode.name + "】权限"
              : ""
          }}
          {{
            operationType === "create"
              ? "【" + currentNode.name + "】权限下新增"
              : ""
          }}
        </span>
        <!-- form -->
        <el-form
          ref="refForm"
          :model="form"
          label-width="80px"
          size="mini"
          class="form"
        >
          <!-- {{ form }} -->
          <el-form-item label="名称">
            <el-input v-model="form.name"></el-input>
          </el-form-item>
          <el-form-item label="code" prop="code">
            <el-input v-model="form.code"></el-input>
          </el-form-item>
          <el-form-item label="icon" prop="icon">
            <el-input v-model="form.icon"></el-input>
          </el-form-item>
          <el-form-item label="url" prop="url">
            <el-input v-model="form.url"></el-input>
          </el-form-item>
          <el-form-item label="类型" prop="featureType">
            <el-select v-model="form.featureType" placeholder="请选择">
              <el-option
                v-for="item in typeList"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              >
              </el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="排序" prop="orderIndex">
            <el-input v-model="form.orderIndex"></el-input>
          </el-form-item>
          <el-form-item label="是否折叠" prop="isCollapse">
            <el-switch
              v-model="form.isCollapse"
              active-color="#409EFF"
              inactive-color="#909399"
              active-text="是"
              inactive-text="否"
            >
            </el-switch>
          </el-form-item>
          <el-form-item>
            <el-button
              type="primary"
              @click="submitForm('form')"
              v-if="operationType"
              >保存{{ operationType === "edit" ? "编辑" : "新增" }}</el-button
            >
          </el-form-item>
        </el-form>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { reactive, toRefs, ref, onMounted } from "vue";
import commonPageHeader from "@/components/common/CommonPageHeader.vue";
import { useCurrentInstance } from "@/utils/toolset";
import { ElMessageBox, ElMessage } from "element-plus";
export default {
  components: {
    commonPageHeader,
  },
  setup() {
    const state = reactive({
      title: "权限管理",
      treeData: [], //权限树
      form: {
        name: "",
        code: "",
        icon: "",
        url: "",
        featureType: 0, // 类型
        orderIndex: 0, // 排序
        namePinyin: "",
        isCollapse: false,
        id: "",
      },
      defaultProps: {
        children: "children",
        label: "name",
      },
      currentNode: "", // 当前点击的node
      level4: [], // 第四级 按钮级权限
      operationType: "", // 操作类型 create edit
      deleteAble: false, // 是否可删除
      createAble: false, // 是否可删除
      typeList: [
        { label: "隐藏", value: 0 }, // 隐藏菜单
        { label: "应用", value: 1 }, // 子模块A，B...
        { label: "模块", value: 2 },
        { label: "菜单", value: 3 }, // 侧边栏菜单
        { label: "设置", value: 4 }, // 全局设置
      ],
      test: "test",
    });
    const { proxy } = useCurrentInstance(); // 拿全局api
    const filterText = ref(""); // 过滤关键词
    console.log("proxy", proxy.$authApi);
    // 全部折叠
    const collapseAll = () => {
      console.log("全部折叠");
    };
    // 全部展开
    const unFoldAll = () => {
      console.log("全部展开");
    };
    const resetForm = (formName: any) => {
      // $refs[formName].resetFields();
      console.log("formName");
    };
    /**
     * 获取权限树
     * 例如CentralSystem 作为一级系统（允许出现多个一级系统），应需如下基本字段
     * 系统级别：id:标识唯一性，url：作为路由跳转路径，name:系统名称，featureType:模块类型，icon：图标，code：权限code，parentCode：父级code,children:子集
     * 应用级别：作为系统级别的children，上同
     * 页面级别：作为应用级别的children，上同
     * 按钮级别，作为页面级别的children，出icon外,上同
     */
    const getAuthTree = () => {
      proxy.$authApi
        .getAuthTree()
        .then((res: any) => {
          console.log("res", res);
          state.treeData = res.data;
        })
        .finally(() => {
          state.operationType = "";
          console.log("finally");
          resetForm("form");
        });
    };
    // 筛选
    const filterNode = (value: any, data: any) => {
      if (!value) {
        return true;
      }
      return data.name.indexOf(value) !== -1;
    };
    // 节点被点击时的回调
    const handleNodeClick = (data: any, node: any) => {
      state.level4 = [];
      if (node.level === 3) {
        for (const i of node.childNodes) {
          (state.level4 as any).push({ key: i.key, label: i.label });
        }
      }
      state.currentNode = data;
      state.operationType = "edit";
      // 回显
      state.form = { ...data };
      state.form.name = "1";
      console.log("state.form", state.form);
      state.deleteAble = true;
      state.createAble = true;
    };

    /**
     * 授权给角色，整体业务逻辑如下
     * 1、authority分为权限树，角色管理，用户管理
     * 2、角色管理主要提供创建/编辑角色功能，并且给角色增加/修改权限，权限来源于权限树，角色与权限关系为一对多，即一个角色下可拥有多个权限
     * 3、用户管理主要提供创建，禁用/启用功能，并且给用户增加/修改角色，角色来源于角色管理，用户于角色管理为一对多，即一个用户下可分配多个权限
     * 4、将角色分配给权限树上的页面/按钮级别，也就是为权限树上的权限添加角色，该角色拥有某一权限，就拥有相应code,能够在平台看到相应权限的页面/按钮
     */
    const handleAddRole = () => {
      console.log("添加角色");
    };
    // 新增
    const handleCreate = () => {
      console.log("新增");
      // state.currentNode = JSON.parse(JSON.stringify(state.form));
      // const code = `${state.form.code}.`;
      // state.operationType = "create";
      // resetForm("form");
      // state.form.code = code;
      // state.deleteAble = false;
    };
    // 删除
    const handleDelete = () => {
      ElMessageBox.confirm("确认删除该权限, 是否继续?", "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then(() => {
        deletePermission();
      });
      // .catch(() => {});
    };
    // 删除
    const deletePermission = () => {
      console.log("删除");
    };
    // 提交表单
    const submitForm = () => {
      console.log("提交表单");
    };
    onMounted(() => {
      getAuthTree();
    });
    return {
      ...toRefs(state),
      filterText,
      collapseAll,
      unFoldAll,
      filterNode,
      handleNodeClick,
      handleAddRole,
      handleCreate,
      handleDelete,
      submitForm,
    };
  },
};
</script>

<style lang="less" scoped>
.auth-tree-con {
  display: flex;
  justify-content: flex-start;
  padding: 5px;

  .auth-tree {
    display: inline-block;
    width: 280px;

    // background-color: #e4e4e4 !important;
    .tree-control {
      display: flex;
      justify-content: flex-end;
    }

    .tree {
      overflow-y: scroll;
    }
  }

  .middle {
    // width: calc(100% - 210px);
    width: 520px;
    margin: 0 0 0 10px;

    .edit {
      color: #e6a23c;
    }

    .create {
      color: #67c23a;
    }

    .hint {
      display: inline-block;
      margin: 20px 0 0 40px;
      height: 30px;
      line-height: 30px;
      font-size: 20px;
    }

    .form {
      width: 500px;
      margin: 10px;
    }
  }

  .right {
    margin: 0 0 0 10px;

    .right-item {
      margin: 10px 0;
      color: #4c4c4c;
    }
  }
}
</style>
