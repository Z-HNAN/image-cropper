<template>
    <view class='image-cropper' @touchmove.stop='_preventTouchMove'>
    <view class='header'>
      <Button size="mini" type="primary" @tap="_confirmCut" class="op-btn confirm-btn">确认裁剪</Button>
      <view class="title">图片裁剪</view>
      <Button size="mini" type="primary" @tap="_cancelCut" class="op-btn cancel-btn">取消裁剪</Button>
    </view>
    <view class='main' @touchend="_cutTouchEnd" @touchstart="_cutTouchStart" @touchmove="_cutTouchMove" @tap="_click">
      <view class='content'>
        <view class='content_top bg_gray' :class="{'bg_black': c_flag_bright === true}" :style="contentTopStyle"></view>
        <view class='content_middle' :style="{height: height + 'px'}">
          <view class='content_middle_left bg_gray' :class="{'bg_black':c_flag_bright === true}" :style="contentMiddleLeftStyle"></view>
          <view class='content_middle_middle' :style="contentMiddleMiddleStyle">
            <view class="border border-top-left"></view>
            <view class="border border-top-right"></view>
            <view class="border border-right-top"></view>
            <view class="border border-right-bottom"></view>
            <view class="border border-bottom-right"></view>
            <view class="border border-bottom-left"></view>
            <view class="border border-left-bottom"></view>
            <view class="border border-left-top"></view>
          </view>
          <view class='content_middle_right bg_gray' :class="{'bg_black': c_flag_bright === true}" :style="{'transition-property': c_cut_animation === true ? 'background' : ''}"></view>
        </view>
        <view class='content_bottom bg_gray' :class="{'bg_black': c_flag_bright === true}" :style="{'transition-property': c_cut_animation === true ? 'background' : ''}"></view>
      </view>
      <image @load="imageLoad" @touchstart="_start" @touchmove="_move" @touchend="_end" :style="imageStyle" class='img' :src='imgSrc'></image>
    </view>
    <canvas canvas-id='image-cropper' disable-scroll="true" :style="canvasStyle" class='image-cropper-canvas'></canvas>
  </view>
</template>

<script>
export default {
  
  props: {
    /**     
    * 图片路径
    */
    'imgSrc': {
      type: String
    },
    /**
     * 裁剪框高度
     */
    'height': {
      type: Number,
      default: 200
    },
    /**
     * 裁剪框宽度
     */
    'width': {
      type: Number,
      default: 200
    },
    /**
     * 裁剪框最小尺寸
     */
    'min_width': {
      type: Number,
      default: 100
    },
    'min_height': {
      type: Number,
      default: 100
    },
    /**
     * 裁剪框最大尺寸
     */
    'max_width': {
      type: Number,
      default: 300
    },
    'max_height': {
      type: Number,
      default: 300
    },
    /**
     * 裁剪框禁止拖动
     */
    'disable_width': {
      type: Boolean,
      default: false
    },
    'disable_height': {
      type: Boolean,
      default: false
    },
    /**
     * 锁定裁剪框比例
     */
    'disable_ratio':{
      type: Boolean,
      default: false
    },
    /**
     * 生成的图片尺寸相对剪裁框的比例
     */
    'export_scale': {
      type: Number,
      default: 3
    },
    /**
     * 生成的图片质量0-1
     */
    'quality': {
      type: Number,
      default: 1
    },
    'cut_top': {
      type: Number,
      default: null
    },
    'cut_left': {
      type: Number,
      default: null
    },
    /**
     * canvas上边距（不设置默认不显示）
     */
    'canvas_top': {
      type: Number,
      default: null
    },
    /**
     * canvas左边距（不设置默认不显示）
     */
    'canvas_left': {
      type: Number,
      default: null
    },
    /**
     * 图片宽度
     */
    'img_width': {
      type: null,
      default: null
    },
    /**
     * 图片高度
     */
    'img_height': {
      type: null,
      default: null
    },
    /**
     * 图片缩放比
     */
    'scale': {
      type: Number,
      default: 1
    },
    /**
     * 图片旋转角度
     */
    'angle': {
      type: Number,
      default: 0
    },
    /**
     * 最小缩放比
     */
    'min_scale': {
      type: Number,
      default: 0.5
    },
    /**
     * 最大缩放比
     */
    'max_scale': {
      type: Number,
      default: 2
    },
    /**
     * 是否禁用旋转
     */
    'disable_rotate': {
      type: Boolean,
      default: false
    },
    /**
     * 是否限制移动范围(剪裁框只能在图片内)
     */
    'limit_move':{
      type: Boolean,
      default: false
    }
  },
  
  data() {
    return {
      el: 'image-cropper', //暂时无用
      info: wx.getSystemInfoSync(),
      ctx: '', // canvas上下文对象
      MOVE_THROTTLE: null,//触摸移动节流settimeout 40hz
      MOVE_THROTTLE_FLAG: true,//节流标识
      INIT_IMGWIDTH: 0, //图片设置尺寸,此值不变（记录最初设定的尺寸）
      INIT_IMGHEIGHT: 0, //图片设置尺寸,此值不变（记录最初设定的尺寸）
      TIME_BG: null,//背景变暗延时函数
      TIME_CUT_CENTER:null,
      CUT_START: { }, // 裁剪框相关信息
      c_touch_img_relative: [{
        x: 0,
        y: 0
      }], //鼠标和图片中心的相对位置
      c_flag_cut_touch:false,//是否是拖动裁剪框
      c_hypotenuse_length: 0, //双指触摸时斜边长度
      c_flag_img_endtouch: false, //是否结束触摸
      c_flag_bright: true, //背景是否亮
      c_canvas_overflow:true,//canvas缩略图是否在屏幕外面
      c_canvas_width:200,
      c_canvas_height:200,
      origin_x: 0.5, //图片旋转中心
      origin_y: 0.5, //图片旋转中心
      c_cut_animation: false,//是否开启图片和裁剪框过渡
      c_img_top: wx.getSystemInfoSync().windowHeight / 2, //图片上边距
      c_img_left: wx.getSystemInfoSync().windowWidth / 2 //图片左边距
    }
  },
  
  computed: {
    contentTopStyle () { // content_top
      return styles({
        'height': this.cut_top + 'px',
        'transition-roperty': this.c_cut_animation === true ? 'background' : '',
      })
    },
    contentMiddleLeftStyle () { //  content_middle_left
      return styles({
        'width': this.cut_left + 'px',
        'transition-roperty': this.c_cut_animation === true ? 'background' : ''
      })
    },
    contentMiddleMiddleStyle () { // content_middle_middle
      return styles({
        'width': this.width + 'px',
        'height': this.height + 'px',
        'transition-duration': '.3s',
        'transition-property': this.c_cut_animation === true ? 'background' : ''
      })
    },
    imageStyle () { // image
      return styles({
        'width': this.img_width ? this.img_width  + 'px' : 'auto',
        'height': this.img_height ? this.img_height  + 'px' : 'auto',
        'transform': `translate3d(${this.c_img_left - this.img_width / 2}px, ${this.c_img_top - this.img_height / 2}px, 0) scale(${this.scale}) rotate(${this.angle}deg)`,
        'transition-duration': this.c_cut_animation ? '.4s' : '0'
      })
    },
    canvasStyle () { //canvas
      return styles({
        'width': this.c_canvas_width * this.export_scale + 'px',
        'height': this.c_canvas_height * this.export_scale + 'px',
        'left': this.canvas_left + 'px',
        'top': this.canvas_top + 'px',
      })
    }
  },
  
  watch: {
    //监听截取框宽高变化
    width(value) {
      if (value < this.min_width){
        this.width = this.min_width
      }
      this._computeCutSize();
    },
    height(value) {
      if (value < this.min_height) {
        this.height = this.min_height
      }
      this._computeCutSize();
    },
    angle(value){
      //停止居中裁剪框，继续修改图片位置
      this._moveStop();
      if(this.limit_move){
        if (this.angle % 90) {
          this.angle = Math.round(that.data.angle / 90) * 90
        }
      }
    },
    c_cut_animation(value){
      //开启过渡300毫秒之后自动关闭
      clearTimeout(this.c_cut_animation_time);
      if (value){
        this.c_cut_animation_time = setTimeout(()=>{
          this.c_cut_animation = false
        },300)
      }
    },
    limit_move(value){
      if (value) {
        if (this.angle%90){
          this.angle = Math.round(that.data.angle / 90)*90
        }
        this._imgMarginDetectionScale();
        !this.c_canvas_overflow && this._draw();
      }
    },
    canvas_top(value){
      this._canvasDetectionPosition();
    },
    canvas_left(value){
      this._canvasDetectionPosition();
    },
    imgSrc(value){
      this.pushImg();
    },
    cut_top(value) {
      this._cutDetectionPosition();
      if (this.limit_move) {
        !this.c_canvas_overflow && this._draw();
      }
    },
    cut_left(value) {
      this._cutDetectionPosition();
      if (this.limit_move) {
        !this.c_canvas_overflow && this._draw();
      }
    }
  },
 
  methods:{
    /**
     * 返回图片信息
     */
    getImg(getCallback) {
      this._draw(()=>{
        wx.canvasToTempFilePath({
          width: this.data.width * this.data.export_scale,
          height: Math.round(this.data.height * this.data.export_scale),
          destWidth: this.data.width * this.data.export_scale,
          destHeight: Math.round(this.data.height) * this.data.export_scale,
          fileType: 'png',
          quality: this.data.quality,
          canvasId: this.data.el,
          success: (res) => {
            getCallback({
              url: res.tempFilePath,
              width: this.data.width * this.data.export_scale,
              height: this.data.height * this.data.export_scale
            });
          }
        }, this)
      });
    },
    /**
       * 设置图片动画
       * {
       *    x:10,//图片在原有基础上向下移动10px
       *    y:10,//图片在原有基础上向右移动10px
       *    angle:10,//图片在原有基础上旋转10deg
       *    scale:0.5,//图片在原有基础上增加0.5倍
       * }
       */
      setTransform(transform) {
        if (!transform) return;
        if (!this.disable_rotate){
          this.angle = transform.angle ? this.angle + transform.angle : this.angle
        }
        var scale = this.scale;
        if (transform.scale) {
          scale = this.scale + transform.scale;
          scale = scale <= this.min_scale ? this.min_scale : scale;
          scale = scale >= this.max_scale ? this.max_scale : scale;
        }
        this.scale = scale;
        let cutX = this.cut_left;
        let cutY = this.cut_top;
        if (transform.cutX){
          this.cut_left = cutX + transform.cutX
          // this.data.watch.cut_left(null, this); [?]
        }
        if (transform.cutY){
          this.cut_top = cutY + transform.cutY
          // this.data.watch.cut_top(null, this); [?]
        }
        this.c_img_top = transform.y ? this.c_img_top + transform.y : this.c_img_top;
        this.c_img_left = transform.x ? this.c_img_left + transform.x : this.c_img_left;
        //图像边缘检测,防止截取到空白
        this._imgMarginDetectionScale();
        //停止居中裁剪框，继续修改图片位置
        this._moveDuring();
        !this.data.c_canvas_overflow && this._draw();
        //可以居中裁剪框了
        this._moveStop();//结束操作
      },
      /**
       * 设置剪裁框位置
       */
      setCutXY(x,y){
        this.cut_top = y
        this.cut_left = x
      },
      /**
       * 设置剪裁框尺寸
       */
      setCutSize(w,h){
        this.width = w
        this.height = h
        this._computeCutSize();
      },
      /**
       * 设置剪裁框和图片居中
       */
      setCutCenter() {
        let cut_top = (this.info.windowHeight - this.height) * 0.5;
        let cut_left = (this.info.windowWidth - this.width) * 0.5;
        //顺序不能变
        this.c_img_top = this.c_img_top - this.cut_top + cut_top
        this.cut_top = cut_top //截取的框上边距
        this.c_img_left = this.c_img_left - this.cut_left + cut_left
        this.cut_left = cut_left //截取的框左边距
      },
      _setCutCenter(){
        let cut_top = (this.info.windowHeight - this.height) * 0.5;
        let cut_left = (this.info.windowWidth - this.width) * 0.5;
        this.cut_top = cut_top //截取的框上边距
        this.cut_left = cut_left //截取的框左边距
      },
      /**
       * 设置剪裁框宽度-即将废弃
       */
      setWidth(width) {
        this.width = width
        this._computeCutSize();
      },
      /**
       * 设置剪裁框高度-即将废弃
       */
      setHeight(height) {
        this.height = height
        this._computeCutSize();
      },
      /**
       * 是否锁定旋转
       */
      setDisableRotate(value){
        this.disable_rotate = value;
      },
      /**
       * 是否限制移动
       */
      setLimitMove(value){
          this.c_cut_animation =  true
          this.limit_move =  !!value
      },
      /**
       * 初始化图片，包括位置、大小、旋转角度
       */
      imgReset() {
        this.scale = 1
        this.angle = 0
        this.c_img_top = wx.getSystemInfoSync().windowHeight / 2
        this.c_img_left = wx.getSystemInfoSync().windowWidth / 2
      },
      /**
       * 加载（更换）图片
       */
      pushImg(src) {
        if (src) {
          this.imgSrc = src
          //发现是手动赋值直接返回，交给watch处理
          return;
        }
        wx.getImageInfo({
          src: this.imgSrc,
          success: (res) => {
            this.imageObject = res;
            //图片非本地路径需要换成本地路径
            if (this.imgSrc.search(/tmp/) == -1){
              this.imgSrc = res.path
            }
            //计算最后图片尺寸
            this._imgComputeSize();
            if (this.limit_move) {
              //限制移动，不留空白处理
              this._imgMarginDetectionScale();
            }
            this._draw();
          },
          fail: (err) => {
            this.imgSrc = ''
          }
        });
      },
      imageLoad(e){
        let timeId
        timeId = setTimeout(()=>{
          this.$emit('imageload', this.imageObject);
          clearTimeout(timeId)
        },1000)
      },
      /**
       * 设置图片放大缩小
       */
      setScale(scale) {
        if (!scale) return;
        this.scale = scale
        !this.c_canvas_overflow && this._draw();
      },
      /**
       * 设置图片旋转角度
       */
      setAngle(angle) {
        if (!angle) return;
        this.c_cut_animation = true,
        this.angle = angle
        this._imgMarginDetectionScale();
        !this.c_canvas_overflow && this._draw();
      },
      _initCanvas() {
        //初始化canvas
        this.ctx = wx.createCanvasContext("image-cropper", this);
      },
      /**
       * 根据开发者设置的图片目标尺寸计算实际尺寸
       */
      _initImageSize(){
        //处理宽高特殊单位 %>px
        if (this.INIT_IMGWIDTH && typeof this.INIT_IMGWIDTH == "string" && this.INIT_IMGWIDTH.indexOf("%") != -1) {
          let width = this.INIT_IMGWIDTH.replace("%", "");
          this.INIT_IMGWIDTH = this.img_width = this.info.windowWidth / 100 * width;
        }
        if (this.INIT_IMGHEIGHT && typeof this.INIT_IMGHEIGHT == "string" && this.INIT_IMGHEIGHT.indexOf("%") != -1) {
          let height = this.img_height.replace("%", "");
          this.INIT_IMGHEIGHT = this.img_height = this.info.windowHeight / 100 * height;
        }
      },
      /**
       * 检测剪裁框位置是否在允许的范围内(屏幕内)
       */
      _cutDetectionPosition(){
        let _cutDetectionPositionTop = () => {
          //检测上边距是否在范围内
          if (this.cut_top < 0) {
            this.cut_top = 0
          }
          if (this.cut_top > this.info.windowHeight - this.height) {
              this.cut_top = this.info.windowHeight - this.height
          }
        } 
        let _cutDetectionPositionLeft = () => {
          //检测左边距是否在范围内
          if (this.cut_left < 0) {
              this.cut_left = 0
          }
          if (this.cut_left > this.info.windowWidth - this.width) {
              this.cut_left = this.info.windowWidth - this.width
          }
        };
        //裁剪框坐标处理（如果只写一个参数则另一个默认为0，都不写默认居中）
        if (this.cut_top == null && this.cut_left == null) {
          // this._setCutCenter();
        } else if (this.cut_top != null && this.cut_left != null){
          _cutDetectionPositionTop();
          _cutDetectionPositionLeft();
        } else if (this.cut_top != null && this.cut_left == null) {
          _cutDetectionPositionTop();
          this.cut_left = (this.info.windowWidth - this.width) / 2
        } else if (this.cut_top == null && this.cut_left != null) {
          _cutDetectionPositionLeft();
          this.cut_top = (this.data.info.windowHeight - this.data.height) / 2
        }
      },
      /**
       * 检测canvas位置是否在允许的范围内(屏幕内)如果在屏幕外则不开启实时渲染
       * 如果只写一个参数则另一个默认为0，都不写默认超出屏幕外
       */
      _canvasDetectionPosition () {
        if(this.canvas_top == null && this.canvas_left == null) {
          this.c_canvas_overflow = false;
          this.canvas_top = -5000
          this.canvas_left = -5000
        }else if(this.canvas_top != null && this.canvas_left != null) {
          if (this.canvas_top < - this.height || this.canvas_top > this.info.windowHeight) {
            this.c_canvas_overflow = true;
          } else {
            this.c_canvas_overflow = false;
          }
        }else if(this.canvas_top != null && this.canvas_left == null) {
          this.canvas_left = 0
        } else if (this.canvas_top == null && this.canvas_left != null) {
          this.canvas_top = 0
          if (this.canvas_left < -this.width || this.canvas_left > this.info.windowWidth) {
            this.c_canvas_overflow = true;
          } else {
            this.c_canvas_overflow = false;
          }
        }
      },
      /**
       * 图片边缘检测-位置
       */
      _imgMarginDetectionPosition(scale) {
        if (!this.limit_move) return;
        let left = this.c_img_left;
        let top = this.c_img_top;
        var scale = scale || this.scale;
        let img_width = this.img_width;
        let img_height = this.img_height;
        if (this.angle / 90 % 2) {
          img_width = this.img_height;
          img_height = this.img_width;
        }
        left = this.cut_left + img_width * scale / 2 >= left ? left : this.cut_left + img_width * scale / 2;
        left = this.cut_left + this.width - img_width * scale / 2 <= left ? left : this.cut_left + this.width - img_width * scale / 2;
        top = this.cut_top + img_height * scale / 2 >= top ? top : this.cut_top + img_height * scale / 2;
        top = this.cut_top + this.height - img_height * scale / 2 <= top ? top : this.cut_top + this.height - img_height * scale / 2;
        this.c_img_left = left
        this.c_img_top = top
        this.scale = scale
      },
      /**
       * 图片边缘检测-缩放
       */
      _imgMarginDetectionScale(){
        if (!this.limit_move) return;
        let scale = this.scale;
        let img_width = this.img_width;
        let img_height = this.img_height;
        if (this.angle / 90 % 2) {
          img_width = this.img_height;
          img_height = this.img_width;
        }
        if (img_width * scale < this.width){
          scale = this.width / img_width;
        }
        if (img_height * scale < this.height) {
          scale = Math.max(scale,this.height / img_height);
        }
        this._imgMarginDetectionPosition(scale);
      },
      _setData(obj) {
        let data = {};
        for (var key in obj) {
          if (this.data[key] != obj[key]){
            data[key] = obj[key];
          }
        }
        this.setData(data);
        return data;
      },
      /**
       * 计算图片尺寸
       */
      _imgComputeSize() {
        let img_width = this.img_width,
            img_height = this.img_height;
        if (!this.INIT_IMGHEIGHT && !this.INIT_IMGWIDTH) {
          //默认按图片最小边 = 对应裁剪框尺寸
          img_width = this.imageObject.width;
          img_height = this.imageObject.height;
          if (img_width / img_height > this.width / this.height){
            img_height = this.height;
            img_width = this.imageObject.width / this.imageObject.height * img_height;
          }else{
            img_width = this.width;
            img_height = this.imageObject.height / this.imageObject.width * img_width;
          }
        } else if (this.INIT_IMGHEIGHT && !this.INIT_IMGWIDTH) {
          img_width = this.imageObject.width / this.imageObject.height * this.INIT_IMGHEIGHT;
        } else if (!this.INIT_IMGHEIGHT && this.INIT_IMGWIDTH) {
          img_height = this.imageObject.height / this.imageObject.width * this.INIT_IMGWIDTH;
        }
        this.img_width = img_width,
        this.img_height = img_height
      },
      //改变截取框大小
      _computeCutSize() {
        if (this.width > this.info.windowWidth) {
          this.width = this.info.windowWidth
        } else if (this.width + this.cut_left > this.info.windowWidth) {
          this.cut_left = this.info.windowWidth - this.cut_left
        }
        if (this.height > this.info.windowHeight) {
          this.height = this.info.windowHeight
        } else if (this.height + this.cut_top > this.info.windowHeight){
          this.cut_top  = this.info.windowHeight - this.cut_top
        }
        !this.c_canvas_overflow && this._draw();
      },
      //开始触摸
      _start(event) {
        this.c_flag_img_endtouch = false;
        if (event.touches.length == 1) {
          //单指拖动
          this.c_touch_img_relative[0] = {
            x: (event.touches[0].clientX - this.c_img_left),
            y: (event.touches[0].clientY - this.c_img_top)
          }
        } else {
          //双指放大
          let width = Math.abs(event.touches[0].clientX - event.touches[1].clientX);
          let height = Math.abs(event.touches[0].clientY - event.touches[1].clientY);
          this.c_touch_img_relative = [{
            x: (event.touches[0].clientX - this.c_img_left),
            y: (event.touches[0].clientY - this.c_img_top)
          }, {
            x: (event.touches[1].clientX - this.c_img_left),
            y: (event.touches[1].clientY - this.c_img_top)
          }];
          this.c_hypotenuse_length = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2));
        }
        !this.c_canvas_overflow && this._draw();
      },
      _move_throttle(){
        //安卓需要节流
        if (this.info.platform =='android'){
          clearTimeout(this.MOVE_THROTTLE);
          this.MOVE_THROTTLE = setTimeout(() => {
            this.MOVE_THROTTLE_FLAG = true;
          }, 1000 / 40)
          return this.MOVE_THROTTLE_FLAG;
        }else{
          this.MOVE_THROTTLE_FLAG = true;
        }
      },
      _move(event) {
        if (this.c_flag_img_endtouch || !this.MOVE_THROTTLE_FLAG) return;
        this.MOVE_THROTTLE_FLAG = false;
        this._move_throttle();
        this._moveDuring();
        if (event.touches.length == 1) {
          //单指拖动
          let left = (event.touches[0].clientX - this.c_touch_img_relative[0].x),
              top = (event.touches[0].clientY - this.c_touch_img_relative[0].y);
          this._imgMarginDetectionPosition();
          //图像边缘检测,防止截取到空白
          this.c_img_left = left;
          this.c_img_top = top;
        } else {
          //双指放大
          let width = (Math.abs(event.touches[0].clientX - event.touches[1].clientX)),
              height = (Math.abs(event.touches[0].clientY - event.touches[1].clientY)),
              hypotenuse = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2)),
              scale = this.scale * (hypotenuse / this.c_hypotenuse_length),
              current_deg = 0;
          scale = scale <= this.min_scale ? this.min_scale : scale;
          scale = scale >= this.max_scale ? this.max_scale : scale;
          //图像边缘检测,防止截取到空白
          this.scale = scale;
          this._imgMarginDetectionScale();
          //双指旋转(如果没禁用旋转)
          let c_touch_img_relative = [{
            x: (event.touches[0].clientX - this.c_img_left),
            y: (event.touches[0].clientY - this.c_img_top)
          }, {
            x: (event.touches[1].clientX - this.c_img_left),
            y: (event.touches[1].clientY - this.c_img_top)
          }];
          if (!this.disable_rotate){
            let first_atan = 180 / Math.PI * Math.atan2(c_touch_img_relative[0].y, c_touch_img_relative[0].x);
            let first_atan_old = 180 / Math.PI * Math.atan2(this.c_touch_img_relative[0].y, this.c_touch_img_relative[0].x);
            let second_atan = 180 / Math.PI * Math.atan2(c_touch_img_relative[1].y, c_touch_img_relative[1].x);
            let second_atan_old = 180 / Math.PI * Math.atan2(this.c_touch_img_relative[1].y, this.c_touch_img_relative[1].x);
            //当前旋转的角度
            let first_deg = first_atan - first_atan_old,
                second_deg = second_atan - second_atan_old;
            if (first_deg != 0) {
              current_deg = first_deg;
            } else if (second_deg != 0) {
              current_deg = second_deg;
            }
          }
          this.c_touch_img_relative = c_touch_img_relative;
          this.c_hypotenuse_length = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2));
          //更新视图
          this.angle = this.angle + current_deg,
          this.scale = this.scale
        }
        !this.c_canvas_overflow && this._draw();
      },
      //结束操作
      _end(event) {
        console.log('_end')
        this.c_flag_img_endtouch = true;
        this._moveStop();
      },
      //点击中间剪裁框处理
      _click(event) {
        this._draw(()=>{
          let x = event.detail ? event.detail.x : event.touches[0].clientX;
          let y = event.detail ? event.detail.y : event.touches[0].clientY;
          if ((x >= this.cut_left && x <= (this.cut_left + this.width)) && (y >= this.cut_top && y <= (this.cut_top + this.height))) {
            //生成图片并回调
            wx.canvasToTempFilePath({
              width: this.width * this.export_scale,
              height: Math.round(this.height * this.export_scale),
              destWidth: this.width * this.export_scale,
              destHeight: Math.round(this.height) * this.export_scale,
              fileType: 'png',
              quality: this.quality,
              canvasId: this.el,
              success: (res) => {
                this.$emit('tapcut', {
                  url: res.tempFilePath,
                  width: this.width * this.export_scale,
                  height: this.height * this.export_scale
                });
              }
            }, this)
          }
        });
      },
      //渲染
      _draw(callback) {
        if (!this.imgSrc) return;
        let draw = () => {
          //图片实际大小
          let img_width = this.img_width * this.scale * this.export_scale;
          let img_height = this.img_height * this.scale * this.export_scale;
          //canvas和图片的相对距离
          var xpos = this.c_img_left - this.cut_left;
          var ypos = this.c_img_top - this.cut_top;
          //旋转画布
          this.ctx.translate(xpos * this.export_scale, ypos * this.export_scale);
          this.ctx.rotate(this.angle * Math.PI / 180);
          console.log(-img_width / 2, -img_height / 2, img_width, img_height)
          this.ctx.drawImage(this.imgSrc, -img_width / 2, -img_height / 2, img_width, img_height);
          this.ctx.draw(false, () => {
            callback && callback();
          });
        }
        if (this.ctx.width != this.width || this.ctx.height != this.height){
          //优化拖动裁剪框，所以必须把宽高设置放在离用户触发渲染最近的地方
          this.c_canvas_height = this.height
          this.c_canvas_width = this.width
          let timeId
          //延迟20毫秒防止点击过快出现拉伸或裁剪过多
          timeId = setTimeout(() => {
            draw()
            clearTimeout(timeId)
          }, 20)
        }else{
          draw()
        }
      },
      //裁剪框处理
      _cutTouchMove(e) {
        if (this.c_flag_cut_touch && this.MOVE_THROTTLE_FLAG) {
          if (this.disable_ratio && (this.disable_width || this.disable_height)) return;
          //节流
          this.MOVE_THROTTLE_FLAG = false;
          this._move_throttle();
          let width = this.width,
            height = this.height,
            cut_top = this.cut_top,
            cut_left = this.cut_left,
            size_correct = () => {
              width = width <= this.max_width ? width >= this.min_width ? width : this.min_width : this.max_width;
              height = height <= this.max_height ? height >= this.min_height ? height : this.min_height : this.max_height;
            },
            size_inspect = () => {
              if ((width > this.max_width || width < this.min_width || height > this.max_height || height < this.min_height) && this.disable_ratio) {
                size_correct();
                return false;
              } else {
                size_correct();
                return true;
              }
            };
          height = this.CUT_START.height + ((this.CUT_START.corner > 1 && this.CUT_START.corner < 4 ? 1 : -1) * (this.CUT_START.y - e.touches[0].clientY));
          switch (this.CUT_START.corner) {
            case 1:
              width = this.CUT_START.width + this.CUT_START.x - e.touches[0].clientX;
              if (this.disable_ratio) {
                height = width / (this.width / this.height)
              }
              if (!size_inspect()) return;
              cut_left = this.CUT_START.cut_left - (width - this.CUT_START.width);
              break
            case 2:
              width = this.CUT_START.width + this.CUT_START.x - e.touches[0].clientX;
              if (this.disable_ratio) {
                height = width / (this.width / this.height)
              }
              if (!size_inspect()) return;
              cut_top = this.CUT_START.cut_top - (height - this.CUT_START.height)
              cut_left = this.CUT_START.cut_left - (width - this.CUT_START.width)
              break
            case 3:
              width = this.CUT_START.width - this.CUT_START.x + e.touches[0].clientX;
              if (this.disable_ratio) {
                height = width / (this.width / this.height)
              }
              if (!size_inspect()) return;
              cut_top = this.CUT_START.cut_top - (height - this.CUT_START.height);
              break
            case 4:
              width = this.CUT_START.width - this.CUT_START.x + e.touches[0].clientX;
              if (this.disable_ratio) {
                height = width / (this.width / this.height)
              }
              if (!size_inspect()) return;
              break
          }
          if (!this.disable_width && !this.disable_height) {
            this.width = width,
            this.cut_left = cut_left
            this.height = height
            this.cut_top = cut_top
          } else if (!this.disable_width) {
            this.width = width
            this.cut_left = cut_left
          } else if (!this.disable_height) {
            this.height = height
            this.cut_top = cut_top
          }
          this._imgMarginDetectionScale();
        }
      },
      _cutTouchStart(e) {
        // 4右下 3右上 2左上 1左下
        let currentX = e.touches[0].clientX;
        let currentY = e.touches[0].clientY; 
        let cutbox_top4 = this.cut_top + this.height - 30;
        let cutbox_bottom4 = this.cut_top + this.height + 20;
        let cutbox_left4 = this.cut_left + this.width - 30;
        let cutbox_right4 = this.cut_left + this.width + 30;
    
        let cutbox_top3 = this.cut_top - 30;
        let cutbox_bottom3 = this.cut_top + 30;
        let cutbox_left3 = this.cut_left + this.width - 30;
        let cutbox_right3 = this.cut_left + this.width + 30;
    
        let cutbox_top2 = this.cut_top - 30;
        let cutbox_bottom2 = this.cut_top + 30;
        let cutbox_left2 = this.cut_left - 30;
        let cutbox_right2 = this.cut_left + 30;
    
        let cutbox_top1 = this.cut_top + this.height - 30;
        let cutbox_bottom1 = this.cut_top + this.height + 30;
        let cutbox_left1 = this.cut_left - 30;
        let cutbox_right1 = this.cut_left + 30;
        if (currentX > cutbox_left4 && currentX < cutbox_right4 && currentY > cutbox_top4 && currentY < cutbox_bottom4) {
          this._moveDuring();
          this.c_flag_cut_touch = true;
          this.c_flag_img_endtouch = true;
          this.CUT_START = {
            width: this.width,
            height: this.height,
            x: currentX,
            y: currentY,
            corner: 4
          }
        } else if (currentX > cutbox_left3 && currentX < cutbox_right3 && currentY > cutbox_top3 && currentY < cutbox_bottom3) {
          this._moveDuring();
          this.c_flag_cut_touch = true;
          this.c_flag_img_endtouch = true;
          this.CUT_START = {
            width: this.width,
            height: this.height,
            x: currentX,
            y: currentY,
            cut_top: this.cut_top,
            cut_left: this.cut_left,
            corner: 3
          }
        } else if (currentX > cutbox_left2 && currentX < cutbox_right2 && currentY > cutbox_top2 && currentY < cutbox_bottom2) {
          this._moveDuring();
          this.c_flag_cut_touch = true;
          this.c_flag_img_endtouch = true;
          this.CUT_START = {
            width: this.width,
            height: this.height,
            cut_top: this.cut_top,
            cut_left: this.cut_left,
            x: currentX,
            y: currentY,
            corner: 2
          }
        } else if (currentX > cutbox_left1 && currentX < cutbox_right1 && currentY > cutbox_top1 && currentY < cutbox_bottom1) {
          this._moveDuring();
          this.c_flag_cut_touch = true;
          this.c_flag_img_endtouch = true;
          this.CUT_START = {
            width: this.width,
            height: this.height,
            cut_top: this.cut_top,
            cut_left: this.cut_left,
            x: currentX,
            y: currentY,
            corner: 1
          }
        }
      },
      _cutTouchEnd(e) {
        this._moveStop();
        this.c_flag_cut_touch = false;
      },
      _confirmCut () { // 激发确认事件
        this._draw(()=>{
          wx.canvasToTempFilePath({
            width: this.width * this.export_scale,
            height: Math.round(this.height * this.export_scale),
            destWidth: this.width * this.export_scale,
            destHeight: Math.round(this.height) * this.export_scale,
            fileType: 'png',
            quality: this.quality,
            canvasId: this.el,
            success: (res) => {
              this.$emit('confirmCut', {
                url: res.tempFilePath,
                width: this.width * this.export_scale,
                height: this.height * this.export_scale
              });
            }
          }, this)
        });
      },
      _cancelCut () { // 取消裁剪
        this.$emit('cancelCut');
      },
      //停止移动时需要做的操作
      _moveStop() {
        //清空之前的自动居中延迟函数并添加最新的
        clearTimeout(this.TIME_CUT_CENTER);
        this.TIME_CUT_CENTER = setTimeout(() => {
          //动画启动
          if (!this.c_cut_animation) {
            this.c_cut_animation = true
          }
          // this.setCutCenter();
        }, 1000)
        //清空之前的背景变化延迟函数并添加最新的
        clearTimeout(this.TIME_BG);
        this.TIME_BG = setTimeout(() => {
          if (this.c_flag_bright) {
            this.c_flag_bright = false
          }
        }, 2000)
        
      },
      //移动中
      _moveDuring() {
        //清空之前的自动居中延迟函数
        clearTimeout(this.TIME_CUT_CENTER);
        //清空之前的背景变化延迟函数
        clearTimeout(this.TIME_BG);
        //高亮背景
        if (!this.c_flag_bright) {
          this.c_flag_bright = true
        }
      },
      _preventTouchMove() { // 阻止弹窗穿透
        return
      }
  },
  
  created () {
    console.log(this.limit_move)
    this.info = wx.getSystemInfoSync();
    this.INIT_IMGWIDTH = this.img_width;
    this.INIT_IMGHEIGHT = this.img_height;
    this.c_canvas_height = this.height,
    this.c_canvas_width = this.width,
    // 初始化
    this._initCanvas();
    this.pushImg();
    //根据开发者设置的图片目标尺寸计算实际尺寸
    this._initImageSize();
    //设置裁剪框大小>设置图片尺寸>绘制canvas
    this._computeCutSize();
    //检查裁剪框是否在范围内
    this._cutDetectionPosition();
    //检查canvas是否在范围内
    this._canvasDetectionPosition();
    //初始化完成
    this.$emit('load', {
      cropper: this
    })
    console.log('finish')
  }
}
function styles(obj) { // styles转换函数
  let stylesStr = '';
  for (let styleName in obj) {
    if (obj.hasOwnProperty(styleName)) stylesStr += `${styleName.replace(/([A-Z])/g, "-$1").toLowerCase()}:${obj[styleName]};`;
  }
  return stylesStr;
}
</script>

<style>
.image-cropper{
  background:rgba(14, 13, 13, 1);
  position: fixed;
  top:0;
  left:0;
  width:100vw;
  height:100vh;
  z-index: 1;
}
.header{
  height: 100rpx;
  font-size: 38rpx;
  color: #fff;
  background: rgba(14, 13, 13, 1);
  left: 0;
  top: 0;
}
.header .op-btn{
  width: 200rpx;
  height: 80rpx;
  line-height: 80rpx;
  font-size: 34rpx;
  display: inline-block;
}
.header .confirm-btn{
  margin: 10rpx 0 0 10rpx;
  color: #fff;
  background: #1AAD19;
}
.header .cancel-btn{
  right: 10rpx;
  top: 10rpx;
  position: absolute;
}
.header .title{
  line-height: 100rpx;
  height: 100%;
  float: right;
  margin-right: 38%;
}
.main{
  position: absolute;
  width:100vw;
  height:100vh;
  overflow: hidden;
  top: 0;
  z-index: -1;
}
.content{
  z-index: 9;
  position: absolute;
  width:100vw;
  height:100vh;
  display: flex;
  flex-direction:column;
  pointer-events:none;
}
.bg_black{
  background: rgba(0, 0, 0, 0.8)!important;
}
.bg_gray{
  background: rgba(0, 0, 0, 0.45);
  transition-duration: .35s;
}
.content>.content_top{
  pointer-events:none;
}
.content>.content_middle{
  display: flex;
  height: 200px;
  width:100%;
}
.content_middle_middle{
  width:200px;
  box-sizing:border-box;
  position: relative;
  transition-duration: .3s;
}
.content_middle_right{
  flex: auto;
}
.content>.content_bottom{
  flex: auto;
}
.image-cropper .img{
  z-index: 2;
  top:0;
  left:0;
  position: absolute;
  border:none;
  width:100%;
  backface-visibility: hidden;
  transform-origin:center;
}
.image-cropper-canvas{
  position: fixed;
  background: white;
  width:150px;
  height:150px;
  z-index: 10;
  top:-200%;
  pointer-events:none;
}
.border{
  background: white;
  pointer-events:auto;
  position:absolute;
}
.border-top-left{
  left:-2.5px;
  top:-2.5px;
  height:2.5px;
  width:33rpx;
}
.border-top-right{
  right:-2.5px;
  top:-2.5px;
  height:2.5px;
  width:33rpx;
}
.border-right-top{
  top:-1px;
  width:2.5px;
  height:30rpx;
  right:-2.5px;
}
.border-right-bottom{
  width:2.5px;
  height:30rpx;
  right:-2.5px;
  bottom:-1px;
}
.border-bottom-left{
  height:2.5px;
  width:33rpx;
  bottom:-2.5px;
  left:-2.5px;
}
.border-bottom-right{
  height:2.5px;
  width:33rpx;
  bottom:-2.5px;
  right:-2.5px;
}
.border-left-top{
  top:-1px;
  width:2.5px;
  height:30rpx;
  left:-2.5px;
}
.border-left-bottom{
  width:2.5px;
  height:30rpx;
  left:-2.5px;
  bottom:-1px;
}
</style>
