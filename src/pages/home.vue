<template>
  <div>
    <!--服务器地址-->
    <el-row>
      <el-col :span="24" style="display: flex;">
        <el-input paceholder="服务器地址" v-model="serverUrlInput"></el-input>
        <el-button @click="clickServerUrlSubmit">确认</el-button>
        <el-tag hit v-text="serverUrl"></el-tag>
      </el-col>
    </el-row>
    <!--当前path-->
    <el-row>
      <el-col><div v-text="nowPath"></div></el-col>
    </el-row>
    <!--path下文件信息表格-->
    <el-row>
      <template v-if="tableData">
        <el-table :data="tableData" style="width: 100%" @row-click="clickFile">
          <el-table-column prop="info.basename" label="文件名"></el-table-column>
          <el-table-column prop="info.type" label="类型"></el-table-column>
          <el-table-column prop="info.perms" label="权限" width="perms"></el-table-column>
          <el-table-column prop="info.fileSize" label="文件大小" width="180"></el-table-column>
          <el-table-column prop="info.date" label="日期" width="date"></el-table-column>
        </el-table>
      </template>
    </el-row>
  </div>

</template>

<script>
//  let _ = require('lodash')
  let Cookies = require('js-cookie')
  let hprose = require('../../node_modules/hprose-html5/dist/hprose-html5')
  const COOKIE_USE_SERVER = 'useServer'                     // cookie: 服务器地址
  const API_GET_FILE_INFO = 'FileSystem_getFileSystemList'
//  const API_CREATE_DIR = 'File_createDir'
//  const API_GET_RES_URL = 'File_getResUrl'
//  const API_REMOVE_DIR = 'File_removeDir'
//  const API_REMOVE_FILE = 'File_removeFile'
//  const API_UPLOAD_FILE = 'File_uploadFile'
  export default {
    name: 'home',
    data () {
      return {
        client: {},         // 客户端对象
        serverUrl: '',        // 使用的服务器地址
        serverUrlInput: '',  // 输入框中的url地址
        nowPath: '',          // 当前所在目录
        // 表格数据
        tableData: []
      }
    },
    mounted () {
      let useServer = Cookies.get(COOKIE_USE_SERVER)
      if (useServer) this.serverUrl = useServer
      this.initData()
    },
    methods: {
//      点击更换服务器地址按钮
      clickServerUrlSubmit () {
        this.serverUrl = this.serverUrlInput
        this.initData(this.serverUrl)
      },
      // 点击文件或文件夹
      clickFile (rowData) {
        let path = rowData.info.dirname
        let type = rowData.info.type
        if (type !== 'dir') return false
        console.info('to path: ', path)
        let promise = this.client.invoke(API_GET_FILE_INFO, [{data: {path: path}}])
        promise.then((data) => {
          console.info(data)
          this.nowPath = data.result.info.dirname
          this.tableData = data.result.child
        }).catch((error) => {
          console.error('接口' + API_GET_FILE_INFO + '返回错误: ', error)
        })
      },
//      点击返回上一层
      clickUp () {
      },
      // 处理传入接口的参数
      handleArgs () {
      },
      initData (server) {
        let useServer = ''
        if (server) {
          useServer = server
        } else {
          useServer = this.serverUrl
        }
        let client = hprose.Client.create(useServer)
        let promise = client.invoke(API_GET_FILE_INFO)
        promise.then((data) => {
          console.log('File_getFileInfo: ', data)
//          data = _.omit(data, ['ret', 'debug'])
          this.$message({type: 'success', showClose: true, message: '连接服务器成功：' + useServer})
          // 开始设置数据
          this.client = client
          this.serverUrl = useServer
          this.nowPath = data.result.info.dirname
          this.tableData = data.result.child
          Cookies.set(COOKIE_USE_SERVER, useServer)  // 设置cookie
        }).catch((err) => {
          console.error(err)
          this.$message({type: 'error', showClose: true, message: '操作未生效, 服务器地址异常，无法加载api信息, 地址为：' + this.serverUrl})
        })
      }
    },
    computed: {},
    filters: {},
    watch: {}
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  h1, h2 {
    font-weight: normal;
  }

  ul {
    list-style-type: none;
    padding: 0;
  }

  li {
    display: inline-block;
    margin: 0 10px;
  }

  a {
    color: #42b983;
  }
</style>
