<template>
  <div class="hello">
    <div class="pages">
      <div class="page" v-for="(page, index) in pages" :key="`page${index}`" @click="currentPage = index" v-bind:class="{ active: currentPage === index }">
        <div class="thumb" v-bind:style="{ background: page.backgroundColor }"></div>
        <div class="index" v-text="index"></div>
      </div>
      <div class="add" @click="addPage"></div>
      <div class="download-all" @click="downloadAll"></div>
    </div>

    <div class="page-setting">
      <!-- <input class="title" type="text" v-model="pageInfo.name"> -->
      <div class="download" @click="saveToPng"></div>
    </div>

    <div class="editor-wrap" id="editor-wrap">
      <div class="header"></div>
      <div class="editor-toolbar" v-if="isEditMode">
        <div @click="saveEditor" class="save"></div>
        <div @click="removeEditor" class="cancel"></div>
      </div>
      <div class="content">
        <div class="layer" v-if="isEditMode" style="z-index: 9999; width:100%;height:100%;">
          <vueCropper
          ref="cropper"
          :img="pageInfo.image"
          :outputSize="1000"
          outputType="png"
          :autoCrop="true"
          :canMoveBox="false"
          :autoCropWidth="1000"
          :autoCropHeight="1000"
          :fixed="true"
          :info="false"
          :fixedNumber="[1,1]"
          ></vueCropper>
        </div>
        <div class="layer watermark" v-bind:class="pageInfo.watermark.type" v-bind:style="{
          zIndex: 9998,
          backgroundPosition: `right ${pageInfo.watermark.position[0]}px top ${pageInfo.watermark.position[1]}px`
        }"></div>
        <div class="layer" v-for="(layer, index) in pageInfo.layer" :key="`layer${index}`" v-bind:style="{ zIndex: 999 - index }">
          <vue-draggable-resizable :resizable="false" :parent="true">
            <img :src="layer">
          </vue-draggable-resizable>
        </div>
        <div class="layer" v-bind:style="{ background: pageInfo.backgroundColor, zIndex: 0 }"></div>
      </div>
      <div class="footer" v-bind:class="{ active: pageInfo.heart }" @click="pageInfo.heart = !pageInfo.heart"></div>
    </div>

    <div class="colorpicker">
      <chrome-picker :value="pageInfo.backgroundColor" @input="updateBackground"></chrome-picker>
    </div>

    <div class="page-layer">
      <div class="setting">
        <div class="title">
          워터마크
        </div>
        <div class="title">
          <span style="margin-right: 4px;" @click="pageInfo.watermark.type = 'none'"><span v-if="pageInfo.watermark.type === 'none'">✔</span> 없음 </span>
          <span style="margin-right: 4px;" @click="pageInfo.watermark.type = 'gray'"><span v-if="pageInfo.watermark.type === 'gray'">✔</span> 그레이 </span>
          <span style="margin-right: 4px;" @click="pageInfo.watermark.type = 'white'"><span v-if="pageInfo.watermark.type === 'white'">✔</span> 화이트 </span>
        </div>
      </div>
      <draggable v-model="pageInfo.layer">
        <div class="photo" v-for="(image, index) in pageInfo.layer" :key="`page-layer${index}`">
          <img :src="image">
          <div class="add-this" @click="pageInfo.layer.splice(index, 1)"></div>
        </div>
      </draggable>
    </div>

    <div class="album" @dragover.prevent @drop="onDrop">
      <div v-show="album.length < 1" class="notice">
        이미지를 이 곳에 드래그 앤 드롭 하세요
      </div>
      <div class="photos" v-show="album.length > 0" v-bind:style="{ width: `${60 * album.length} px` }">
        <div class="photo" v-for="(image, index) in album" :key="`album-image${index}`" v-bind:style="{ 'background-image': `url(${image})` }">
          <!--<img :src="image">-->
          <div class="add-this" @click="editImage(index)"></div>
        </div>
      </div>
    </div>

  </div>
</template>

<script>
import VueCropper from 'vue-cropper'
import VueDraggableResizable from 'vue-draggable-resizable'
import Draggable from 'vuedraggable'
import { Chrome } from 'vue-color'
import html2canvas from 'html2canvas'
import download from 'downloadjs'

export default {
  name: 'HelloWorld',
  components: {
    VueCropper,
    VueDraggableResizable,
    Draggable,
    'chrome-picker': Chrome
  },
  mounted () {
    // this.$refs.cropper.startCrop()
  },
  data () {
    return {
      currentPage: 0,
      pages: [
        {
          watermark: {
            type: 'none',
            position: [30, 20]
          },
          name: 'file-name',
          image: '',
          layer: [],
          backgroundColor: '#eeeeee',
          heart: false
        }
      ],
      page: {
        watermark: {
          type: 'none',
          position: [30, 20]
        },
        name: 'file-name',
        image: '',
        layer: [],
        backgroundColor: '#eeeeee',
        heart: false
      },
      album: [],
      isEditMode: false,
      isRendered: false
    }
  },
  methods: {
    onDrop: function (e) {
      console.log(e)
      e.stopPropagation()
      e.preventDefault()
      var files = e.target.files || e.dataTransfer.files
      if (files) {
        for (let i = 0; i < files.length; i++) {
          var reader = new FileReader()
          reader.onloadend = (e) => {
            this.album.push(this.dataURItoBlob(e.target.result))
          }
          reader.readAsDataURL(files[i])
        }
      }
    },
    dataURItoBlob: function (dataURI) {
      var byteString = atob(dataURI.split(',')[1])
      // var mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0]
      var ab = new ArrayBuffer(byteString.length)
      var ia = new Uint8Array(ab)
      for (var i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i)
      }

      var bb = new Blob([ab])
      return window.URL.createObjectURL(bb)
    },
    editImage: function (index) {
      this.isEditMode = true
      this.pageInfo.image = this.album[index]
    },
    saveEditor: function () {
      this.$refs.cropper.getCropBlob((data) => {
        this.pageInfo.layer.splice(0, 0, window.URL.createObjectURL(data))
        this.isEditMode = false
        this.pageInfo.image = ''
      })
    },
    removeEditor: function () {
      this.isEditMode = false
      this.pageInfo.image = ''
    },
    addPage: function () {
      const page = this.page
      this.pages.push(JSON.parse(JSON.stringify(page)))
      this.currentPage = this.pages.length - 1
    },
    updateBackground: function (value) {
      this.pageInfo.backgroundColor = `rgba(${value.rgba.r}, ${value.rgba.g}, ${value.rgba.b} ,${value.rgba.a})`
    },
    saveToPng: function () {
      if (this.isEditMode) {
        return alert('체크 버튼을 눌러 편집을 마치고 다운로드 해주세요.')
      }
      var filename = 'render-' + this.currentPage + '-' + Date.now() + '.jpg'
      html2canvas(document.getElementById('editor-wrap'), { allowTaint: true, scale: 1 }).then(function (canvas) {
        var data = canvas.toDataURL('image/jpeg', 0.9)
        download(data, filename, 'image/jpeg')
      })
    },
    downloadAll: async function () {
      if (this.isEditMode) {
        return alert('체크 버튼을 눌러 편집을 마치고 다운로드 해주세요.')
      }
      if (confirm('전부 내려받을까요? 다운로드 창이 여러번 뜰 수 있습니다.')) {
        for (let i = 0; i < this.pages.length; i++) {
          this.currentPage = i
          var filename = 'render-' + this.currentPage + '-' + Date.now() + '.jpg'
          await html2canvas(document.getElementById('editor-wrap'), { allowTaint: true }).then(function (canvas) {
            var data = canvas.toDataURL('image/jpeg', 0.9)
            download(data, filename, 'image/jpeg')
          })
        }
      }
    }
  },
  computed: {
    pageInfo: function () {
      return this.pages[this.currentPage]
    }
  }
}
</script>

<style lang="scss" scoped>
.hello {
  padding-top: 100px;
  padding-bottom: 170px;
  padding-left: 170px;
  button {
    border: 0;
    padding: 0 16px;
    line-height: 50px;
    border-radius: 8px;
    background-color: #228df4;
    outline: none;
    cursor: pointer;
    color: #ffffff;
  }
  .pages {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 50px;
    z-index: 1000;
    background-color: #ffffff;
    font-size: 0;
    box-shadow: 0 1px 0 #dddddd;
    .page {
      display: inline-block;
      width: 50px;
      height: 50px;
      margin: 0 6px;
      padding-top: 6px;
      font-size: 12px;
      .thumb {
        width: 50px;
        height: 24px;
        border-radius: 4px;
      }
      .index {
        font-size: 12px;
        text-align: center;
        line-height: 20px;
      }
      &.active {
        .thumb {
          border: 2px solid #000000;
        }
        .index {
          font-weight: 700;
        }
      }
    }
    .add {
      width: 50px;
      height: 50px;
      position: absolute;
      right: 0;
      top: 0;
      background-color: #ffffff;
      background-image: url('/static/simple-add-black.svg');
      background-size: 24px;
      background-position: center;
      background-repeat: no-repeat;
    }
    .download-all {
      width: 50px;
      height: 50px;
      position: absolute;
      right: 50px;
      top: 0;
      background-color: #ffffff;
      background-image: url('/static/hit-down-black.svg');
      background-size: 24px;
      background-position: center;
      background-repeat: no-repeat;
    }
  }
  .page-setting {
    width: 100%;
    text-align: center;
    .title {
      border: 0;
      background-color: rgba(0, 0, 0, 0);
      color: #ffffff;
      text-align: center;
    }
    .download {
      width: 32px;
      height: 32px;
      cursor: pointer;
      background-image: url('/static/hit-down.svg');
      background-repeat: no-repeat;
      background-position: center;
      background-size: 24px;
      margin: 0 auto 30px auto;
    }
  }
  .colorpicker {
    text-align: center;
    margin: 20px auto;
    width: 225px;
  }
  .editor-toolbar {
    position: absolute;
    right: -100px;
    top: 50%;
    width: 72px;
    .save {
      width: 72px;
      height: 72px;
      background-image: url('/static/check-2.svg');
      background-size: 48px;
      background-position: center;
      background-repeat: no-repeat;
      cursor: pointer;
    }
    .crop {
      width: 72px;
      height: 72px;
      background-image: url('/static/crop.svg');
      background-size: 48px;
      background-position: center;
      background-repeat: no-repeat;
      cursor: pointer;
    }
    .cancel {
      width: 72px;
      height: 72px;
      background-image: url('/static/simple-remove.svg');
      background-size: 48px;
      background-position: center;
      background-repeat: no-repeat;
      cursor: pointer;
    }
  }
  .editor-wrap {
    position: relative;
    margin: 0 auto;
    width: 1000px;
    height: 1236px;
    zoom: 0.6;
    .header {
      width: 100%;
      height: 122px;
      background-image: url('/static/header.png');
      background-size: 1000px 122px;
    }
    .content {
      width: 1000px;
      height: 1000px;
      position: relative;
      overflow: hidden;
      .layer {
        position: absolute;
        left: 0;
        right: 0;
        width: 1000px;
        height: 1000px;
        background-repeat: no-repeat;
        &.watermark {
          background-size: auto;
          &.none {
            background: none;
          }
          &.gray {
            background-image: url('/static/water_gray.png');
          }
          &.white {
            background-image: url('/static/water_white.png');
          }
        }
      }
    }
    .footer {
      width: 100%;
      height: 107px;
      background-image: url('/static/footer-off.png');
      background-size: 1000px 107px;
      &.active {
        background-image: url('/static/footer-on.png')
      }
    }
  }
  .album {
    position: fixed;
    bottom: 0;
    left: 0;
    display: flex;
    flex-wrap: nowrap;
    width: 100%;
    height: 90px;
    z-index: 999;
    background-color: #000000;
    font-size: 0;
    box-shadow: 0px 0 4px 1px rgba(0, 0, 0, 0.3);
    padding: 10px;
    overflow-x: auto;
    .notice {
      display: block;
      flex: 1;
      align-self: center;
      font-size: 14px;
      color: #ffffff;
      text-align: center;
    }
    .photos {
      display: flex;
    }
    .photo {
      position: relative;
      display: block;
      margin-right: 10px;
      width: 50px;
      height: 50px;
      border-radius: 4px;
      overflow: hidden;
      background-size: cover;
      background-position: center;
      background-repeat: no-repeat;
      &:hover {
        .add-this {
          opacity: 1;
        }
      }
      .add-this {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        z-index: 1;
        background-color: rgba(0, 0, 0, .5);
        background-image: url('/static/simple-add.svg');
        background-repeat: no-repeat;
        background-position: center;
        background-size: 24px;
        opacity: 0;
        transition: opacity 300ms;
        cursor: pointer
      }
    }
  }
  .page-layer {
    position: fixed;
    top: 0;
    left: 0;
    width: 160px;
    height: 100%;
    padding: 70px 10px 10px 10px;
    z-index: 990;
    background-color: #5e5e5e;
    font-size: 0;
    box-shadow: 0px 0 4px 1px rgba(0, 0, 0, 0.3);
    overflow-y: scroll;
    .setting {
      font-size: 12px;
      margin-bottom: 10px;
      color: #eeeeee;
      .title {
        line-height: 16px;
        span {
          cursor: pointer;
        }
      }
    }
    .photo {
      position: relative;
      margin-bottom: 10px;
      width: auto;
      height: auto;
      border-radius: 4px;
      overflow: hidden;
      img {
        border-radius: 4px;
        max-width: 140px;
      }
      &:hover {
        .add-this {
          opacity: 1;
        }
      }
      .add-this {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        z-index: 1;
        background-color: rgba(0, 0, 0, .5);
        background-image: url('/static/simple-delete.svg');
        background-repeat: no-repeat;
        background-position: center;
        background-size: 24px;
        opacity: 0;
        transition: opacity 300ms;
        cursor: pointer
      }
    }
  }
}
</style>
