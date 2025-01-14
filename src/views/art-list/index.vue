<template>
  <div>
    <!-- 本身的页面卡片 -->
    <el-card>
      <div slot="header">
        <span>文章列表</span>
      </div>
      <div>
        <el-form inline :model="q">
          <el-form-item label="文章分类" style="margin-right: 25px">
            <el-select
              v-model="q.cate_id"
              placeholder="请选择分类"
              @click="getCateData"
            >
              <el-option
                :label="obj.cate_name"
                :value="obj.id"
                v-for="obj in cateFrom"
                :key="obj.id"
              ></el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="发布状态">
            <el-select v-model="q.state" placeholder="请选择状态">
              <el-option label="已发布" value="已发布"></el-option>
              <el-option label="草稿" value="草稿"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="chooseFn">筛选</el-button>
            <el-button type="info" @click="resetFn">重置</el-button>
          </el-form-item>
          <el-form-item style="float: right">
            <el-button type="primary" @click="showPubDialogFn"
              >发表文章</el-button
            >
          </el-form-item>
        </el-form>
      </div>
      <el-table :data="artList" style="width: 100%" border stripe>
        <el-table-column label="文章标题">
          <template v-slot="{ row }">
            <el-link type="primary" @click="getArticleDetailFn(row.id)">{{
              row.title
            }}</el-link>
          </template>
        </el-table-column>
        <el-table-column label="分类" prop="cate_name"></el-table-column>
        <el-table-column label="发表时间" prop="pub_date">
          <template v-slot="{ row }">
            <span>{{ $formatDate(row.pub_date) }}</span>
          </template>
        </el-table-column>
        <el-table-column label="状态" prop="state"></el-table-column>
        <el-table-column label="操作">
          <!-- 记住作用域插槽写法 v-slot 会有一个scope对象 解构赋值出里面的row -->
          <template v-slot="{ row }">
            <el-button type="danger" @click="deleteArt(row.id)" size="mini"
              >删除</el-button
            >
          </template>
        </el-table-column>
      </el-table>
      <!-- 分页区域 -->
      <el-pagination
        @size-change="handleSizeChangeFn"
        @current-change="handleCurrentChangeFn"
        :current-page.sync="q.pagenum"
        :page-sizes="[2, 3, 5, 10]"
        :page-size.sync="q.pagesize"
        layout="total, sizes, prev, pager, next, jumper"
        :total="total"
      >
      </el-pagination>
    </el-card>
    <!-- 写文章弹出来的对话框 -->
    <el-dialog
      title="发表文章"
      :visible.sync="pubDialogVisible"
      fullscreen
      :before-close="handleClose"
      @close="dialogCloseFn"
    >
      <!-- 发布文章的对话框 -->
      <el-form
        :model="pubForm"
        :rules="pubFormRules"
        ref="pubFormRef"
        label-width="100px"
      >
        <el-form-item label="文章标题" prop="title">
          <el-input v-model="pubForm.title" placeholder="请输入标题"></el-input>
        </el-form-item>
        <el-form-item label="文章分类" prop="cate_id">
          <el-select
            v-model="pubForm.cate_id"
            placeholder="请选择分类"
            style="width: 100%"
          >
            <el-option
              :label="obj.cate_name"
              :value="obj.id"
              v-for="obj in cateFrom"
              :key="obj.id"
            ></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="文章内容" prop="content">
          <quill-editor
            v-model="pubForm.content"
            @blur="contentChangeFn"
          ></quill-editor>
        </el-form-item>
        <el-form-item label="文章封面" prop="cover_img">
          <!-- 用来显示封面的图片 -->
          <img src="@/assets/cover.jpg" class="cover-img" ref="imgRef" />
          <br />
          <!-- 文件选择框，默认被隐藏 -->
          <input
            type="file"
            style="display: none"
            accept="image/*"
            ref="iptFileRef"
            @change="changeCover"
          />
          <!-- 选择封面的按钮 -->
          <el-button type="text" @click="selCover">+ 选择封面</el-button>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="pubArticleFn('已发布')"
            >发布</el-button
          >
          <el-button type="info" @click="pubArticleFn('草稿')"
            >存为草稿</el-button
          >
        </el-form-item>
      </el-form>
    </el-dialog>
    <!-- 查看文章详情的对话框 -->
    <el-dialog title="文章预览" :visible.sync="detailVisible" width="80%">
      <h1 class="title">{{ artDetail.title }}</h1>

      <div class="info">
        <span>作者：{{ artDetail.nickname || artDetail.username }}</span>
        <span>发布时间：{{ $formatDate(artDetail.pub_date) }}</span>
        <span>所属分类：{{ artDetail.cate_name }}</span>
        <span>状态：{{ artDetail.state }}</span>
      </div>

      <!-- 分割线 -->
      <el-divider></el-divider>

      <!-- 文章的封面 -->
      <img :src="baseURL + artDetail.cover_img" alt="" />
      <!-- 后端返回图片链接地址的经验://为何后端返回的图片地址是半截的?
            原因:因为服务器的域名可能会来回变化，所以数据库里的地址只有相对路径
            要求:前端请求此图片的时候，后端告诉你图片文件真身所在的服务器域名，前端目己拼按丽缴
       -->
      <!-- 文章的详情 -->
      <div v-html="artDetail.content" class="detail-box"></div>
    </el-dialog>
  </div>
</template>

<script>
import {
  getCateListAPI,
  addArticleAPI,
  getArticleListAPI,
  getArticleDetailAPI,
  deleteArticleAPI
} from '@/api'
import defaultImg from '@/assets/cover.jpg'
import { baseURL } from '@/utils/request'
export default {
  name: 'art-list',
  data () {
    return {
      pubDialogVisible: false,
      pubForm: {
        // 表单的数据对象
        title: '', // 文章标题
        cate_id: '', // 文章分类id
        content: '', // 文章内容
        cover_img: '', // 文章封面
        state: '' // 文章状态 草稿或已发布
      },
      pubFormRules: {
        // 表单的验证规则对象
        title: [
          { required: true, message: '请输入文章标题', trigger: 'blur' },
          {
            min: 1,
            max: 30,
            message: '文章标题的长度为1-30个字符',
            trigger: 'blur'
          }
        ],
        cate_id: [
          { required: true, message: '请选择文章分类', trigger: 'change' }
        ],
        content: [
          // content对应quill-editor富文本编辑器，它不是el提供表单标签
          // quill-editor2个事件都没有，所以你输入内容也不会自动走校验//解决:
          // 自己来给quill-editor绑定change事件(在文档里查到的它支持change事件内容改变事件)
          { required: true, message: '请输入文章内容', trigger: 'blur' }
        ],
        cover_img: [
          { required: true, message: '请选择文章封面', trigger: 'blur' }
        ]
      },
      cateFrom: [],
      artList: [], // 文章的列表数据
      total: 0, // 总数据条数
      q: {
        pagenum: 1, // 默认拿第一页的数据
        pagesize: 2, // 当前页需要几条数据
        cate_id: '', // 分类id
        state: '' // 状态
      },
      detailVisible: false, // 控制文章详情对话框的显示与隐藏
      artDetail: {}, // 文章的详情信息对象
      baseURL: baseURL
    }
  },
  methods: {
    // 对话框关闭前的回调
    async handleClose (done) {
      // done的作用:调用就会关闭对话框
      // 询问用户是否确认关闭对话框
      const confirmResult = await this.$confirm(
        '此操作将导致文章信息丢失, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).catch((err) => err)
      // $confirm内部虽然是一个确认提示框，但是它借用了Promise语法来管理，点击确定它的状态为兑现成功状态返回'confirm'，
      // 如果用户点击了取消按钮，此Promise对象状态为拒绝状态，返回'cancel'字符串
      // 因为await只能捕获成功的结果，用catch可以把cancel字符串变成成功的结果 就被捕获到了
      // 因为catch把所有非Promise的值返回为成功，（对↑的补充）

      // 取消了关闭-阻止住, 什么都不干
      if (confirmResult === 'cancel') return
      // 确认关闭
      done()
    },
    showPubDialogFn () {
      this.pubDialogVisible = true
    },
    // 获取文章分类数据
    async getCateData () {
      const { data: res } = await getCateListAPI()
      this.cateFrom = res.data
    },
    selCover () {
      this.$refs.iptFileRef.click()
    },
    changeCover (e) {
      const files = e.target.files
      if (files.length === 0) {
        this.pubForm.cover_img = null
        this.$refs.imgRef.setAttribute('src', defaultImg)
      } else {
        this.pubForm.cover_img = files[0]
        const url = URL.createObjectURL(files[0])
        this.$refs.imgRef.setAttribute('src', url)
      }
      this.$refs.pubFormRef.validateField('cover_img')
    },
    // 发布或存为草稿
    pubArticleFn (str) {
      this.$refs.pubFormRef.validate(async (valid) => {
        if (valid) {
          this.pubForm.state = str
          const fd = new FormData() // 准备一个表单数据对象的容器，FormData类是HTML5新出的专门为了装文件内容的一个容器
          // fd.append("参数名"，值)
          fd.append('title', this.pubForm.title)
          fd.append('cate_id', this.pubForm.cate_id)
          fd.append('content', this.pubForm.content)
          fd.append('cover_img', this.pubForm.cover_img)
          fd.append('state', this.pubForm.state)
          const { data: res } = await addArticleAPI(fd)
          console.log(res)
          if (res.code !== 0) return this.$message.error('发布文章失败！')
          this.$message.success('发布文章成功！')
          this.pubDialogVisible = false
          // 关闭对话框
          this.initArtListFn()
          // 发布后重新获取文章数据
        } else return false
      })
    },
    // 富文本编辑器内容改变触发此事件方法
    contentChangeFn () {
      this.$refs.pubFormRef.validateField('content')
    },
    // 新增文章的对话框关闭后清空表单
    dialogCloseFn () {
      this.$refs.pubFormRef.resetFields()
      // 手动给封面标签清空
      this.$refs.imgRef.setAttribute('src', defaultImg)
    },
    // 初始化文章列表
    async initArtListFn () {
      const { data: res } = await getArticleListAPI(this.q)
      if (res.code !== 0) return this.$message.error('获取文章列表失败!')
      this.artList = res.data
      this.total = res.total
    },
    // pageSize 发生了变化
    handleSizeChangeFn (newSize) {
      // 为 pagesize 赋值
      this.q.pagesize = newSize
      // 问题:先点击最后一个页码，切换每页显示条数2->3，总数不够，分页只能分到2
      // 每页条数改变了，页码从3到2页改变了，2个事件都会触发
      // 偶发性的bug:有的时候自动早到第二页有数据有的时候没有
      // 知识点:2个网络请求一起发，谁先回来不一定
      // 解决:当切换每页显示的条数，我们就把当前页码设置为1，而且标签里要设置
      // 原因:所以可能第2页3条数据回来有值铺设，紧接着第3页的3条数据回来了，空数组所以页面就是空的
      this.q.pagenum = 1
      // 重新发起请求
      this.initArtListFn()
    },
    // 核心思想:根据选择的页码/条数，影响q对象对应属性的值，再重新发一次请求让后台重新返回数据

    // 页码值发生了变化
    handleCurrentChangeFn (newPage) {
      // 为页码值赋值
      this.q.pagenum = newPage
      // 重新发起请求

      this.initArtListFn()
    },
    // 筛选按钮
    chooseFn () {
      this.q.pagenum = 1
      this.q.pagesize = 2
      this.initArtListFn()
    },
    // 重置按钮
    resetFn () {
      this.q.pagenum = 1
      this.q.pagesize = 2
      this.q.cate_id = ''
      this.q.state = ''
      this.initArtListFn()
    },
    // 点击文章标题进入文章详情
    async getArticleDetailFn (id) {
      const { data: res } = await getArticleDetailAPI(id)
      console.log(res.data)
      this.artDetail = res.data
      this.detailVisible = true
    },
    // 删除文章
    async deleteArt (id) {
      // const {data:res}=await deleteArticleAPI(id)
      // console.log(res)
      // if(res.code!==0) return this.$message.error(res.message)
      // this.$message.success(res.message)
      // this.initArtListFn()
      // 1. 询问用户是否要删除
      const confirmResult = await this.$confirm(
        '此操作将永久删除该文件, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).catch((err) => err)

      // 2. 取消了删除
      if (confirmResult === 'cancel') return

      // 执行删除的操作
      const { data: res } = await deleteArticleAPI(id)

      if (res.code !== 0) return this.$message.error('删除失败!')
      this.$message.success('删除成功!')
      // 刷新列表数据

      // 判断最后一页不止一条时 不需要往前跳
      // 当删除最后一页的最后一条时  页面需要往前跳一页
      // 当页面为第一页时 不需要往前跳
      if (this.total % this.q.pagesize === 1) {
        if (this.q.pagenum > 1) {
          this.q.pagenum--
        }
      }
      this.initArtListFn()
    }
  },
  created () {
    this.getCateData()
    this.initArtListFn()
  }
}
</script>

<style lang="less" scoped>
// scoped原理: webpack打包的时候，会给组件标签上添加相同data-v-hash值，
// 然后也会给所有选择器后面加上一个[data-v-hash]值的属性选择器
//<标签data-v-390246 class="my_a"></标签>
//选择器会变成.my_a[data-v-390246]

// 设置富文本编辑器的默认最小高度
// ::v-deep作用: 穿透选择, 正常style上加了scope的话, 会给.ql-editor[data-v-hash]属性, 只能选择当前页面标签或者组件的根标签
// 如果想要选择组件内的标签(那些标签没有data-v-hash值)所以正常选择选不中, 加了::v-deep空格前置的话, 选择器就会变成如下形式
// [data-v-hash] .ql-editor 这样就能选中组件内的标签的class类名了
//重要注意事项: scoped只会给当前组件所有原生标签添加data-v-hash值属性，还会给组件标签内根标签添加data-v-hash值性,组件内的标签不会添加
// 里面的没被添加v-data-hash就不会起作用

::v-deep .ql-editor {
  //最小高度:标签本身的高度靠内容撑开，但是无内容没有300高度，标签会设置最小高度为300pX
  //如果内容大于300px,,标签高度也会随着撑开(比300px大) 4rem,4rem
  //直接给height:那么无论容器内的内容有多少，超出300高度的内容会溢出到外面而不是撑开此容器

  min-height: 300px;
}

// 设置图片封面的宽高
.cover-img {
  width: 400px;
  height: 280px;
  object-fit: cover;
}

.el-pagination {
  margin-top: 15px;
}

.title {
  font-size: 24px;
  text-align: center;
  font-weight: normal;
  color: #000;
  margin: 0 0 10px 0;
}

.info {
  font-size: 12px;
  span {
    margin-right: 20px;
  }
}

// 修改 dialog 内部元素的样式，需要添加样式穿透
::v-deep .detail-box {
  img {
    width: 500px;
  }
}
</style>
