<template>
  <div class="app-container">
    <el-form
      :model="queryParams"
      ref="queryForm"
      size="small"
      :inline="true"
      v-show="showSearch"
      label-width="68px"
    >
      <el-form-item label="问题" prop="question">
        <el-input
          v-model="queryParams.question"
          placeholder="请输入问题"
          clearable
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item label="答案" prop="answer">
        <el-input
          v-model="queryParams.answer"
          placeholder="请输入答案"
          clearable
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item>
        <el-button
          type="primary"
          icon="el-icon-search"
          size="mini"
          @click="handleQuery"
          >搜索</el-button
        >
        <el-button icon="el-icon-refresh" size="mini" @click="resetQuery"
          >重置</el-button
        >
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="el-icon-plus"
          size="mini"
          @click="handleAdd"
          v-hasPermi="['ai:faq:add']"
          >新增</el-button
        >
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="success"
          plain
          icon="el-icon-edit"
          size="mini"
          :disabled="single"
          @click="handleUpdate"
          v-hasPermi="['ai:faq:edit']"
          >修改</el-button
        >
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="danger"
          plain
          icon="el-icon-delete"
          size="mini"
          :disabled="multiple"
          @click="handleDelete"
          v-hasPermi="['ai:faq:remove']"
          >删除</el-button
        >
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="info"
          plain
          icon="el-icon-upload2"
          size="mini"
          @click="handleImport"
          v-hasPermi="['ai:faq:import']"
          >导入</el-button
        >
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="warning"
          plain
          icon="el-icon-download"
          size="mini"
          @click="handleExport"
          v-hasPermi="['ai:faq:export']"
          >导出</el-button
        >
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="el-icon-upload"
          size="mini"
          @click="handleImportFaq"
          v-hasPermi="['ai:faq:add_list']"
        >
          faq提取
        </el-button>
      </el-col>
      <right-toolbar
        :showSearch.sync="showSearch"
        @queryTable="getList"
      ></right-toolbar>
    </el-row>

    <el-table
      v-loading="loading"
      :data="faqList"
      @selection-change="handleSelectionChange"
    >
      <el-table-column type="selection" width="55" align="center" />
      <el-table-column label="ID" align="center" prop="id" />
      <el-table-column label="问题" align="center" prop="question" />
      <el-table-column label="答案" align="center" prop="answer" />
      <el-table-column label="创建时间" align="center" prop="created" />
      <el-table-column
        label="操作"
        align="center"
        class-name="small-padding fixed-width"
      >
        <template slot-scope="scope">
          <el-button
            size="mini"
            type="text"
            icon="el-icon-edit"
            @click="handleUpdate(scope.row)"
            v-hasPermi="['ai:faq:edit']"
            >修改</el-button
          >
          <el-button
            size="mini"
            type="text"
            icon="el-icon-delete"
            @click="handleDelete(scope.row)"
            v-hasPermi="['ai:faq:remove']"
            >删除</el-button
          >
        </template>
      </el-table-column>
    </el-table>

    <pagination
      v-show="total > 0"
      :total="total"
      :page.sync="queryParams.pageNum"
      :limit.sync="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 添加或修改FAQ管理对话框 -->
    <el-dialog :title="title" :visible.sync="open" width="500px" append-to-body>
      <el-form ref="form" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="问题" prop="question">
          <el-input v-model="form.question" placeholder="请输入问题" />
        </el-form-item>
        <el-form-item label="答案" prop="answer">
          <el-input
            v-model="form.answer"
            type="textarea"
            placeholder="请输入内容"
          />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="submitForm">确 定</el-button>
        <el-button @click="cancel">取 消</el-button>
      </div>
    </el-dialog>

    <!-- 用户导入对话框 -->
    <el-dialog :title="upload.title" :visible.sync="upload.open" width="400px">
      <el-upload
        ref="upload"
        :limit="1"
        accept=".xlsx, .xls"
        :headers="upload.headers"
        :action="upload.url + '?updateSupport=' + upload.updateSupport"
        :disabled="upload.isUploading"
        :on-progress="handleFileUploadProgress"
        :on-success="handleFileSuccess"
        :auto-upload="false"
        drag
      >
        <i class="el-icon-upload"></i>
        <div class="el-upload__text">
          将文件拖到此处，或
          <em>点击上传</em>
        </div>
        <div class="el-upload__tip" slot="tip">
          <el-checkbox
            v-model="upload.updateSupport"
          />是否更新已经存在的用户数据
          <el-link type="info" style="font-size: 12px" @click="importTemplate"
            >下载模板</el-link
          >
        </div>
        <div class="el-upload__tip" style="color: red" slot="tip">
          提示：仅允许导入“xls”或“xlsx”格式文件！
        </div>
      </el-upload>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="submitFileForm">确 定</el-button>
        <el-button @click="upload.open = false">取 消</el-button>
      </div>
    </el-dialog>

    <!--faq导入对话框-->
    <el-dialog
      title="faq提取"
      :visible.sync="faqVisible"
      :close-on-click-modal="false"
      width="800px"
    >
      <el-form v-if="activeBlock === '0'">
        <el-form-item label="session_id">
          <el-input v-model="sessionId"></el-input>
        </el-form-item>
        <el-form-item label="scene_user_id">
          <el-input v-model="sceneUserId"></el-input>
        </el-form-item>
        <el-form-item label="文件">
          <el-upload
            action="currentAction"
            ref="upload"
            drag
            name="file"
            :multiple="true"
            :on-change="changeFiles"
            :limit="1"
            :auto-upload="false"
          >
            <i class="el-icon-upload"></i>
            <div class="el-upload__text">
              将文件拖到此处，或<em>点击上传</em>
            </div>
          </el-upload>
        </el-form-item>
      </el-form>
      <div style="display: flex" v-if="activeBlock === '1'">
        <el-table :data="faqData" style="width: 100%">
          <el-table-column prop="question" label="问题"> </el-table-column>
          <el-table-column prop="answer" label="回答"> </el-table-column>
          <el-table-column fixed="right" label="操作" width="100">
            <template slot-scope="scope">
              <el-button type="text" size="small" @click="handleEdit(scope.row)"
                >编辑</el-button
              >
            </template>
          </el-table-column>
        </el-table>
      </div>
      <div v-if="activeBlock === '2'">
        <el-form :model="faqForm" label-width="80px">
          <el-form-item label="问题:">
            <el-input v-model="faqForm.faqQuestion"></el-input>
          </el-form-item>
          <el-form-item label="答案:">
            <el-input type="textarea" v-model="faqForm.faqAnswer"></el-input>
          </el-form-item>
        </el-form>
      </div>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="submitFaqFileForm">确 定</el-button>
        <el-button @click="cancelFaqFileForm">取 消</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import {
  listFaq,
  getFaq,
  delFaq,
  addFaq,
  updateFaq,
  parseFaq,
  addFaqList,
} from "@/api/ai/faq";
import { getToken } from "@/utils/auth";

export default {
  name: "Faq",
  data() {
    return {
      // 遮罩层
      loading: true,
      // 选中数组
      ids: [],
      // 非单个禁用
      single: true,
      // 非多个禁用
      multiple: true,
      // 显示搜索条件
      showSearch: true,
      // 总条数
      total: 0,
      // FAQ管理表格数据
      faqList: [],
      // 弹出层标题
      title: "",
      // 是否显示弹出层
      open: false,
      // 查询参数
      queryParams: {
        pageNum: 1,
        pageSize: 10,
        sessionId: null,
        sceneUserId: null,
        question: null,
        answer: null,
      },
      // 表单参数
      form: {},
      // 表单校验
      rules: {},
      // 用户导入参数
      upload: {
        // 是否显示弹出层（用户导入）
        open: false,
        // 弹出层标题（用户导入）
        title: "",
        // 是否禁用上传
        isUploading: false,
        // 是否更新已经存在的用户数据
        updateSupport: 0,
        // 设置上传的请求头部
        headers: { Authorization: "Bearer " + getToken() },
        // 上传的地址
        url: process.env.VUE_APP_BASE_API + "/ai/faq/importData",
      },
      //faq提取start
      faqVisible: false,
      sessionId: "1",
      sceneUserId: "tax",
      fileList: [],
      activeBlock: "0",
      faqData: [],
      faqForm: {
        faqAnswer: "",
        faqQuestion: "",
      },
      // faq提取end
    };
  },
  created() {
    this.getList();
  },
  methods: {
    /** 查询FAQ管理列表 */
    getList() {
      this.loading = true;
      listFaq(this.queryParams).then((response) => {
        this.faqList = response.rows;
        this.total = response.total;
        this.loading = false;
      });
    },
    // 取消按钮
    cancel() {
      this.open = false;
      this.reset();
    },
    // 表单重置
    reset() {
      this.form = {
        id: null,
        sessionId: null,
        sceneUserId: null,
        question: null,
        answer: null,
        created: null,
      };
      this.resetForm("form");
    },
    /** 搜索按钮操作 */
    handleQuery() {
      this.queryParams.pageNum = 1;
      this.getList();
    },
    /** 重置按钮操作 */
    resetQuery() {
      this.resetForm("queryForm");
      this.handleQuery();
    },
    // 多选框选中数据
    handleSelectionChange(selection) {
      this.ids = selection.map((item) => item.id);
      this.single = selection.length !== 1;
      this.multiple = !selection.length;
    },
    /** 新增按钮操作 */
    handleAdd() {
      this.reset();
      this.open = true;
      this.title = "添加FAQ管理";
    },
    /** 修改按钮操作 */
    handleUpdate(row) {
      this.reset();
      const id = row.id || this.ids;
      getFaq(id).then((response) => {
        this.form = response.data;
        this.open = true;
        this.title = "修改FAQ管理";
      });
    },
    /** 提交按钮 */
    submitForm() {
      this.$refs["form"].validate((valid) => {
        if (valid) {
          if (this.form.id != null) {
            updateFaq(this.form).then((response) => {
              this.$modal.msgSuccess("修改成功");
              this.open = false;
              this.getList();
            });
          } else {
            addFaq(this.form).then((response) => {
              this.$modal.msgSuccess("新增成功");
              this.open = false;
              this.getList();
            });
          }
        }
      });
    },
    /** 删除按钮操作 */
    handleDelete(row) {
      const ids = row.id || this.ids;
      this.$modal
        .confirm('是否确认删除FAQ管理编号为"' + ids + '"的数据项？')
        .then(function () {
          return delFaq(ids);
        })
        .then(() => {
          this.getList();
          this.$modal.msgSuccess("删除成功");
        })
        .catch(() => {});
    },
    /** 导入按钮操作 */
    handleImport() {
      this.upload.title = "用户导入";
      this.upload.open = true;
    },
    /** faq提取start */
    handleImportFaq() {
      this.faqVisible = true;
    },
    changeFiles(_, fileList) {
      this.fileList = fileList;
    },
    handleEdit(row) {
      (this.faqForm = {
        faqQuestion: row.question,
        faqAnswer: row.answer,
      }),
        (this.activeBlock = "2");
    },
    submitFaqFileForm() {
      if (this.activeBlock === "0") {
        const files = this.fileList;
        if (files.length === 0) {
          this.$message({
            message: "请选择文件",
            type: "warning",
          });
          return;
        }
        let fileData = new FormData();
        for (let i = 0; i <= files.length - 1; i++) {
          fileData.set("file", files[i].raw);
          parseFaq(this.sessionId, this.sceneUserId, fileData).then(
            (response) => {
              if (
                response.status === 200 &&
                response.data.questions?.length > 0
              ) {
                this.activeBlock = "1";
                this.faqData = response.data.questions.map((item, index) => ({
                  question: item,
                  answer: response.data.answers[index],
                }));
              } else {
                this.$notify.error("解析失败");
              }
            }
          );
        }
      } else if (this.activeBlock === "2") {
        addFaqList([
          {
            question: this.faqForm.faqQuestion,
            answer: this.faqForm.faqAnswer,
            sessionId: this.sessionId,
            sceneUserId: this.sceneUserId,
          },
        ]).then((res) => {
          if (res.code === 200) {
            this.$notify.success("修改成功");
            this.activeBlock = "1";
            // 需要重新查更新一下当前列表
          } else {
            this.$notify.error("修改失败");
          }
        });
      }
    },
    cancelFaqFileForm() {
      if (this.activeBlock === "2") {
        this.activeBlock = "1";
      } else if (this.activeBlock === "1") {
        this.faqVisible = false;
        this.activeBlock = "0";
      }
    },
    /** faq提取end */
    /** 下载模板操作 */
    importTemplate() {
      this.download(
        "ai/faq/importTemplate",
        {},
        `faq_template_${new Date().getTime()}.xlsx`
      );
    },
    // 文件上传中处理
    handleFileUploadProgress(event, file, fileList) {
      this.upload.isUploading = true;
    },
    // 文件上传成功处理
    handleFileSuccess(response, file, fileList) {
      this.upload.open = false;
      this.upload.isUploading = false;
      this.$refs.upload.clearFiles();
      this.$alert(response.msg, "导入结果", { dangerouslyUseHTMLString: true });
      this.getList();
    },
    // 提交上传文件
    submitFileForm() {
      this.$refs.upload.submit();
    },
    /** 导出按钮操作 */
    handleExport() {
      this.download(
        "ai/faq/export",
        {
          ...this.queryParams,
        },
        `faq_${new Date().getTime()}.xlsx`
      );
    },
  },
};
</script>
