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
      <el-col :span="12">
        <el-button v-text="showPath"></el-button>
      </el-col>
      <el-col :span="12" style="display: flex; justify-content: flex-end;">
        <el-button @click="clickDeleteSelectionFile">删除选中文件</el-button>
        <el-button @click="clickCreateDir">创建文件夹</el-button>
        <el-button @click="uploadFile">上传文件</el-button>
        <el-button @click="clickBack" :disabled="isRoot">返回</el-button>
      </el-col>
    </el-row>
    <!--path下文件信息表格-->
    <el-row>
      <template v-if="tableData">
        <el-table :data="tableData"
                  style="width: 100%"
                  @selection-change="handleSelectionChange"
                  border>
          <el-table-column type="selection" width="50"></el-table-column>
          <el-table-column prop="basename" label="文件名"></el-table-column>
          <el-table-column prop="type" label="类型" sortable></el-table-column>
          <el-table-column prop="perms" label="权限" sortable></el-table-column>
          <el-table-column prop="fileSize" label="文件大小" sortable></el-table-column>
          <el-table-column prop="date" label="日期" sortable></el-table-column>
          <el-table-column inline-template>
            <div>
              <el-button size="small" @click="clickFile(row)"  :disabled="row.type != 'dir'">进入</el-button>
              <el-button size="small" @click="clickDeleteFile(row)" type="danger">删除</el-button>
            </div>
          </el-table-column>
        </el-table>
      </template>
    </el-row>
  </div>

</template>
<script>
  let _ = require('lodash')
  let Cookies = require('js-cookie')
  let hprose = require('../../node_modules/hprose-html5/dist/hprose-html5')
  const COOKIE_USE_SERVER = 'useServer'                     // cookie: 服务器地址
  const API_GET_FILE_INFO = 'FileSystem_getFileSystemList'
  const API_REMOVE = 'FileSystem_remove'  // 删除文件或文件夹
  const API_CREATE_DIR = 'FileSystem_createDir'
//  const API_UPLOAD_FILE = 'File_uploadFile'

  export default {
    name: 'home',
    data () {
      return {
        client: {},         // 客户端对象
        serverUrl: '',        // 使用的服务器地址
        serverUrlInput: '',  // 输入框中的url地址
        nowPath: '',          // 当前所在目录
        dirname: '',          // 当前文件夹名
        rootPath: '',         // 根目录
        // 表格数据
        tableData: [],
        selectionChange: []   // 多选框勾选数据
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
        let dirname = rowData.dirname
        let basename = rowData.basename
        let type = rowData.type
        if (type !== 'dir') return false
        let path = dirname + '/' + basename
        this.changePath(path, type)
      },
//      点击返回上一层
      clickBack () {
        let path = this.nowPath
        let backPath = ''
        let lastIndex = path.lastIndexOf('/')
        if (lastIndex !== -1) {
          backPath = path.substr(0, lastIndex)
        }
        console.info('backPath', backPath)
        this.changePath(backPath)
      },
//      切换路径时执行操作
      changePath (path = '', type = 'dir') {
        if (type !== 'dir') return false
        console.info('to path: ', path)
        if (!path) path = this.nowPath
        let promise = this.client.invoke(API_GET_FILE_INFO, [{data: {path: path}}])
        promise.then((data) => {
          console.info(API_GET_FILE_INFO, data)
          if (data.ret.code === 0) {
            this.nowPath = data.result.info.dirname + '/' + data.result.info.basename
            this.dirname = data.result.info.basename
            this.tableData = data.result.child
          } else {
            console.error(API_GET_FILE_INFO, data.ret.msg)
            this.$message({
              type: 'error',
              message: data.ret.msg
            })
          }
        }).catch((error) => {
          console.error('接口' + API_GET_FILE_INFO + '返回错误: ', error)
        })
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
          this.$message({type: 'success', showClose: true, message: '连接服务器成功：' + useServer})
          // 开始设置数据
          this.client = client
          this.serverUrl = useServer
          this.nowPath = data.result.info.dirname + '/' + data.result.info.basename
          this.dirname = data.result.info.basename
          this.tableData = data.result.child
          this.rootPath = data.result.info.rootPath
          Cookies.set(COOKIE_USE_SERVER, useServer)  // 设置cookie
        }).catch((err) => {
          console.error(err)
          this.$message({type: 'error', showClose: true, message: '操作未生效, 服务器地址异常，无法加载api信息, 地址为：' + this.serverUrl})
        })
      },
      // 勾选文件时触发
      handleSelectionChange (val) {
        console.info('handleSelectionChange', val)
        this.selectionChange = val
      },
      // 点击单行的删除按钮
      clickDeleteFile (row) {
        let fileArr = []
        let path = row.dirname + '/' + row.basename
        fileArr.push(path)
        this.deleteFile(fileArr)
        console.info(fileArr)
      },
      // 点击删除选中文件按钮
      clickDeleteSelectionFile () {
        let selectionFileArr = this.selectionChange
        if (_.isEmpty(selectionFileArr)) return false
        let fileArr = []
        selectionFileArr.forEach((x, i) => {
          console.log(i, x)
          fileArr.push(x.dirname + '/' + x.basename)
        })
        this.deleteFile(fileArr)
        console.info(fileArr)
      },
      // 调用删除按钮
      deleteFile (fileArr) {
        let promise = this.client.invoke(API_REMOVE, [{data: {paths: fileArr}}])
        promise.then((data) => {
          console.info(API_REMOVE, data)
          if (data.ret.code === 0) {
            this.$message({
              type: 'success',
              message: '删除成功'
            })
          } else {
            this.$message({
              type: 'error',
              message: '删除失败：' + data.ret.msg
            })
          }
          this.changePath()
        }).catch((error) => {
          console.error('接口' + API_REMOVE + '返回错误: ', error)
        })
      },
      // 点击创建文件夹按钮
      clickCreateDir () {
        console.info(API_CREATE_DIR)
        this.$prompt('输入文件夹名称', '新建文件夹', {
          confirmButtonTest: '确定',
          cancelButtonText: '取消'
        }).then(({value}) => {
          this.createDir(value)
        })
      },
      // 调用创建文件夹接口
      createDir (dirname) {
        console.info('dirname', dirname)
        let promise = this.client.invoke(API_CREATE_DIR, [{data: {dirname: dirname, path: this.nowPath}}])
        promise.then((data) => {
          console.info(API_CREATE_DIR, data)
          if (data.ret.code === 0) {
            this.$message({
              type: 'success',
              message: '创建成功'
            })
          } else {
            this.$message({
              type: 'error',
              message: '创建失败，错误信息: ' + data.ret.msg
            })
          }
        }).catch((error) => {
          console.error(API_CREATE_DIR, error)
          this.$message({
            type: 'error',
            message: '创建失败'
          })
        })
        this.changePath(this.nowPath)
      },
      uploadFile () {
        // todo 上传文件到服务器操作
        this.$message({
          type: 'error',
          message: '功能未实现'
        })
      }
    },
    computed: {
      // 显示的目录路径
      showPath: function () {
        let rootPath = this.rootPath
        let nowPath = this.nowPath
        return nowPath.replace(rootPath, 'root')
      },
      isRoot: function () {
        return this.rootPath === this.nowPath
      }
    },
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
