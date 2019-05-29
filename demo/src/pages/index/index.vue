<template>
  <div>
    <div class="wrap">
      <Button class="show-btn" @tap="handleUpload">点击上传</Button>
      <div class="show-img"><img :src="destSrc"></div>
      <!-- 图片裁剪框 -->
      <image-cropper v-if="showCropper" id="image-cropper" 
         :width="width" :height="height" :imgSrc="imgSrc" export_scale="1"
         :limit_move="true" :disable_rotate="true"
        @confirmCut="handleConfirmCut"
        @cancelCut="handleCancelCut"
        >
      </image-cropper>
    </div>
  </div>
</template>

<script>
import ImageCropper from '@/componment/image-cropper.vue'
export default {
  data () {
    return {
      showCropper: false,
      width: 200, // 需要 宽度
      height: 200, // 需要 高度
      imgSrc: '', // 需要 图片源路径
      destSrc: '' // 不需要 最终
    }
  },
  
  components: {
    ImageCropper
  },

  methods: {
    handleUpload () { // 打开选择
      wx.showActionSheet({
        itemList: ['从相册中选择', '拍照'],
        itemColor: "#999",
        success: res => { // 正式选择图片
          let tapType = res.tapIndex === 0 ? 'album' : 'camera'
          wx.chooseImage({ 
            sizeType: ['original', 'compressed'],
            sourceType: [tapType],
            success: res => { // 打开裁剪框进行裁剪
              this.showCropper = true
              this.imgSrc = res.tempFilePaths[0]
            }
          })
        }
      })
    },
    handleConfirmCut (e) { // 确认裁剪
      // e.url  e.width  e.height  
      // 关闭裁剪框
      this.showCropper = false
      this.destSrc = e.url
      /*
      wx.cloud.uploadFile({
        // 指定上传到的云路径
        cloudPath: 'imgName.png',
        // 指定要上传的文件的小程序临时文件路径
        filePath: e.mp.detail.url,
      })
      */
    },
    handleCancelCut () { // 取消裁剪
      this.showCropper = false
    },
  }

}
</script>

<style>
.wrap{
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-around;
}
.show-btn{
  height: 260rpx;
  line-height: 260rpx;
  width: 260rpx;
  border-radius: 130rpx;
}
.show-img{
  height: 300rpx;
  width: 300rpx;
  border: 1rpx solid #eee;
}
.show-img img{
  width: 100%;
  height: 100%;
}
</style>
