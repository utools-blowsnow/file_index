<template>
  <div id="app">
    <div class="tools">
      <el-button type="primary" size="mini" icon="el-icon-plus" @click="openSelectDialog">新增</el-button>
      <el-button type="primary" size="mini" @click="reScanDirs">重建索引</el-button>
      <div style="float: right;">
        支持的文件后缀(正则)：<el-input size="mini" v-model="filters" style="width: 200px;" @change="saveFilters"></el-input>
      </div>
    </div>
    <el-table
        :data="items"
        border
        style="width: 100%">
      <el-table-column
          prop="name"
          label="名称"
          min-width="120">
      </el-table-column>
      <el-table-column
          prop="path"
          label="路径"
          min-width="300">
      </el-table-column>
      <el-table-column
          fixed="right"
          label="操作"
          min-width="120">
        <template slot-scope="scope">
          <el-button type="text" size="small" @click="seeFileIndexes(scope.row)">查看索引({{scope.row.list.length}})</el-button>
          <el-button type="text" size="small" @click="reScanDir(scope.row)">重建索引</el-button>
          <el-button type="text" size="small" @click="delItem(scope.$index)" style="color: red;">删除</el-button>
        </template>
      </el-table-column>
    </el-table>


    <el-dialog title="索引列表" :visible.sync="dialogTableVisible" width="70%">
      <el-table :data="activeItem.list">
        <el-table-column property="name" label="名称" min-width="100">
          <template slot-scope="scope">
            <el-input v-model="scope.row.name" size="mini" @change="changeFileNameAlias($event,scope.row.path)"></el-input>
          </template>
        </el-table-column>
        <el-table-column property="path" label="路径" min-width="200"></el-table-column>
      </el-table>
    </el-dialog>
  </div>
</template>

<script>

export default {
  name: 'app',
  data(){
    return {
      items: [],
      filters: '\.exe|\.lnk',

      dialogTableVisible: false,
      activeItem: {}
    }
  },
  created(){
    utools.onPluginReady(()=>{
      this.init();
    })

    utools.onPluginEnter(({code, type, payload, optional}) => {
      console.log('用户进入插件', code, type, payload,optional)
      console.log(utils.file.existsSync(code));
      if (type === "text"){
        if (window.utils.file.existsSync(code)){
          utools.shellOpenPath(code)
          utools.outPlugin()
        }
      }else if (type === "files" && payload.length > 0){
        this.addItem(payload[0].path);
      }
    })
  },
  methods:{
    /**
     * 初始化数据
     */
    async init(){
      if (window.db('items')){ //文件夹列表
        this.items = window.db('items');
      }
      if (window.db('filters')){ //过滤列表
        this.filters = window.db('filters');
      }
      //重建目录文件索引
      await this.reScanDirs();
    },
    /**
     * 打开选择弹窗
     */
    openSelectDialog(){
      let list = utools.showOpenDialog({
        properties: ['openDirectory','multiSelections']
      })
      if (list !== undefined){
        for(let dir of list){
          this.addItem(dir);
        }
      }
    },

    /**
     * 查看文件索引
     */
    seeFileIndexes(item){
      this.dialogTableVisible = true;
      this.activeItem = item;
    },


    /**
     * 扫描目录
     */
    scanDir(dir){
      var list = utils.scanDirFiles(dir,['node_modules','.git']);

      var nlist = [];
      for(let item of list){
        let type = this.getFileAllName(item);
        let regexp = new RegExp(this.filters,"i")
        if (!regexp.test(type)) continue;
        nlist.push(item);
      }

      return nlist;
    },
    /**
     * 添加项目
     */
    addItem(dir){
      let flag = false;
      for(let item of this.items){
          if (item.path === dir) {
            flag = true;
            break;
          }
      }
      if (flag) {
        return;
      }
      console.log('addItem',dir);
      let item = {
        name: this.getDirName(dir),
        path: dir,
        list: []
      };
      this.items.push(item)

      this.reScanDir(item);

      this.saveItems();
    },
    /**
     * 删除项目
     */
    delItem(index){
      let item = this.items[index];
      this.rmItemFileFeatures(item.path);

      this.items.splice(index,1);

      this.saveItems();
    },
    /**
     * 保存项目列表
     */
    saveItems(){
      let items = JSON.parse(JSON.stringify(this.items));
      for(let item of items){
        item.list = [];
      }
      console.log(items);
      window.db('items',items);
    },
    /**
     * 保存过滤
     */
    saveFilters(){
      window.db('filters',this.filters);
    },
    /**
     * 重建索引
     */
    reScanDir(item){
      const loading = this.$loading({
        lock: true,
        text: '正在扫描目录索引：' + item.path,
        spinner: 'el-icon-loading',
        background: 'rgba(0, 0, 0, 0.7)'
      });

      console.log('reScanDir',item);

      return new Promise(resolve => {
        setTimeout(()=>{

          let list = this.scanDir(item.path);

          this.setItemFileFeatures(item.path,list);

          let nlist = [];
          for(let path of list){
            nlist.push({
              name: this.getFileNameAlias(path),
              path: path,
            })
          }
          item.list = nlist;


          loading.close();


          resolve();
        },100)
      })

    },
    /**
     * 重建索引
     */
    async reScanDirs(){
      for(let item of this.items){
          await this.reScanDir(item);
      }
    },

    addItemFileFeature(dir,path){
      utools.setFeature({
        "code": path,
        "explain": path,
        "icon": utools.getFileIcon(path),
        "cmds": [this.getFileNameAlias(path),dir]
      })
    },
    setItemFileFeatures(dir,paths){
      //移除之前的
      this.rmItemFileFeatures(dir);

      for(let path of paths){
        this.addItemFileFeature(dir,path);
      }
    },
    rmItemFileFeatures(dir){
      const features = utools.getFeatures()
      for(let feature of features){
        if (feature.cmds.length === 2 && feature.cmds[1] === dir){
          utools.removeFeature(feature.code);
        }
      }
    },



    getFileType(file){
      var filename=file;
      var index1=filename.lastIndexOf(".") + 1;
      var index2=filename.length;
      var type=filename.substring(index1,index2);
      return type;
    },
    getFileName(file){
      var filename=file;
      var index1 = filename.lastIndexOf("/") + 1 || filename.lastIndexOf("\\") + 1;
      var index2=filename.lastIndexOf(".");
      var type=filename.substring(index1,index2);
      return type;
    },
    getFileAllName(file){
      var filename=file;
      var index1 = filename.lastIndexOf("/") + 1 || filename.lastIndexOf("\\") + 1;
      var index2=filename.length;
      var type=filename.substring(index1,index2);
      return type;
    },
    getDirName(file){
      var filename=file;
      var index1 = filename.lastIndexOf("/") + 1 || filename.lastIndexOf("\\") + 1;
      var index2=filename.length;
      var type=filename.substring(index1,index2);
      return type;
    },


    changeFileNameAlias(val,file){
      this.setFileNameAlias(file,val);
      this.addItemFileFeature(this.activeItem.path,file)
    },
    setFileNameAlias(file,name){
      window.db("file_alias_" + file,name);
    },

    getFileNameAlias(file){
      if (window.db("file_alias_" + file)){
        return window.db("file_alias_" + file);
      }
      return this.getFileName(file);
    }
  }
}
</script>

<style>
.tools{
  padding: 10px 0;
}
</style>
