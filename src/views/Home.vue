<template>
  <div class="home">
    <div class="layout">
    <!--头部 ------------------------------------------------------------------------------------------------------- -->
    <div class="header">
      <router-link to="/">
        <img v-if="info.logo" :src="info.logo" alt="AList" style="height:56px;width:auto;" id="logo">
        <a-spin v-else />
      </router-link>
      <a-space>
        <a-popover title="二维码" class="qrcode">
          <template slot="content">
            <img :src="'https://api.xhofe.top/qr?size=200&text='+info.site_url+'/'+file_id"/>
          </template>
          <a-button type="primary" shape="circle" icon="qrcode" size="large" />
        </a-popover>
        <a-space v-if="show.preview&&(!preview_show.other)">
        <a-button type="primary" shape="circle" icon="copy" size="large" @click="copyFileLink" />
        <a target="_blank" :href="file.download_url"><a-button type="primary" shape="circle" icon="download" size="large" /></a>
        </a-space>
      </a-space>
    </div>
    <a-divider class="header-content" />
    <!--主体内容 ------------------------------------------------------------------------------------------------------- -->
    <div class="content">
      <div class="tool">
        <!--路径 ------------------------------------------------------------------------------------------------------- -->
        <div class="routes" v-show="show.routes">
          <a-icon type="home" id="home-icon"/>
          <a-breadcrumb :routes="routes">
            <template slot="itemRender" slot-scope="{ route, params, routes, paths }">
              <span v-if="routes.indexOf(route) === routes.length - 1">
                {{ route.breadcrumbName }}
              </span>
              <router-link v-else :to="`/${route.path}`">
                {{ route.breadcrumbName }}
              </router-link>
            </template>
          </a-breadcrumb>
        </div>
      </div>
      <a-divider class="header-content" />
      <!--文件列表 ------------------------------------------------------------------------------------------------------- -->
      <div class="files" v-show="show.files">
        <a-table 
          :columns="columns" :data-source="files" :pagination="false"
          rowKey="file_id" :customRow="customRow"
          :loading="files_loading"
          :scroll="{x:'max-content'}"
          >
          <template slot="name" slot-scope="text,record">
            <a-icon :type="`${record.icon}`" theme="filled" class="file-icon" />{{text}}
          </template>
        </a-table>
      </div>
      <br/>
      <!--Readme ------------------------------------------------------------------------------------------------------- -->
      <div class="readme" v-show="show.readme">
        <a-card title="Readme.md" style="width: 100%" size="small">
          <!-- <a slot="extra" href="#">more</a> -->
          <MarkdownPreview :initialValue="readme" />
        </a-card>
      </div>
      <!--预览 ------------------------------------------------------------------------------------------------------- -->
      <div class="preview" v-show="show.preview">
        <a-result :title="file.name" v-if="preview_show.other">
          <template #icon>
            <a-icon :type="file.icon" theme="filled" />
          </template>
          <template #extra>
            <a target="_blank" :href="file.download_url">
              <a-button type="primary">下载</a-button>
            </a>
            <a-button type="primary" @click="copyFileLink">复制直链</a-button>
            <!-- <a-popover title="二维码" class="qrcode">
              <template slot="content">
                <img :src="'https://api.xhofe.top/qr?size=200&text='+info.site_url+'/'+file_id"/>
              </template>
              <a-button type="primary">二维码</a-button>
            </a-popover> -->
          </template>
        </a-result>
        <a-spin :spinning="iframe_spinning" v-if="preview_show.doc">
          <iframe :src="url" class="doc-preview" frameborder="no" @load="iframe_spinning=false"></iframe>
        </a-spin>
        <div class="img-preview" v-if="preview_show.image"><img :src="url"/></div>
        <div class="video-preview" v-if="preview_show.video">
          <d-player id="d-player" screenshot=true autoplay=true :options="video_options"></d-player>
        </div>
        <div class="audio-preview" v-if="preview_show.audio">
          <aplayer autoplay :music="audio_options" />
        </div>
        <div class="text-preview" v-if="preview_show.text">
          <MarkdownPreview :initialValue="text_content" />
        </div>
      </div>
    </div>
    <a-divider id="footer-line"/>
    <div class="footer">
      Powered By <a target="_blank" href="https://github.com/Xhofe/alist">AList</a>
      <a-divider v-if="info.footer_text" type="vertical" />
      <a v-if="info.footer_text" target="_blank" :href="info.footer_url">{{info.footer_text}}</a>
    </div>
    <a-modal v-model="show.password" title="Input password" @ok="handleOkPassword" @cancel="cancelPassword">
      <a-input-password placeholder="input password" v-model="password" @pressEnter="handleOkPassword"/>
    </a-modal>
    </div>
  </div>
</template>

<script>

import {list,get,search,info,getWebLatest,getBackLatest,getText} from '../utils/api'
import {copyToClip} from '../utils/copy_clip'
import {formatDate} from '../utils/date'
import {getFileSize} from '../utils/file_size'
import { MarkdownPreview } from 'vue-meditor'
import VueDPlayer from 'vue-dplayer'
import 'vue-dplayer/dist/vue-dplayer.css'
import Aplayer from 'vue-aplayer'
import {Base64} from '../utils/base64'

export default {
  name: 'Home',
  components:{
    MarkdownPreview,Aplayer,
    'd-player': VueDPlayer,
  },
  watch:{
    '$route'(to,from){
      this.init()
    }
  },
  data(){
    return{
      version:'v0.1.4',
      //表格列
      columns:[{align:'left',dataIndex:'name',title:'文件',scopedSlots:{customRender:'name'},
                sorter:(a,b)=>{
                  return a.name<b.name?1:-1
                }},
               {align:'right',dataIndex:'sizeStr',title:'大小',width:100,
                sorter:(a,b)=>{
                  return a.size-b.size
                }
               },
               {align:'right',dataIndex:'time',title:'时间',width:170,
                sorter:(a,b)=>{
                  return a.time<b.time?1:-1
                }}],
      files:[],//当前文件夹下文件
      files_loading:true,
      //是否展示的组件
      show:{
        search:false,
        routes:true,
        files:true,
        preview:false,
        readme:false,
        password:false,
      },
      //预览组件展示
      preview_show:{
        image:false,
        video:false,
        audio:false,
        doc:false,
        other:false,
        text:false,
      },
      //当前文件
      file:{},
      //当前文件预览url
      url:'',
      //预览视频选项
      video_options:{},
      //预览音频选项
      audio_options:{},
      //当前请求file_id
      file_id:undefined,
      //请求的密码
      password:undefined,
      //当前路径
      routes:[],
      //readme内容
      readme: '',
      // 文件与图标对应关系
      file_extensions:{
        exe:'windows',
        xls:'file-excel',
        xlsx:'file-excel',
        md:'file-markdown',
        pdf:'file-pdf',
        ppt:'file-ppt',
        pptx:'file-ppt',
        doc:'file-word',
        docx:'file-word',
        // jpg:'file-jpg',
        zip:'file-zip',
        gz:'file-zip',
        rar:'file-zip',
        "7z":'file-zip',
        tar:'file-zip',
        jar:'file-zip',
        xz:'file-zip',
        apk:'android',
      },
      categorys:{
        image:'file-image',
        doc:'file-text',
        video:'youtube',
        audio:'customer-service',
      },
      //自定义内容
      info:{
        
      },
      text_content:'',//文本内容
      iframe_spinning:true,
    }
  },
  methods:{
    async checkBackUpdate(){
      getBackLatest().then(res=>{
        if(res.data.tag_name!=this.version){
          this.$notify.open({
            message: '发现新版本',
            description:
              '后端新版本:'+res.data.tag_name+', 请至'+res.data.html_url+'获取新版本',
            icon: <a-icon type="smile" style="color: #1890ff" />,
          });
        }else{
          //已经是最新版本
        }
      }).catch(err=>{
        console.log("failed check update",error);
      })
    },
    async checkWebUpdate(){
      getWebLatest().then(res=>{
        if(res.data.tag_name!=this.version){
          this.$notify.open({
            message: '发现新版本',
            description:
              '前端新版本:'+res.data.tag_name+', 请至'+res.data.html_url+'获取新版本',
            icon: <a-icon type="smile" style="color: #1890ff" />,
          });
        }else{
          //已经是最新版本
        }
      }).catch(err=>{
        console.log("failed check update",error);
      })
    },
    initInfo(){
      info().then(res=>{
        if (res.meta.code==200) {
          this.info=res.data
          if(res.data.check_update){
            this.checkBackUpdate()
            this.checkWebUpdate()
          }
          if (res.data.title && res.data.title!="") {
            document.title=res.data.title
          }
          if(!this.info.logo){
            this.info.logo=require('../assets/alist.png')
          }
          if(this.info.script){
            this.loadJS(this.info.script)
          }
        }else{
          this.$msg.error(res.meta.msg)
        }
      })
    },
    init(){
      this.show.routes=true
      this.show.files=true
      this.show.search=false
      this.show.preview=false
      this.show.readme=false
      this.preview_show={
        image:false,
        video:false,
        audio:false,
        doc:false,
        other:false,
        text:false,
      }
      this.file_id=this.$route.params.id
      if(this.file_id==undefined){
        this.file_id="root"
      }
      this.refreshFileId()
    },
    refreshFileId(){
      //首先获取列表看是不是
      this.files_loading=true
      list(this.file_id,this.password).then(res=>{
        if (res.meta.code==200) {
          //获取成功 是文件夹
          this.showRoutes(res.data.paths)
          this.showFiles(res.data.items)
          //展示Readme?
          this.showReadme(res.data.readme)
        }else if(res.meta.code==401){
          //需要密码
          this.$msg.error(res.meta.msg)
          this.show.password=true
        }else{
          // 不是文件夹，尝试获取文件
          this.show.files=false
          get(this.file_id).then(res=>{
            if (res.meta.code==200) {
              //获取成功 是文件
              this.showRoutes(res.data.paths)
              this.showFile(res.data)
            }else{
              // 也不是文件
              this.$msg.warning(res.meta.msg)
              this.show.routes=false
            }
          })
        }
      })
    },
    showRoutes(paths){
      this.routes=paths.reverse().map(item=>{
        item.path=item.file_id
        item.breadcrumbName=item.name
        return item
      })
    },
    showFiles(files){
      this.files_loading=false
      this.files=files.map(item=>{
        item.time=formatDate(item.updated_at)
        if(item.type=='folder'){
          item.sizeStr='-'
        }else{
          item.sizeStr=getFileSize(item.size)
        }
        item.icon=this.getIcon(item)
        return item
      })
    },
    getIcon(record){
      if (record.type=='folder'){
        return 'folder'
      }
      if (this.file_extensions.hasOwnProperty(record.file_extension)){
        return this.file_extensions[record.file_extension]
      }
      if (this.categorys.hasOwnProperty(record.category)){
        return this.categorys[record.category]
      }
      return 'file'
    },
    showReadme(readme){
      this.readme=readme
      if (readme!=""){
        this.show.readme=true
      }else{
        this.show.readme=false
      }
    },
    showFile(file){
      this.file=file
      this.file.icon=this.getIcon(file)
      this.url=file.download_url
      this.show.preview=true
      // if (file.category=='doc') {
      //   // TODO 无法预览,为啥啊
      //   this.preview_show.doc=true
      //   this.url='https://view.officeapps.live.com/op/view.aspx?src='+encodeURIComponent(file.download_url)
      // }
      if (file.category=='image') {
        // 预览图片
        this.preview_show.image=true
        return
      }
      if (file.category=='video') {
        // 预览视频
        this.video_options={
          video:{
            url:this.url
          },
          autoplay:this.info.autoplay?true:false,
        }
        this.preview_show.video=true
        return
      }
      if (file.category=='audio'){
        this.audio_options={
          title: file.name,
          artist: '',
          src: this.url,
          pic: this.info.music_img?this.info.music_img:'https://img.oez.cc/2020/12/07/f6e43dc79d74a.png'
        }
        this.preview_show.audio=true
        return
      }
      if (this.info.preview.text.includes(file.file_extension.toLowerCase())){
        this.showText(file)
        return
      }
      this.showOther(file)
      // this.preview_show.other=true
    },
    showText(file){
      this.text_content=''
      this.preview_show.text=true
      getText(file.url).then(res=>{
        if (file.file_extension.toLowerCase()=='md') {
          this.text_content=res.data
        }else{
          this.text_content='```'+file.file_extension+'\n'+res.data+'\n```'
        }
      })
    },
    showOther(file){
      if (this.info.preview.extensions.includes(file.file_extension)) {
        if (file.size<=this.info.preview.max_size) {
          let direct_url=this.info.backend_url+"/d/"+this.file_id+'/'+file.name
          for(const v of this.info.preview.pre_process){
            switch (v){
              case 'base64':
                direct_url=Base64.encode(direct_url)
                break
              case 'encodeURIComponent':
                direct_url=encodeURIComponent(direct_url)
                break
              case 'encodeURI':
                direct_url=encodeURI(direct_url)
                break
              default:
                this.$msg.warning('配置文件中不支持的encode.')
                this.preview_show.other=true
                return
            }
          }
          this.iframe_spinning=true
          this.url=this.info.preview.url+direct_url
          this.preview_show.doc=true
          return
        }else{
          this.$msg.warning("文件过大,请下载查看.")
        }
      }
      this.preview_show.other=true
    },
    customRow(record, index){
      return{
        on:{
          click:event=>{
            this.$router.push(record.file_id)
          }
        }
      }
    },
    handleOkPassword(){
      this.show.password=false
      this.refreshFileId()
    },
    cancelPassword(){
      this.$router.go(-1)
    },
    copyFileLink(){
      let content=this.info.backend_url+"/d/"+this.file_id
      copyToClip(content)
      this.$msg.success('链接已复制到剪贴板.');
    },
    loadJS(content){
      return new Promise((resolve,reject)=>{
        let script = document.createElement('script')
        script.type = "text/javascript"
        script.onload = ()=>{
          resolve()
        }
        script.onerror = ()=>{
          reject()
        }
        if(/^(http|https):\/\/([\w.]+\/?)\S*/.test(content)){
          script.src=content
        }else{
          script.text= content
        }
        document.querySelector('body').appendChild(script)
      })
    }
  },
  mounted(){
    console.log("\n %c Alist %c https://github.com/Xhofe/alist \n\n","color: #fadfa3; background: #030307; padding:5px 0;","background: #fadfa3; padding:5px 0;")
    this.initInfo()
    this.init()
  }
}
</script>

<style scoped>
.home{
  width: 100%;
  display: flex;
  display: -webkit-flex; /* Safari */
  justify-content: center;
  padding: 0;
  margin: 0;
}
.layout{
  display: flex;
  display: -webkit-flex; /* Safari */
  flex-direction: column;
  justify-content: center;
  align-items: center;
  width: min(1200px,98vw);
}
.header{
  padding-top: 3px;
  height: 56px;
  width: 100%;
  display: flex;
  display: -webkit-flex; /* Safari */
  justify-content: space-between;
  align-items: center;
}
.header-content{
  margin: 10px 0 5px 0;
}
.content{
  min-height: 80vh;
  width: 100%;
  display: flex;
  display: -webkit-flex; 
  flex-direction: column;
  justify-content: flex-start;
}
.routes{
  /* box-shadow: 0 2px 4px rgba(0, 0, 0, .12), 0 0 6px rgba(0, 0, 0, .04); */
  margin: 2px 2px;
  padding: 2px;
  display: flex;
  display: -webkit-flex; 
  font-size: 20px;
}
#home-icon{
  margin-right: 10px;
}
/* .files{
  
} */
.file-icon{
  margin-right: 10px;
  font-size: 20px;
  color: #1890ff;
}
/* .preview{

} */

/* .video-preview{

} */

#d-player{
  /* height: 80vh; */
  width: 100%;
}

@media screen and (min-width: 600px) {
  #d-player{
    height: 80vh;
  }
}

.doc-preview{
  width: 100%;
  height: 80vh;
  box-sizing: inherit;
}

.img-preview{
  width: 100%;
  height: 80vh;
}

.img-preview img  {
  max-width: 100%;
  max-height: 100%;
  display: block;
  margin: auto;
}

.footer{
  width: 100%;
  text-align: center;
  height: 40px;
}
#footer-line{
  margin: 10px 0;
}

@media screen and (max-width: 600px) {
    .qrcode {
        display: none;
    }
}

</style>
