<template>
  <div class="background">
    <div id="container">
    </div>
    <div class="box">
      <div class="left-list">
        <a-list
            class="demo-loadmore-list"
            :loading="loading"
            item-layout="horizontal"
            :data-source="listData"
        >
          <div
              v-if="showLoadingMore"
              slot="loadMore"
              :style="{ textAlign: 'center', marginTop: '12px', height: '32px', lineHeight: '32px' }"
          >
            <a-spin v-if="loadingMore"/>
            <a-button v-else @click="onLoadMore">
              loading more
            </a-button>
          </div>
          <a-list-item slot="renderItem" slot-scope="item, index" :key="index">
            <a @click="editArea(item)" slot="actions">编辑</a>
            <a @click="delArea(item)" slot="actions">删除</a>
            <a-list-item-meta>
              <a slot="title" href="#">{{ item.name }}</a>
            </a-list-item-meta>
          </a-list-item>
        </a-list>
      </div>
    </div>
    <div id="panel">
      <p>输入关键字，将展示相关地点提示，点击提示可定位到该处。</p>
      <input id='keyword' type="text" value='' @input="getSuggestions()"><input id="search" type="button" class="btn"
                                                                                value="搜索"
                                                                                onclick="searchByKeyword()"/>
      <ul id="idSuggestionList">
        <li v-for="(item, index) in suggestionList" :key="index"><a href="#"
                                                                    @click="setSuggestion(index)">{{ item.title }}<span
            class="item_info">{{ item.address }}</span></a></li>
      </ul>
    </div>
    <a-drawer
        title="Basic Drawer"
        placement="right"
        :width="720"
        :closable="false"
        :visible="visible"
        :get-container="false"
        :wrap-style="{ position: 'absolute' }"
        @close="onClose"
    >
      <a-form :form="form" class="form">
        <a-form-item
            label="经度"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
        >
          <a-input v-model="formData.lng" :placeholder="$t('titleInput')"/>
        </a-form-item>
        <a-form-item
            label="维度"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
        >
          <a-input v-model="formData.lat" :placeholder="$t('titleInput')"/>
        </a-form-item>
        <a-form-item
            label="名称"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
            :required="false"
        >
          <a-input v-model="formData.name" :placeholder="$t('customerInput')"/>
        </a-form-item>
        <a-form-item
            label="工程名称"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
            :required="false"
        >
          <a-input v-model="formData.projectName" :placeholder="$t('criticsInput')"/>
        </a-form-item>
        <a-form-item
            label="工程内容"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
            :required="false"
        >
          <a-textarea v-model="formData.projectContent" :placeholder="$t('criticsInput')"/>
        </a-form-item>
        <a-form-item
            label="责任单位"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
            :required="false"
        >
          <a-input v-model="formData.projectUnit" :placeholder="$t('criticsInput')"/>
        </a-form-item>
        <a-form-item
            label="施工年限"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
            :required="false"
        >
          <a-input v-model="formData.projectPeriod" :placeholder="$t('criticsInput')"/>
        </a-form-item>
        <a-form-item
            label="影响道路"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
            :required="false"
        >
          <a-input v-model="formData.road" :placeholder="$t('criticsInput')"/>
        </a-form-item>
        <a-form-item
            label="积水程度"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 10}"
            :required="false"
        >
          <a-input-number :min="0" :max="100"/>
          <span>%</span>
        </a-form-item>
        <a-form-item
            label="图片"
            :labelCol="{span: 3}"
            :wrapperCol="{span: 30}"
            :required="false"
        >
          <div class="clearfix">
            <a-upload
                action="https://www.mocky.io/v2/5cc8019d300000980a055e76"
                list-type="picture-card"
                :file-list="fileList"
                @preview="handlePreview"
                @change="handleChange"
            >
              <div v-if="fileList.length < 4">
                <a-icon type="plus"/>
                <div class="ant-upload-text">
                  Upload
                </div>
              </div>
            </a-upload>
            <a-modal :visible="previewVisible" :footer="null" @cancel="handleCancel">
              <img alt="example" style="width: 100%" :src="previewImage"/>
            </a-modal>
          </div>
        </a-form-item>
        <a-form-item style="margin-top: 24px" :wrapperCol="{span: 10, offset: 7}">
          <a-button @click="handleSubmit" type="primary">确定</a-button>
          <a-button @click="visible=false" style="margin-left: 8px">取消</a-button>
        </a-form-item>
      </a-form>
    </a-drawer>
  </div>
</template>
<script>
import {request, METHOD} from '@/utils/request'

function getBase64(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.readAsDataURL(file);
    reader.onload = () => resolve(reader.result);
    reader.onerror = error => reject(error);
  });
}

export default {
  name: 'jishui',
  components: {},
  data() {
    return {
      map: null,
      markerLayer: null,
      loading: false,
      loadingMore: false,
      showLoadingMore: false,
      query: {
        page: 0,
        size: 20,
      },
      listData: [],
      infoWindowList: [],
      suggestionList: [],
      visible: false,
      // 上传管理
      previewVisible: false,
      previewImage: '',
      form: this.$form.createForm(this),
      fileList: [
        {
          uid: '-1',
          name: 'image.png',
          status: 'done',
          url: 'https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png',
        },
        {
          uid: '-2',
          name: 'image.png',
          status: 'done',
          url: 'https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png',
        },
        {
          uid: '-3',
          name: 'image.png',
          status: 'done',
          url: 'https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png',
        },
        {
          uid: '-4',
          name: 'image.png',
          status: 'done',
          url: 'https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png',
        },
      ],
      // 表单内容
      formData: {
        lat: '',
        lng: '',
        name: '',
        area: '',
        road: '',
        projectName: '',
        projectContent: '',
        projectUnit: '',
        projectPeriod: '',
        process: '',
        pictures: [],
      }
    }
  },
  computed: {},
  mounted() {
    this.getData();
    this.initMap();
  },
  methods: {
    onLoadMore() {
      this.getData();
    },
    getData() {
      this.query.page++;
      this.loading = true
      request("/jishui/page", METHOD.GET, this.query).then((res) => {
        this.loading = false;
        let result = res.data.data.data;
        this.showLoadingMore = true;
        if (result.length < this.query.size) {
          this.showLoadingMore = false
        }
        this.listData = this.listData.concat(result);
        console.log(this.listData)
      })
    },
    resetQuery() {
      this.query.page = 0;
      this.listData = [];
    },
    editArea(item) {
      let position = new window.TMap.LatLng(item.lat, item.lng)
      var infoWindow = new window.TMap.InfoWindow({
        map: this.map,
        position: position,
        content: `<h3>${item.name}</h3><p>地址：${item.address}</p><p>电话：${item.tel}</p>`,
        offset: {x: 0, y: -50},
      }); // 新增信息窗体显示地标的名称与地址、电话等信息
      infoWindow.open();
      this.map.setCenter(position);
      this.formData = item;
      this.visible = true;
    },
    async delArea(item) {
      await request("/jishui/" + item.id, METHOD.DELETE)
      this.resetQuery();
      this.getData()
    },
    onClose() {
      this.visible = false;
    },
    handleSubmit(e) {
      e.preventDefault()
      this.formData.pictures = this.fileList.map((item) => {
        return item.url
      });
      let that = this;
      this.form.validateFields((err, values) => {
        if (!err) {
          console.log('Received values of form: ', values)
          request("/jishui", this.formData.id ? METHOD.PUT : METHOD.POST, this.formData).then(res => {
            that.markerLayer.add({
              position: that.formData.latLng
            });
            that.visible = false
            console.log(res)
            this.resetQuery();
            this.getData();
          })
        }
      })
    },
    resetForm() {
      this.formData = {
        lat: '',
        lng: '',
        name: '',
        road: '',
        projectName: '',
        projectContent: '',
        projectUnit: '',
        projectPeriod: '',
        area: '',
        process: '',
        pictures: [
          {
            uid: '-1',
            name: 'image.png',
            status: 'done',
            url: 'https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png',
          },
          {
            uid: '-2',
            name: 'image.png',
            status: 'done',
            url: 'https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png',
          },
          {
            uid: '-3',
            name: 'image.png',
            status: 'done',
            url: 'https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png',
          },
          {
            uid: '-4',
            name: 'image.png',
            status: 'done',
            url: 'https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png',
          },
        ],
      }
    },
    handleCancel() {
      this.previewVisible = false;
    },
    async handlePreview(file) {
      if (!file.url && !file.preview) {
        file.preview = await getBase64(file.originFileObj);
      }
      this.previewImage = file.url || file.preview;
      this.previewVisible = true;
    },
    handleChange({fileList}) {
      this.fileList = fileList;
    },
    initMap() {
      console.log(window); // 通过window获取
      const {TMap} = window;
      const center = new window.TMap.LatLng(34.261792, 117.184811);
      const code = 320302;
      //初始化地图
      var map = new window.TMap.Map("container", {
        center: center,//设置地图中心点坐标
        zoom: 17.2,   //设置地图缩放级别
        pitch: 43.5,  //设置俯仰角
        rotation: 45,    //设置地图旋转角度
        code: code
      })
      //初始化marker图层
      var markerLayer = new TMap.MultiMarker({
        id: 'marker-layer',
        map: map
      });
      //定义事件处理方法
      let that = this;
      var clickHandler = function (evt) {
        var lat = evt.latLng.getLat().toFixed(6);
        var lng = evt.latLng.getLng().toFixed(6);
        console.log("您点击的的坐标是：" + lat + "," + lng);
        that.formData.lat = lat
        that.formData.lng = lng
        that.formData.latLng = evt.latLng
        that.visible = true;
      }
      //Map实例创建后，通过on方法绑定点击事件
      map.on("click", clickHandler)
      this.map = map;
      this.markerLayer = markerLayer;
      console.log(map);
      // this.deleteSomeInfo();
    },
    searchByKeyword() {
      var search = new window.TMap.service.Search({pageSize: 10}); // 新建一个地点搜索类
      var markers = new window.TMap.MultiMarker({
        map: this.map,
        geometries: [],
      });
      var infoWindowList = Array(10);
      infoWindowList.forEach((infoWindow) => {
        infoWindow.close();
      });
      infoWindowList.length = 0;
      markers.setGeometries([]);
      // 在地图显示范围内以给定的关键字搜索地点
      search
          .searchRectangle({
            // keyword: document.getElementById('keyword').value,
            keyword: "地铁",
            bounds: this.map.getBounds(),
          })
          .then((result) => {
            result.data.forEach((item, index) => {
              var geometries = markers.getGeometries();
              var infoWindow = new window.TMap.InfoWindow({
                map: this.map,
                position: item.location,
                content: `<h3>${item.title}</h3><p>地址：${item.address}</p><p>电话：${item.tel}</p>`,
                offset: {x: 0, y: -50},
              }); // 新增信息窗体显示地标的名称与地址、电话等信息
              infoWindow.close();
              infoWindowList[index] = infoWindow;
              geometries.push({
                id: String(index), // 点标注数据数组
                position: item.location,
              });
              markers.updateGeometries(geometries); // 绘制地点标注
              markers.on('click', (e) => {
                infoWindowList[Number(e.geometry.id)].open();
              }); // 点击标注显示信息窗体
            });
          });
    },
    getSuggestions() {
      const {TMap} = window;
      let {map} = this;
      // 新建一个地点搜索类
      const suggest = new TMap.service.Suggestion({
        // 新建一个关键字输入提示类
        pageSize: 10, // 返回结果每页条目数
        region: '徐州', // 限制城市范围
        regionFix: true, // 搜索无结果时是否固定在当前城市
      });
      // 使用者在搜索框中输入文字时触发
      var keyword = document.getElementById('keyword').value;
      if (keyword) {
        suggest
            .getSuggestions({keyword: keyword, location: map.getCenter()})
            .then((result) => {
              // 以当前所输入关键字获取输入提示
              this.suggestionList = result.data;
            })
            .catch((error) => {
              console.log(error);
            });
      }
    },
    // 高亮选择的地址点
    setSuggestion(index) {
      const {TMap} = window;
      const {map, suggestionList, infoWindowList} = this;
      var markers = new window.TMap.MultiMarker({
        map: this.map,
        geometries: [],
      });
      // 点击输入提示后，于地图中用点标记绘制该地点，并显示信息窗体，包含其名称、地址等信息
      infoWindowList.forEach((infoWindow) => {
        infoWindow.close();
      });
      infoWindowList.length = 0;
      document.getElementById('keyword').value = suggestionList[index].title;
      document.getElementById('idSuggestionList').innerHTML = '';
      markers.setGeometries([]);
      markers.updateGeometries([
        {
          id: '0', // 点标注数据数组
          position: suggestionList[index].location,
        },
      ]);
      var infoWindow = new TMap.InfoWindow({
        map: map,
        position: suggestionList[index].location,
        content: `<h3>${suggestionList[index].title}</h3><p>地址：${suggestionList[index].address}</p>`,
        offset: {x: 0, y: -50},
      });
      infoWindowList.push(infoWindow);
      map.setCenter(suggestionList[index].location);
      markers.on('click', (e) => {
        infoWindowList[Number(e.geometry.id)].open();
      });
    },
    // 去除无用信息
    deleteSomeInfo() {
      // 腾讯地图去除logo
      var logo = document.querySelector(
          "#QQMap > div > div:nth-child(2) > div:nth-child(1) > div:nth-child(2)"
      );
      logo.setAttribute("style", "display: none;");
      // 腾讯地图去除logo文字
      var logoText = document.getElementsByClassName("logo-text");
      logoText[0].setAttribute("style", "display: none;");
      // 腾讯地图去除罗盘
      var logoCompass = document.getElementsByClassName("rotate-circle");
      logoCompass[0].setAttribute("style", "display: none;");
      // 获取缩放控件实例
      let control = this.map.getController(window.TMap.constants.DEFAULT_CONTROL_ID.ZOOM);
      // 设置控件在右下角
      control.setPosition(window.TMap.constants.CONTROL_POSITION.BOTTOM_RIGHT);
      // 缩放控件显示缩放级别
      control.setNumVisible(true);
    }
  }
}
</script>
<style lang="less" scoped>
.background {
  width: 100%;
  height: 100%;
  position: relative;
  margin: 10px;

  .box {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    z-index: 1001;
    width: 200px;
    height: 100%;
    overflow: hidden;
    background-color: rgba(128, 128, 128, 0.8);

    .left-list {
      width: 100%;
      height: 100%;
      overflow: auto;
    }
  }
}

#panel {
  position: absolute;
  background: #FFF;
  width: 350px;
  padding: 20px;
  z-index: 1001;
  top: 0px;
  left: 200px;
}

#suggestionList {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

#suggestionList li a {
  margin-top: -1px;
  background-color: #f6f6f6;
  text-decoration: none;
  font-size: 18px;
  color: black;
  display: block;
}

#suggestionList li .item_info {
  font-size: 12px;
  color: grey;

}

#suggestionList li a:hover:not(.header) {
  background-color: #eee;
}
</style>