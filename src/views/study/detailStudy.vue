<template>
  <div class="app-container">
    <el-form ref="form" :model="form" :rules="createRules" label-width="120px">
      <el-form-item label="标题" prop="title">
        <el-input v-model="form.title" :disabled="!isEdit"></el-input>
      </el-form-item>
      <el-form-item label="作者" prop="author">
        <el-input v-model="form.author" :disabled=true></el-input>
      </el-form-item>
      <el-form-item label="审核状态:" prop="reviewStatus">
        <el-switch v-model="!form.reviewStatus" active-text="完成审核" inactive-text="正在审核" :disabled=true></el-switch>
      </el-form-item>
      <el-form-item label="学习资料详情" prop="des">
        <!-- <el-input type="textarea" v-model="form.des"></el-input> -->
        <div id="editor"></div>
      </el-form-item>
      <el-form-item v-if="isEdit" >
        <el-button @click="openStudyDesPage">学习资料详情页面</el-button>
        <el-button type="primary" @click="onSubmit">编辑</el-button>
        <el-button @click="onCancel">取消</el-button>
      </el-form-item>
      <el-form-item v-else >
        <el-button @click="onCancel">返回</el-button>
      </el-form-item>
    </el-form>
  </div>
</template>

<script>
import { isValidTitle} from '@/utils/validate'
import { getStudy, editStudy} from '@/api/study'
import WangEditor from 'wangeditor'
// import '@/utils/custom-menu'

export default {
  data() {
    const validateTitle = (rule, value, callback) => {
      console.log('-----------------123')
      if (!isValidTitle(value)) {
        callback(new Error('请输入正确的标题'))
      } else {
        callback()
      }
    }
    const validateDes = (rule, value, callback) => {
      if (!value) {
        callback(new Error('请输入正确的详情'))
      } else {
        callback()
      }
    }

    return {
      isEdit: this.$route.path.split('/')[2] === 'edit' ? true : false,
      editor:null,
      form: {
        ID: this.$route.path.split('/')[3],
        title: '',
        author: '',
        isPublic: '',
        reviewStatus: '',
        isGood: '',
        des: '',
      },
      createRules: {
        title: [{ required: true, trigger: 'blur', validator: validateTitle }],
        des: [{required: true, trigger: 'blur', validator: validateDes }],
      }
    }
  },

  mounted() {
    console.log('mounted')
    console.log(this.isEdit)
    this.getStudyData(this.form.ID)
    this.initEditor()
  },

  methods: {
    initEditor(){
      console.log('initEditor')
      this.editor = new WangEditor('#editor')  //这个地方传入div元素的id 需要加#号
      this.editor.change = function () { // 这里是change 不是官方文档中的 onchange
        // 编辑区域内容变化时，实时打印出当前内容
        console.log(this.txt.html())
      }
      this.editorConfigUploadImg(this.editor)
      this.editor.create()     // 生成编辑器
      console.log(this.isEdit)
      if(!this.isEdit) {
        console.log('---------------------------------')
        this.editor.$textElem.attr('contenteditable', false)
      }
      // this.editor.txt.html('<p>输入内容...</p>')   //注意：这个地方是txt  不是官方文档中的$txt
    },

    editorConfigUploadImg(editor) {
      editor.customConfig.debug = true
      console.log(editor)
      editor.customConfig.showLinkImg = false
      editor.customConfig.uploadImgServer = 'http://picture-system-live.ejudata.com/qiniu/image/single'  // 上传图片到服务器
      // // 图片上传限制
      // editor.customConfig.uploadImgMaxSize = 5 * 1024 * 1024
      // editor.customConfig.uploadImgMaxLength = 1
      // header和 body 配置
      editor.customConfig.uploadFileName = 'file'
      editor.customConfig.uploadImgParams = {
        bizType:2
      },
      editor.customConfig.uploadImgHeaders = {
          ts: 1524539752494,
          token: 'd4c349f09cc2dd8d5b2c2da3347c1497',
          platform: 'WEB'
      }
      editor.customConfig.uploadImgTimeout = 8000
      editor.customConfig.uploadImgHooks = {
        before: function (xhr, editor, files) {
            console.log('before')
            console.log(files.length)
            if(files.length > 1) {
              alert('一次最多上传1张图片')
              return {prevent: true,msg: '一次最多上传1张图片'}
            }
            const file = files[0]
            const isJPGOoPNG = ['image/jpeg', 'image/png'].indexOf(file.type) !== -1
            const isLt = file.size / 1024 / 1024 < 5
            if (!isJPGOoPNG) {
              alert('上传图片只能是jpg/png格式!')
              return {prevent: true,msg: '上传图片只能是jpg/png格式'}
            }
            if (!isLt) {
              alert('上传头像图片大小不能超过 5MB!')
              return {prevent: true,msg: '上传头像图片大小不能超过 5MB!'}
            } 
            return {prevent:false}
      },
      timeout: function (xhr, editor) {
         alert('请求超时')
      },
      customInsert: function (insertImg, result, editor) {
        // 图片上传并返回结果，自定义插入图片的事件（而不是编辑器自动插入图片！！！）
        // insertImg 是插入图片的函数，editor 是编辑器对象，result 是服务器端返回的结果
        console.log(result)
        // 举例：假如上传图片成功后，服务器端返回的是 {url:'....'} 这种格式，即可这样插入图片：
        const url = result.data.url
        insertImg(url)
      }
    }
},
    getStudyData(ID) {
      getStudy(ID).then(response => {
        console.log('get study success')
        console.log(response)
        response.data.reviewStatus = response.data.review.status === 0
        this.form = response.data
        this.editor.txt.html(this.form.des) 
      })
    },

    editStudyData(study) {
      editStudy(study).then(response => {
        console.log('edit success')
        this.$router.push({ path: '/study/index' })
      })
    },

    onSubmit() {
      this.form.des = this.editor.txt.html()
      this.$refs.form.validate(valid => {
        if (valid) {
          console.log('--------')
          console.log(this.form)
          this.editStudyData(this.form)
        } else {
          console.log('error submit!!')
          return false
        }
      })
    },

    onCancel() {
      this.$router.push({ path: '/study/index' })
    },

    openStudyDesPage() {
      window.open(window.location.origin + '#/study/des')
    }
  }
}
</script>

<style scoped>
.line{
  text-align: center;
}
.upload_img{
  width: 0.2rem;
  height: 0.2rem;
}
</style>

